# Shadowfax order tracking — design

**Date:** 2026-08-03
**Owner:** Jayabrata Pramanik
**Scope:** `sections/track-order.liquid` (single file), page template unchanged.

## Goal

Turn the current placeholder tracking page into a working Shadowfax lookup:
customer enters a Shadowfax AWB, page calls Shadowfax, shows current status +
a timeline of scans, styled to match the site.

## Non-goals

- Order # → AWB resolution (needs Admin API / backend).
- Email verification / customer identity checks.
- Auto-refresh / polling.
- Local caching of results.

## Architecture

Pure client-side. Section is self-contained: markup, styles, and JS in one
Liquid file. No backend added.

Liquid injects two values into the JS via a small inline config block:

- `SFX_TOKEN` — `{{ shop.metafields.custom.shadowfox_delivery_tracking_key.value }}`
- `SFX_BASE` — `https://dale.shadowfax.in/api` by default, or
  `https://dale.staging.shadowfax.in/api` when the `use_staging` section
  setting is on.

The browser calls
`GET {SFX_BASE}/v4/clients/orders/{awb}/track/`
with header `Authorization: Token {SFX_TOKEN}` (the word `Token` is part of
the value, not a scheme). Response is normalised and rendered in place.

### Accepted risks

1. **Token exposure.** The token is rendered into page HTML and readable by
   any visitor via View Source. This is a conscious tradeoff: no backend to
   run, and the token is rotatable in Shopify admin if abused.
2. **CORS.** Shadowfax's API is documented as server-to-server. If it does
   not send `Access-Control-Allow-Origin`, the browser will block the
   request. Mitigation path: move the call behind a Cloudflare Worker or
   Vercel edge function later. The client code is written so only the fetch
   URL needs to change.

## Data contract (subset consumed by the UI)

Shadowfax response shape (relevant fields only):

```json
{
  "order_details": {
    "awb_number": "SF...",
    "client_order_id": "1042",
    "status": "ofd",
    "status_display": "Out For Delivery",
    "promised_delivery_date": "2026-08-05",
    "customer_track_url": "https://..."
  },
  "tracking_details": [
    { "status": "New", "location": "SFX Kolkata Hub",
      "created": "2026-08-01T09:12:00Z", "remarks": "Item New at SFX Kolkata Hub",
      "status_id": "new" }
  ]
}
```

Everything else in `order_details` (customer name, phone, address, product
value) is ignored — it is not read into any variable and never rendered.

## UI

### Form

- Single text input: **Tracking Number (AWB)**.
- Placeholder: `SF1360593963TES`. Hint: "Found in your shipping confirmation email."
- Submit on click or Enter key.
- Input value is trimmed and uppercased before the API call.
- Loading state: button disables, spinner icon replaces search icon, label
  becomes "Tracking…".

### Result — status hero (top block)

- Large label: `status_display`.
- Colored pill using palette below. Pill state derived from `status_id` and
  the classification lists in section settings (see Section settings).
- Sub-line: `AWB · Last location · ETA {promised_delivery_date}` — ETA is
  hidden for terminal states.
- If `customer_track_url` is present, show a small text link
  "View on Shadowfax →".

### Result — timeline (middle block)

- Vertical timeline of `tracking_details`, newest first.
- Each entry renders:
  - Colored dot (matches pill color for the newest entry; muted grey for
    older entries).
  - Status label (`status`), bold.
  - Location (small, muted).
  - Timestamp formatted `MMM D, YYYY · h:mm A` in the user's local timezone.
  - Remarks (small, muted, wraps to new line).
- Newest entry uses slightly heavier weight and full-opacity text; older
  entries render at ~80% opacity.

### Result — error states

Reuse the existing `.to-error` block. Messages:

- HTTP 400 or 404 → "No shipment found for `{AWB}`. Double-check the number
  in your shipping email."
- HTTP 401 → "Tracking is temporarily unavailable. Please try again later."
  (Do not reveal that the token was rejected.)
- Fetch failure / CORS / network → "Couldn't reach the tracking service.
  Try again in a minute."
- Token metafield empty → same as 401 message; also `console.warn` for the
  developer.

## Status classification

Three buckets, driven by section settings:

| Bucket    | Default `status_id`s        | Pill palette                 |
|-----------|-----------------------------|------------------------------|
| Delivered | `delivered, rts_d, rto_d`   | bg `#ecfdf5`, fg `#047857`   |
| Failed    | `rto_i, ndr, lost`          | bg `#fef2f2`, fg `#b91c1c`   |
| In transit / other (default) | *(everything else)* | bg `#fff7ed`, fg `#b45309`   |

The initial `New` / `Manifested` state also falls into the "in transit"
bucket — this is intentional; a single non-terminal color keeps the UI
simple.

## Styling

Extend existing `.to-*` CSS. New selectors:

- `.to-status-hero`, `.to-status-hero__label`, `.to-status-hero__meta`
- `.to-status-pill`, `.to-status-pill--delivered`,
  `.to-status-pill--failed`, `.to-status-pill--transit`
- `.to-timeline`, `.to-scan`, `.to-scan__dot`, `.to-scan__body`,
  `.to-scan__head`, `.to-scan__meta`, `.to-scan__remarks`
- `.to-track-link`

Mobile-first, keeps the existing 640px content max-width. No new fonts, no
external CSS or JS dependencies.

## Section settings

Three new settings on the section schema:

- `use_staging` — checkbox, default `false`. When on, hits the staging base
  URL.
- `terminal_status_ids` — text, default `delivered,rts_d,rto_d`.
  Comma-separated `status_id`s treated as delivered.
- `failed_status_ids` — text, default `rto_i,ndr,lost`. Comma-separated
  `status_id`s treated as failed. Anything not in either list is treated as
  in-transit.

Existing `sections` array in `templates/page.track-order.json` does not
change — the section is still referenced as `track-order` with default
settings.

## File-level changes

Only file touched:

- `sections/track-order.liquid` — replace form + result markup, replace
  script block, extend `<style>` block, extend `{% schema %}`.

Template file `templates/page.track-order.json` is not modified.

## Manual test plan

Run against staging with `use_staging` on.

1. Enter `SF1360593963TES` → hero shows a non-terminal status pill, timeline
   populates with multiple scans, ETA visible.
2. Enter `SF1360594301TES` → same shape, different status.
3. Enter garbage AWB (e.g. `NOTFOUND`) → 400 branch renders the
   "No shipment found" message; result panel stays hidden.
4. Temporarily clear the metafield → error message renders, console warns.
5. Simulate CORS block (DevTools → block request) → network error message
   renders.
6. Small viewport (≤540px) → form and timeline stack cleanly, no horizontal
   scroll.

Once staging looks right, flip `use_staging` off and re-test on production
with a real live AWB.
