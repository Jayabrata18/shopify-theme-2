# Collections/All Page — Design Spec

**Date:** 2026-05-15
**URL:** `/collections/all`
**Approach:** Option B — New dedicated hero section + existing `main-collection` section

---

## Context and Goals

Deliver a dedicated `/collections/all` page for the urbnmyth Shopify theme. The page presents a full-width hero banner with a background image and "Shop All" title, followed by the standard filtered product grid. Design tokens and standards are drawn from `design.md` and `skills/design-system-shop-all.md`.

---

## Files to Create

| File | Purpose |
|---|---|
| `sections/collection-all-hero.liquid` | Custom hero section: background image, dark overlay, h1 title |
| `templates/collection.all.json` | Template JSON wiring hero + main-collection for `/collections/all` |

---

## Design Tokens

From `design.md`:

- **Font family:** `Montserrat`
- **Base font size:** `14px`, weight `400`, line-height `21px`
- **Title size:** `font.size.4xl = 19px` (used for h1 on hero)
- **Background overlay:** `#00000061` (matches existing collection.json pattern)
- **Text color:** `color.text.secondary = #ffffff`
- **Surface:** `color.surface.base = #000000`
- **Motion:** `motion.duration.normal = 300ms`

---

## Section 1: `sections/collection-all-hero.liquid`

### Anatomy

- Full-width container (`section--full-width`)
- Background image layer via existing `background-media` snippet
- Dark overlay via existing `overlay` snippet (toggle-able via schema, default on)
- Content layer: left-aligned `<h1>` with configurable title text

### Schema Settings

| Setting | Type | Default |
|---|---|---|
| `background_image` | image_picker | — |
| `toggle_overlay` | checkbox | true |
| `overlay_color` | color | `#00000061` |
| `title` | text | `"Shop All"` |
| `padding-block-start` | range (0–200px) | 100 |
| `padding-block-end` | range (0–100px) | 32 |
| `padding-inline-start` | range (0–100px) | 40 |

### States

- **Default:** image visible, overlay active, white h1 visible
- **No image set:** fallback to solid `#000000` background via `color_scheme`
- **Focus-visible:** any focusable elements inside must show visible outline
- **Mobile:** same layout, padding may reduce proportionally

### Responsive Behavior

- Full-width at all breakpoints
- Title text must not overflow; `wrap: pretty` applied
- Image covers full section (`object-fit: cover`)

### Accessibility

- Title must render as `<h1>` — only one per page
- Image must have `alt` attribute (empty `alt=""` for decorative use acceptable if title provides context)
- Overlay contrast: white text on `#00000061` overlay over image — must pass WCAG 2.2 AA (4.5:1 for normal text)

---

## Section 2: `main-collection` (existing)

Reused unchanged from `collection.json`. Configuration:

- Horizontal filter bar, full-width, `scheme-4`, sorting enabled
- Product card: square image, `scheme-2`, title + price row, subtitle metafield
- Grid: full-width, 1px gaps, large cards desktop / small mobile
- Pagination: standard (no infinite scroll)

---

## Template: `templates/collection.all.json`

Wires two sections in order:

1. `collection-all-hero` (new)
2. `main` using type `main-collection` (cloned from `collection.json`)

No `list-collections` section is used — this is a product grid, not a collection browser.

---

## Anti-Patterns

- Must NOT use `section.liquid` for the hero — that is Option A; this spec is Option B
- Must NOT hardcode hex colors outside of the overlay — use color scheme classes
- Must NOT add a second `<h1>` anywhere on the page
- Must NOT skip the overlay toggle — merchant must be able to disable it in theme editor

---

## QA Checklist

- [ ] `/collections/all` URL renders the new template (not `collection.json`)
- [ ] Hero shows background image with dark overlay
- [ ] Title renders as `<h1>Shop All</h1>` in Montserrat white
- [ ] Filters and product grid load correctly below the hero
- [ ] No second `<h1>` on the page
- [ ] White text on overlay passes WCAG 2.2 AA contrast
- [ ] Focus indicators visible on all interactive elements
- [ ] No layout break on mobile viewport
- [ ] Theme editor: image picker, overlay toggle, and title field all work
