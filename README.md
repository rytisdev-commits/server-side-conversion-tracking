# Server-Side Conversion Tracking - Meta CAPI + Browser/Server Deduplication

A real case study from a live WooCommerce store I build and run. It's the same work I do for clients: make the conversion data trustworthy, recover the events browsers quietly drop, and stop the platforms from disagreeing.

## The problem

Most ecommerce stores track purchases with a browser-side pixel only. That pixel loses a large and growing share of conversions to iOS / ITP, ad blockers, consent tools that block the tag, and tab closes before the pixel fires.

The result is the pattern every founder eventually notices: Meta says one number, GA4 says another, and the store's own order count says a third. When the numbers can't be trusted, ad platforms optimise against bad signal and budget decisions are made on fiction.

## What I built

A custom WooCommerce mu-plugin that fires every `Purchase` twice on purpose - once in the browser (Meta Pixel) and once server-side (Meta Conversions API) - both carrying the same `event_id`. Meta sees the shared ID and deduplicates, so you get one clean conversion (no double-counting) and the recovered events the browser-only pixel was losing.

```
Browser:   fbq('track','Purchase', {...}, { eventID: 'purchase_' + orderId })
Server:    POST /events  ->  event_id: 'purchase_' + orderId
                                         ^ shared key -> Meta deduplicates
```

## How it works

```php
// Fire Purchase to the Meta Conversions API on payment complete,
// with the SAME event_id the browser pixel used -> Meta deduplicates.
add_action( 'woocommerce_payment_complete', function ( $order_id ) {
    $order    = wc_get_order( $order_id );
    $event_id = 'purchase_' . $order_id;            // shared dedup key

    $payload = [ 'data' => [[
        'event_name'       => 'Purchase',
        'event_time'       => time(),
        'event_id'         => $event_id,            // the dedup key
        'action_source'    => 'website',
        'event_source_url' => $order->get_checkout_order_received_url(),
        'user_data'        => [
            'em'  => [ hash( 'sha256', strtolower( $order->get_billing_email() ) ) ],
            'ph'  => [ hash( 'sha256', preg_replace( '/\D/', '', $order->get_billing_phone() ) ) ],
            'fbp' => $_COOKIE['_fbp'] ?? null,       // browser/server stitch
            'fbc' => $_COOKIE['_fbc'] ?? null,       // click id -> attribution
            'client_ip_address' => $_SERVER['REMOTE_ADDR']     ?? '',
            'client_user_agent' => $_SERVER['HTTP_USER_AGENT'] ?? '',
        ],
        'custom_data'      => [
            'currency' => $order->get_currency(),
            'value'    => $order->get_total(),
        ],
    ]]];

    wp_remote_post(
        "https://graph.facebook.com/v19.0/{$pixel_id}/events?access_token={$token}",
        [ 'headers' => [ 'Content-Type' => 'application/json' ],
          'body'    => wp_json_encode( $payload ) ]
    );
}, 10, 1 );
```

Key details that separate a working setup from a broken one:

- Shared `event_id` (`purchase_{order_id}`) on both browser and server - the single most common thing people get wrong.
- Hashed PII (email, phone) plus `fbp` / `fbc` so server events match to a user and keep click-id attribution.
- `action_source: website` and the real order-received URL so events are attributed, not orphaned.
- Verified live in Meta Events Manager (Test Events + deduplication report) before it goes near a real campaign.

## The outcome

On my own store this setup recovered Purchase events the browser-only pixel was dropping, raised Event Match Quality, and brought Meta, GA4 and the store's own order count into agreement - so the reported numbers can finally be trusted.

## How I audit a client's stack

1. Trace every event end to end - fire the action, watch it in the browser network, then GTM Preview, GA4 DebugView, and each platform's Events Manager.
2. Reconcile the numbers - count events platform by platform against the source of truth (orders) and find exactly where the number drops: missing server events, broken `event_id` dedup, consent blocking the tag, or a tag firing twice.
3. Report which number to trust, then fix the data, and verify the fix live.

## Stack

`Meta Pixel + Conversions API (CAPI)` · `event_id deduplication` · `Google Analytics 4` · `Google Tag Manager (web + server-side / Stape)` · `WooCommerce` · `Shopify` · `Google Ads Enhanced Conversions` · `Consent Mode`

---

Built and running on a live store I operate, not a sandbox. If your Meta, GA4 and store numbers don't agree, that gap is measurable and fixable, and I can usually point to the cause before you commit to anything.

## Field guides

Working notes from real audits - the diagnostic checklists I actually use:

- [Meta Pixel not firing? The 6 real causes, in the order to check them](https://rytisbalys.com/meta-pixel-not-firing)
- [GA4 purchases don't match your backend orders - which number to trust](https://rytisbalys.com/ga4-does-not-match-backend)
- [Shopify's checkout upgrade silently breaks tracking - what dies and how to migrate](https://rytisbalys.com/shopify-checkout-upgrade-tracking)
- [Google Ads only counts form fills, not real sales - how to import offline conversions](https://rytisbalys.com/offline-conversions-google-ads)

## Contact

**Rytis Balys** · [rytisbalys.com](https://rytisbalys.com) · work@rytisbalys.com · Vilnius, Lithuania (EET) · remote, worldwide
