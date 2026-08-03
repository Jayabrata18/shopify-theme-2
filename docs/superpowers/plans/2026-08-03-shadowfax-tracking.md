# Shadowfax Order Tracking Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Replace the placeholder tracking form in `sections/track-order.liquid` with a working Shadowfax integration that shows live status + a scan timeline for a customer-entered AWB.

**Architecture:** Pure client-side. Liquid injects the Shadowfax token (from a shop metafield) and base URL (from a section setting) into an inline JS config. Browser calls `GET {base}/v4/clients/orders/{awb}/track/` with `Authorization: Token {token}`. Response is normalised and rendered as a status hero + vertical scan timeline. No backend, no external JS libraries.

**Tech Stack:** Shopify Liquid, vanilla JavaScript (no libraries), CSS on the existing coolvetica-based `.to-*` system.

## Global Constraints

- **Single file only:** all changes go in `sections/track-order.liquid`. Do not touch `templates/page.track-order.json`, do not add asset files, do not add snippets.
- **No new dependencies:** no external CSS or JS. No new fonts (coolvetica is already loaded by the theme).
- **No PII rendering:** the Shadowfax response includes customer name, phone, address, and product value in `order_details`. These fields must never be read into a variable, logged, or rendered. Only `awb_number`, `client_order_id`, `status`, `status_display`, `promised_delivery_date`, `customer_track_url` from `order_details`, and the full `tracking_details` array (status, location, created, remarks, status_id) are consumed.
- **Token source:** `{{ shop.metafields.custom.shadowfox_delivery_tracking_key.value }}` — output as a JSON string in the inline config.
- **Base URL:** production `https://dale.shadowfax.in/api` by default; staging `https://dale.staging.shadowfax.in/api` when the `use_staging` section setting is on.
- **Auth header format:** `Authorization: Token <token_value>` — the literal word `Token` is part of the value, not a scheme name.
- **Content max-width:** 640px (matches existing `.to-page`). Mobile breakpoint at 540px.
- **Testing model:** there is no JS test framework in this Shopify theme. Each task ends with manual browser verification against the staging Shadowfax environment. Use section setting `use_staging = true` while implementing.
- **Test AWBs (staging):** `SF1360593963TES`, `SF1360594301TES`. Any obviously-invalid AWB (e.g. `NOTFOUND`) will return HTTP 400 from Shadowfax.

---

## File Structure

Only one file is changed:

- **Modify:** `sections/track-order.liquid` — replace the `<style>` block additions, form markup, result panel markup, `<script>` block, and `{% schema %}` block. The outer `.to-page`, eyebrow/heading/sub, support links, and info strip are kept as-is.

The file grows to roughly ~600 lines. That's within reason for a self-contained Shopify section — do not split into snippets (the spec is explicit about single-file scope).

### Shared identifiers across tasks (interface contract)

DOM ids used across tasks:

- `toAwbInput` — the AWB text input (Task 1)
- `toSubmit` — the submit button (Task 1)
- `toError` — the error banner (existing, kept in Task 1)
- `toResult` — the result panel root (existing, kept in Task 1)
- `toStatusHero` — status hero container (Task 3)
- `toStatusLabel` — status text (Task 3)
- `toStatusPill` — pill element that gets a modifier class (Task 3)
- `toStatusMeta` — sub-line with AWB · location · ETA (Task 3)
- `toTrackLink` — anchor to `customer_track_url` (Task 3)
- `toTimeline` — timeline container (Task 4)

JS functions/objects used across tasks:

- `SFX_CONFIG` — `{ token: string, base: string, terminalIds: string[], failedIds: string[] }` — populated from Liquid, read by all JS (Task 2).
- `fetchTracking(awb) → Promise<Normalised | null>` — resolves to a normalised object on success, `null` on 400/404, rejects on other errors (Task 2).
- `normalise(raw)` — pulls the whitelist of fields from the raw Shadowfax response (Task 2).
- `classify(statusId) → 'delivered' | 'failed' | 'transit'` — bucket lookup (Task 2, used by Task 3).
- `renderStatusHero(data)` — populates `#toStatusHero` (Task 3, called from Task 2's submit handler).
- `renderTimeline(scans, bucket)` — populates `#toTimeline` (Task 4, called from Task 2's submit handler).
- `formatScanTime(iso)` — `MMM D, YYYY · h:mm A` in local time (Task 4).

Normalised shape (produced by Task 2, consumed by Tasks 3 and 4):

```js
{
  awb: string | null,
  orderId: string | null,
  status: string,            // human label
  statusId: string | null,   // machine id, e.g. "ofd"
  lastLocation: string | null,
  edd: string | null,        // "YYYY-MM-DD" or null
  trackUrl: string | null,
  scans: [{ status, location, time, note, statusId }]  // in API order (oldest first)
}
```

---

## Task 1: Form markup, schema settings, form CSS

**Files:**
- Modify: `sections/track-order.liquid` — form fields inside `.to-card`, `<style>` block, `{% schema %}` block.

**Interfaces:**
- Consumes: nothing.
- Produces:
  - DOM ids `toAwbInput`, `toSubmit`, `toError`, `toResult` (last two already exist and are preserved).
  - Section settings: `use_staging` (checkbox), `terminal_status_ids` (text), `failed_status_ids` (text) — read in Task 2.
  - CSS classes `.to-input` and existing form styling stay usable.

- [ ] **Step 1: Expected outcome** — page renders with a single "Tracking Number (AWB)" input (no email field), the schema shows three new settings in the theme editor, and the existing look/feel is preserved.

- [ ] **Step 2: Replace form fields inside `.to-card`**

Locate the two `.to-field` blocks (currently `toOrderNum` and `toEmail`) and replace them with a single AWB field. The `.to-error`, `.to-submit` button, and `.to-result` container stay exactly as they are — only the two input fields change.

```liquid
    <div class="to-field">
      <label class="to-label" for="toAwbInput">Tracking Number (AWB)</label>
      <input
        class="to-input"
        id="toAwbInput"
        type="text"
        placeholder="SF1360593963TES"
        autocomplete="off"
        autocapitalize="characters"
        spellcheck="false"
      >
      <p class="to-hint">Found in your shipping confirmation email.</p>
    </div>
```

Also update the submit button `id="toSubmit"` — the existing element already has this id, keep it. Do not delete `.to-error`, `.to-submit`, or `.to-result` — later tasks depend on them.

- [ ] **Step 3: Update section schema**

Replace the existing `{% schema %}` block (currently `"settings": []`) with:

```liquid
{% schema %}
{
  "name": "Track Order",
  "settings": [
    {
      "type": "header",
      "content": "Shadowfax API"
    },
    {
      "type": "checkbox",
      "id": "use_staging",
      "label": "Use Shadowfax staging environment",
      "default": false,
      "info": "When on, calls dale.staging.shadowfax.in instead of production."
    },
    {
      "type": "text",
      "id": "terminal_status_ids",
      "label": "Delivered status IDs",
      "default": "delivered,rts_d,rto_d",
      "info": "Comma-separated Shadowfax status_id values shown with the green pill."
    },
    {
      "type": "text",
      "id": "failed_status_ids",
      "label": "Failed status IDs",
      "default": "rto_i,ndr,lost",
      "info": "Comma-separated Shadowfax status_id values shown with the red pill."
    }
  ]
}
{% endschema %}
```

- [ ] **Step 4: Load the page and verify markup**

Push to the Shopify dev theme (or preview locally with `shopify theme dev`) and open `/pages/track-order`.

Expected:
- One "Tracking Number (AWB)" field with placeholder `SF1360593963TES` and the hint text.
- No "Email Address" field.
- "Track Order" button still present.
- No visible result panel or errors on first load.
- In the theme editor, opening the section shows the three new settings under a "Shadowfax API" header.

- [ ] **Step 5: Commit**

```bash
git add sections/track-order.liquid
git commit -m "track-order: switch form to AWB-only, add Shadowfax section settings"
```

---

## Task 2: JS wire-up — config, API call, submit handler, minimal render

**Files:**
- Modify: `sections/track-order.liquid` — replace the entire `<script>` block (the existing IIFE).

**Interfaces:**
- Consumes: DOM ids from Task 1 (`toAwbInput`, `toSubmit`, `toError`, `toResult`); section settings from Task 1.
- Produces:
  - `SFX_CONFIG` global (window-scoped inside the IIFE; not exported).
  - `fetchTracking(awb)`, `normalise(raw)`, `classify(statusId)` functions inside the IIFE.
  - Normalised data object shape documented in the plan header.
  - Placeholders in `#toResult` for the status label and AWB (this task renders minimal text; Tasks 3 and 4 replace it with hero + timeline markup).

- [ ] **Step 1: Expected outcome** — entering a valid staging AWB and clicking submit hits Shadowfax, receives a real response, and renders a plain-text summary of status + AWB into the result panel. Errors (400, 401 simulated, network) render into `.to-error` with the correct message.

- [ ] **Step 2: Replace `.to-result` inner markup with placeholder containers**

Replace the current contents inside `<div class="to-result" id="toResult">` with empty containers that later tasks fill:

```liquid
    <div class="to-result" id="toResult">
      <div class="to-status-hero" id="toStatusHero"></div>
      <div class="to-timeline" id="toTimeline"></div>
    </div>
```

- [ ] **Step 3: Replace the entire `<script>` block**

Delete the existing IIFE completely and replace with:

```liquid
<script>
(function () {
  var SFX_CONFIG = {
    token: {{ shop.metafields.custom.shadowfox_delivery_tracking_key.value | default: '' | json }},
    base: {% if section.settings.use_staging %}'https://dale.staging.shadowfax.in/api'{% else %}'https://dale.shadowfax.in/api'{% endif %},
    terminalIds: {{ section.settings.terminal_status_ids | default: 'delivered,rts_d,rto_d' | split: ',' | json }},
    failedIds:   {{ section.settings.failed_status_ids   | default: 'rto_i,ndr,lost'         | split: ',' | json }}
  };

  var submitBtn   = document.getElementById('toSubmit');
  var awbInput    = document.getElementById('toAwbInput');
  var errorBox    = document.getElementById('toError');
  var resultPanel = document.getElementById('toResult');
  var heroBox     = document.getElementById('toStatusHero');
  var timelineBox = document.getElementById('toTimeline');

  var ERROR_NOT_FOUND    = 'No shipment found for {AWB}. Double-check the number in your shipping email.';
  var ERROR_UNAVAILABLE  = 'Tracking is temporarily unavailable. Please try again later.';
  var ERROR_NETWORK      = "Couldn't reach the tracking service. Try again in a minute.";

  var SUBMIT_IDLE_HTML = '<svg viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" style="width:16px;height:16px"><circle cx="11" cy="11" r="8"/><line x1="21" y1="21" x2="16.65" y2="16.65"/></svg> Track Order';
  var SUBMIT_BUSY_HTML = '<svg viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" style="animation:to-spin 0.8s linear infinite;width:16px;height:16px"><path d="M12 2v4M12 18v4M4.93 4.93l2.83 2.83M16.24 16.24l2.83 2.83M2 12h4M18 12h4M4.93 19.07l2.83-2.83M16.24 7.76l2.83-2.83"/></svg> Tracking…';

  function showError(msg) {
    errorBox.textContent = msg;
    errorBox.style.display = 'block';
    resultPanel.style.display = 'none';
  }

  function clearError() {
    errorBox.style.display = 'none';
    errorBox.textContent = '';
  }

  function classify(statusId) {
    if (!statusId) return 'transit';
    var id = String(statusId).toLowerCase();
    if (SFX_CONFIG.terminalIds.indexOf(id) !== -1) return 'delivered';
    if (SFX_CONFIG.failedIds.indexOf(id)   !== -1) return 'failed';
    return 'transit';
  }

  function normalise(raw) {
    var o = (raw && raw.order_details) || {};
    var scans = (raw && raw.tracking_details) || [];
    var last = scans[scans.length - 1] || {};
    return {
      awb:          o.awb_number || null,
      orderId:      o.client_order_id || null,
      status:       o.status_display || last.status || 'Unknown',
      statusId:     o.status || last.status_id || null,
      lastLocation: last.location || null,
      edd:          o.promised_delivery_date || null,
      trackUrl:     o.customer_track_url || null,
      scans: scans.map(function (s) {
        return {
          status:   s.status,
          location: s.location,
          time:     s.created,
          note:     s.remarks,
          statusId: s.status_id || null
        };
      })
    };
  }

  function fetchTracking(awb) {
    if (!SFX_CONFIG.token) {
      console.warn('[track-order] Shadowfax token metafield is empty.');
      return Promise.reject(new Error('no-token'));
    }
    var url = SFX_CONFIG.base + '/v4/clients/orders/' + encodeURIComponent(awb) + '/track/';
    return fetch(url, {
      headers: {
        'Authorization': 'Token ' + SFX_CONFIG.token,
        'Accept': 'application/json'
      }
    }).then(function (res) {
      if (res.status === 400 || res.status === 404) return null;
      if (res.status === 401) throw new Error('unauthorized');
      if (!res.ok) throw new Error('http-' + res.status);
      return res.json().then(normalise);
    });
  }

  function setBusy(busy) {
    submitBtn.disabled = busy;
    submitBtn.innerHTML = busy ? SUBMIT_BUSY_HTML : SUBMIT_IDLE_HTML;
  }

  // Minimal render — replaced by richer renderers in Tasks 3 and 4.
  function renderMinimal(data) {
    heroBox.textContent = data.status + ' — ' + (data.awb || '');
    timelineBox.textContent = '';
    resultPanel.style.display = 'block';
    resultPanel.scrollIntoView({ behavior: 'smooth', block: 'nearest' });
  }

  submitBtn.addEventListener('click', function () {
    clearError();
    var awb = awbInput.value.trim().toUpperCase();
    if (!awb) { showError('Please enter your tracking number.'); awbInput.focus(); return; }

    setBusy(true);
    fetchTracking(awb).then(function (data) {
      setBusy(false);
      if (!data) {
        showError(ERROR_NOT_FOUND.replace('{AWB}', awb));
        return;
      }
      renderMinimal(data);
    }).catch(function (err) {
      setBusy(false);
      if (err && err.message === 'unauthorized') return showError(ERROR_UNAVAILABLE);
      if (err && err.message === 'no-token')     return showError(ERROR_UNAVAILABLE);
      showError(ERROR_NETWORK);
    });
  });

  awbInput.addEventListener('keydown', function (e) {
    if (e.key === 'Enter') submitBtn.click();
  });
})();
</script>
```

- [ ] **Step 4: Turn on staging in the theme editor**

Open the section settings and toggle **Use Shadowfax staging environment** on.

- [ ] **Step 5: Verify successful lookup**

Load the page, enter `SF1360593963TES`, click Track Order.

Expected:
- Button briefly shows "Tracking…" then returns to "Track Order".
- Result panel appears with plain text like `Out For Delivery — SF1360593963TES` (exact status depends on staging data).
- No error banner.
- DevTools Network tab shows a `GET` to `dale.staging.shadowfax.in/api/v4/clients/orders/SF1360593963TES/track/` returning 200.

**If CORS blocks the request** (browser console shows a CORS error) — this is the documented risk from the spec. Stop here, report the block to the user, and do not proceed with Tasks 3–5 until a proxy path is chosen.

- [ ] **Step 6: Verify not-found path**

Clear the input, enter `NOTFOUND`, submit.

Expected: red error banner reads `No shipment found for NOTFOUND. Double-check the number in your shipping email.` Result panel does not appear.

- [ ] **Step 7: Verify empty-token path**

Temporarily clear the metafield value in Shopify admin (or set `token: ''` inline for one test), reload, enter any AWB, submit.

Expected: banner reads `Tracking is temporarily unavailable. Please try again later.` Console shows the warn `[track-order] Shadowfax token metafield is empty.` Restore the metafield afterwards.

- [ ] **Step 8: Commit**

```bash
git add sections/track-order.liquid
git commit -m "track-order: call Shadowfax API and render minimal status"
```

---

## Task 3: Status hero — pill, meta line, external link

**Files:**
- Modify: `sections/track-order.liquid` — add CSS, replace `renderMinimal` with `renderStatusHero`.

**Interfaces:**
- Consumes:
  - `SFX_CONFIG`, `classify(statusId)`, normalised `data` shape from Task 2.
  - DOM ids `toStatusHero`, `toResult`, `toTimeline` from Task 2.
- Produces:
  - `renderStatusHero(data)` function (called from the submit handler in place of `renderMinimal`).
  - DOM structure inside `#toStatusHero`: `.to-status-hero__label`, `.to-status-pill.to-status-pill--{bucket}`, `.to-status-hero__meta`, `.to-track-link` (link only rendered when `data.trackUrl` present).

- [ ] **Step 1: Expected outcome** — after a successful lookup, the result panel shows a bold status label, a color-coded pill (green/red/amber), a sub-line with `AWB · Last location · ETA <date>` (ETA hidden for terminal states), and an optional external link.

- [ ] **Step 2: Add hero CSS**

Append the following inside the existing `<style>` block, after the `.to-result__link` rule and before the `/* ── Divider ── */` comment. Also, delete the now-unused rules `.to-result__label`, `.to-result__value`, `.to-result__status`, `.to-result__status--processing`, and `.to-result__link` — they belonged to the old placeholder result panel.

```css
/* ── Result: status hero ── */
.to-status-hero {
  padding-bottom: 20px;
  border-bottom: 1px solid #eee;
  margin-bottom: 20px;
}

.to-status-hero__label {
  font-size: 1.35rem;
  font-weight: 500;
  color: #111;
  letter-spacing: 0.01em;
  margin: 0 0 10px;
}

.to-status-pill {
  display: inline-flex;
  align-items: center;
  gap: 6px;
  font-size: 0.72rem;
  font-weight: 700;
  letter-spacing: 0.12em;
  text-transform: uppercase;
  padding: 5px 12px;
  border-radius: 999px;
  margin-bottom: 12px;
}
.to-status-pill::before {
  content: '';
  width: 6px;
  height: 6px;
  border-radius: 50%;
  background: currentColor;
}
.to-status-pill--delivered { background: #ecfdf5; color: #047857; }
.to-status-pill--failed    { background: #fef2f2; color: #b91c1c; }
.to-status-pill--transit   { background: #fff7ed; color: #b45309; }

.to-status-hero__meta {
  font-size: 0.82rem;
  color: #666;
  line-height: 1.55;
  margin: 6px 0 0;
}

.to-status-hero__meta strong {
  color: #111;
  font-weight: 600;
}

.to-track-link {
  display: inline-block;
  margin-top: 10px;
  font-size: 0.78rem;
  font-weight: 700;
  letter-spacing: 0.08em;
  text-transform: uppercase;
  color: #111;
  text-decoration: underline;
  transition: opacity 0.2s;
}
.to-track-link:hover { opacity: 0.6; }
```

- [ ] **Step 3: Add `renderStatusHero`, keep the timeline call as a stub**

In the `<script>` block, delete the entire `renderMinimal` function. Add `renderStatusHero` and a `renderTimeline` stub (Task 4 replaces the stub):

```javascript
  function esc(str) {
    return String(str == null ? '' : str).replace(/[&<>"']/g, function (c) {
      return { '&': '&amp;', '<': '&lt;', '>': '&gt;', '"': '&quot;', "'": '&#39;' }[c];
    });
  }

  function formatEdd(iso) {
    if (!iso) return '';
    var d = new Date(iso);
    if (isNaN(d.getTime())) return esc(iso);
    return d.toLocaleDateString(undefined, { month: 'short', day: 'numeric', year: 'numeric' });
  }

  function renderStatusHero(data, bucket) {
    var parts = [];
    if (data.awb)          parts.push('<strong>' + esc(data.awb) + '</strong>');
    if (data.lastLocation) parts.push(esc(data.lastLocation));
    if (data.edd && bucket !== 'delivered' && bucket !== 'failed') {
      parts.push('ETA ' + esc(formatEdd(data.edd)));
    }

    var linkHtml = data.trackUrl
      ? '<a class="to-track-link" href="' + esc(data.trackUrl) + '" target="_blank" rel="noopener">View on Shadowfax →</a>'
      : '';

    heroBox.innerHTML =
      '<p class="to-status-hero__label">' + esc(data.status) + '</p>' +
      '<span class="to-status-pill to-status-pill--' + bucket + '">' + esc(data.status) + '</span>' +
      (parts.length ? '<p class="to-status-hero__meta">' + parts.join(' &middot; ') + '</p>' : '') +
      linkHtml;
  }

  function renderTimeline(scans, bucket) {
    // Replaced in Task 4.
    timelineBox.innerHTML = '';
  }
```

- [ ] **Step 4: Update the submit handler to call the new renderers**

Inside the submit handler's success branch, replace `renderMinimal(data);` with:

```javascript
      var bucket = classify(data.statusId);
      renderStatusHero(data, bucket);
      renderTimeline(data.scans, bucket);
      resultPanel.style.display = 'block';
      resultPanel.scrollIntoView({ behavior: 'smooth', block: 'nearest' });
```

- [ ] **Step 5: Verify hero rendering (in-transit AWB)**

Reload, enter `SF1360593963TES`, submit.

Expected:
- Bold status label at the top of the result panel.
- Amber pill (`--transit`) with the same status text and a small colored dot before the text.
- Sub-line reading `SF1360593963TES · <last location> · ETA <Mon D, YYYY>` (ETA present because the status is not terminal).
- "View on Shadowfax →" link if the API returned a `customer_track_url`; otherwise no link. No customer name, phone, or address anywhere.

- [ ] **Step 6: Verify hero rendering (delivered AWB)**

If a delivered AWB is available on staging, enter it. Otherwise temporarily add `delivered` to a known in-transit AWB by editing the `classify` call locally — but revert before committing.

Expected: green pill (`--delivered`), no ETA in the sub-line.

- [ ] **Step 7: Commit**

```bash
git add sections/track-order.liquid
git commit -m "track-order: render Shadowfax status hero with color-coded pill"
```

---

## Task 4: Timeline of scans

**Files:**
- Modify: `sections/track-order.liquid` — add CSS, replace the `renderTimeline` stub.

**Interfaces:**
- Consumes: `esc()` from Task 3; the `scans` array from the normalised data (each scan has `status`, `location`, `time`, `note`, `statusId`); `bucket` from `classify()`.
- Produces: `renderTimeline(scans, bucket)` full implementation; `formatScanTime(iso)` helper.

- [ ] **Step 1: Expected outcome** — below the hero, a vertical timeline of all scans renders newest-first. Each row: colored dot on the left, then status label (bold), location, timestamp, and remarks. The newest row is emphasised; older rows fade to ~80% opacity.

- [ ] **Step 2: Add timeline CSS**

Append to the `<style>` block, right after the hero rules from Task 3:

```css
/* ── Result: timeline ── */
.to-timeline {
  position: relative;
  padding-left: 22px;
}
.to-timeline::before {
  content: '';
  position: absolute;
  left: 5px;
  top: 6px;
  bottom: 6px;
  width: 1px;
  background: #e8e8e8;
}

.to-scan {
  position: relative;
  padding: 0 0 20px 0;
  opacity: 0.8;
}
.to-scan:last-child { padding-bottom: 0; }
.to-scan--latest { opacity: 1; }

.to-scan__dot {
  position: absolute;
  left: -22px;
  top: 4px;
  width: 11px;
  height: 11px;
  border-radius: 50%;
  background: #cbd5e1;
  border: 2px solid #fff;
  box-shadow: 0 0 0 1px #cbd5e1;
}
.to-scan--latest .to-scan__dot--delivered { background: #047857; box-shadow: 0 0 0 1px #047857; }
.to-scan--latest .to-scan__dot--failed    { background: #b91c1c; box-shadow: 0 0 0 1px #b91c1c; }
.to-scan--latest .to-scan__dot--transit   { background: #b45309; box-shadow: 0 0 0 1px #b45309; }

.to-scan__head {
  font-size: 0.86rem;
  font-weight: 600;
  color: #111;
  margin: 0 0 3px;
  line-height: 1.4;
}

.to-scan__meta {
  font-size: 0.74rem;
  color: #888;
  margin: 0;
  line-height: 1.5;
}

.to-scan__remarks {
  font-size: 0.76rem;
  color: #666;
  margin: 4px 0 0;
  line-height: 1.5;
}
```

- [ ] **Step 3: Replace the `renderTimeline` stub**

Delete the stub added in Task 3 and add the full implementation plus a `formatScanTime` helper:

```javascript
  function formatScanTime(iso) {
    if (!iso) return '';
    var d = new Date(iso);
    if (isNaN(d.getTime())) return esc(iso);
    var date = d.toLocaleDateString(undefined, { month: 'short', day: 'numeric', year: 'numeric' });
    var time = d.toLocaleTimeString(undefined, { hour: 'numeric', minute: '2-digit' });
    return date + ' · ' + time;
  }

  function renderTimeline(scans, bucket) {
    if (!scans || !scans.length) {
      timelineBox.innerHTML = '';
      return;
    }
    // Newest first. Shadowfax returns oldest-first, so reverse a copy.
    var ordered = scans.slice().reverse();
    var html = ordered.map(function (scan, i) {
      var latestCls = i === 0 ? ' to-scan--latest' : '';
      var dotCls    = i === 0 ? ' to-scan__dot--' + bucket : '';
      var metaBits  = [];
      if (scan.location)          metaBits.push(esc(scan.location));
      if (scan.time)              metaBits.push(esc(formatScanTime(scan.time)));
      var remarksHtml = scan.note
        ? '<p class="to-scan__remarks">' + esc(scan.note) + '</p>'
        : '';
      return '' +
        '<div class="to-scan' + latestCls + '">' +
          '<span class="to-scan__dot' + dotCls + '"></span>' +
          '<p class="to-scan__head">' + esc(scan.status || 'Update') + '</p>' +
          (metaBits.length ? '<p class="to-scan__meta">' + metaBits.join(' · ') + '</p>' : '') +
          remarksHtml +
        '</div>';
    }).join('');
    timelineBox.innerHTML = html;
  }
```

- [ ] **Step 4: Verify timeline renders**

Reload, submit a staging AWB with multiple scans (either test AWB should have several).

Expected:
- Below the hero, a vertical line runs down the left with a colored dot per scan.
- Scans are ordered newest → oldest.
- The top (newest) scan uses full opacity with a colored dot matching the hero pill (green/red/amber).
- Older scans fade slightly (opacity ~0.8) with grey dots.
- Each row shows: bold status, then `location · Mon D, YYYY · h:mm AM` in a muted line, then the remarks in a smaller muted line beneath.
- No customer name, phone, or address anywhere on the page.

- [ ] **Step 5: Verify on mobile viewport**

DevTools → responsive mode → 375×667. Reload.

Expected:
- Form and timeline stack cleanly, no horizontal scroll.
- Timeline vertical rule stays flush left of the dots; no wrapping into the rule.

- [ ] **Step 6: Commit**

```bash
git add sections/track-order.liquid
git commit -m "track-order: render Shadowfax scan timeline"
```

---

## Task 5: Final error-path QA + production flip

**Files:**
- Modify: `sections/track-order.liquid` — only if issues are found in QA.

**Interfaces:** none new.

- [ ] **Step 1: Expected outcome** — every documented error path renders the correct message, no console errors in the success path, section works with `use_staging` off (production).

- [ ] **Step 2: Confirm each error branch on staging**

With staging on:

- Empty input → click submit. Expected: banner `Please enter your tracking number.`, input focused, no network call in DevTools.
- Garbage AWB `NOTFOUND` → submit. Expected: banner `No shipment found for NOTFOUND...`, one `GET` to Shadowfax returning 400, result panel hidden.
- Simulate 401: DevTools → Network → Block request URL pattern `*shadowfax*` after entering an AWB, or temporarily rewrite the header to a bad value. Expected: banner `Tracking is temporarily unavailable. Please try again later.`
- Simulate network failure: DevTools → Network → Offline. Submit any AWB. Expected: banner `Couldn't reach the tracking service. Try again in a minute.`
- Empty token: temporarily clear the metafield, reload, submit. Expected: `unavailable` banner + console warn. Restore the metafield.

- [ ] **Step 3: Confirm success path has no console output**

Restore online mode, run a valid lookup. Expected: DevTools Console is clean (no warnings, no errors, no leftover `console.log`).

If any `console.log` is present, remove it in this task and re-commit.

- [ ] **Step 4: Flip to production and re-verify**

In the theme editor, turn **Use Shadowfax staging environment** off. Save.

Look up a real, live AWB you have (from a shipped order).

Expected:
- `GET` fires against `dale.shadowfax.in/api/...` (no `staging` subdomain).
- Hero and timeline populate correctly.
- Pill color matches the real order's stage.

**If production returns CORS errors even though staging worked** — stop and report; production may require CORS to be enabled on their side, or a proxy is needed.

- [ ] **Step 5: Commit only if changes were made**

If Step 3 required removing a leftover log or another cleanup was needed:

```bash
git add sections/track-order.liquid
git commit -m "track-order: QA cleanup"
```

Otherwise skip the commit — nothing to record.

---

## Self-Review

- **Spec coverage:**
  - Architecture (single file, client-side, config from Liquid) → Tasks 1, 2. ✅
  - CORS risk called out as blocking condition → Task 2 Step 5 halts execution. ✅
  - PII whitelist → Global Constraints + Task 2 `normalise` uses explicit whitelist. ✅
  - Form with AWB only, no email → Task 1. ✅
  - Status hero with color-coded pill, meta line, external link, ETA hidden for terminal → Task 3. ✅
  - Timeline newest-first with fading older entries → Task 4. ✅
  - Four error messages exactly as specified → Task 2 (constants) + Task 5 (verification). ✅
  - Three section settings (`use_staging`, `terminal_status_ids`, `failed_status_ids`) → Task 1 schema. ✅
  - Status classification buckets and defaults → Task 2 `classify` + Task 1 defaults. ✅
  - Mobile ≤540px → verified in Task 4 Step 5 (existing 540px media query in the file still applies). ✅
  - No new dependencies, no new fonts → nothing in any task adds them. ✅
  - Template file unchanged → no task touches `templates/page.track-order.json`. ✅

- **Placeholder scan:** no TBDs, no "add appropriate error handling" hand-waves — every branch has an exact message and behaviour. Every code step has actual code.

- **Type consistency:**
  - Normalised object shape in header matches what `normalise` returns in Task 2 and what Tasks 3–4 consume.
  - `classify(statusId)` returns strings `'delivered' | 'failed' | 'transit'`; Task 3 uses these as pill modifier suffixes (`to-status-pill--transit` etc.) and Task 4 uses them as dot modifier suffixes. Consistent.
  - `esc()` defined in Task 3, used in Task 4 — dependency order respected.
  - DOM ids used in Task 2's handler (`toStatusHero`, `toTimeline`) are created in Task 2 Step 2. Consistent.
