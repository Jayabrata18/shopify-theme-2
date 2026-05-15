# Collections/All Page Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Create a `/collections/all` page with a full-width image hero and the standard filtered product grid.

**Architecture:** Two files — a new `sections/collection-all-hero.liquid` (custom hero: background image, dark overlay, h1 title, schema-controlled padding) and `templates/collection.all.json` (template JSON wiring hero + existing `main-collection` section). No JavaScript required. Snippets `background-media` and `overlay` are reused from the existing theme.

**Tech Stack:** Shopify Liquid, theme JSON templates, existing theme snippets (`background-media`, `overlay`), `{% stylesheet %}` blocks for scoped CSS.

---

## File Map

| Action | Path | Responsibility |
|---|---|---|
| Create | `sections/collection-all-hero.liquid` | Hero: background image, overlay, h1 |
| Create | `templates/collection.all.json` | Wires hero + main-collection for `/collections/all` |

---

### Task 1: Create `sections/collection-all-hero.liquid`

**Files:**
- Create: `sections/collection-all-hero.liquid`

This section renders a full-width hero with a background image (via the existing `background-media` snippet), a dark overlay (via the existing `overlay` snippet), and a configurable `<h1>` title. It follows the same structural pattern as `snippets/section.liquid` but is purpose-built for the Shop All hero.

- [ ] **Step 1: Create the file with the full liquid + stylesheet + schema**

Create `sections/collection-all-hero.liquid` with the following content:

```liquid
<div class="section-background color-{{ section.settings.color_scheme }}"></div>

<div
  class="collection-all-hero section--full-width color-{{ section.settings.color_scheme }}"
  style="
    --hero-padding-block-start: {{ section.settings.padding-block-start }}px;
    --hero-padding-block-end: {{ section.settings.padding-block-end }}px;
    --hero-padding-inline-start: {{ section.settings.padding-inline-start }}px;
  "
>
  <div class="collection-all-hero__background">
    {% render 'background-media',
      background_media: 'image',
      background_image: section.settings.background_image,
      background_image_position: 'cover',
      section: section
    %}
  </div>

  <div class="collection-all-hero__content">
    {% if section.settings.toggle_overlay %}
      {% render 'overlay', settings: section.settings, layer: '0' %}
    {% endif %}

    <h1 class="collection-all-hero__title">
      {{ section.settings.title | default: 'Shop All' | escape }}
    </h1>
  </div>
</div>

{% stylesheet %}
  .collection-all-hero {
    position: relative;
    width: 100%;
    display: grid;
  }

  .collection-all-hero__background {
    grid-column: 1 / -1;
    grid-row: 1 / -1;
    overflow: hidden;
  }

  .collection-all-hero__background img,
  .collection-all-hero__background .placeholder-svg {
    width: 100%;
    height: 100%;
    object-fit: cover;
    display: block;
  }

  .collection-all-hero__content {
    grid-column: 1 / -1;
    grid-row: 1 / -1;
    position: relative;
    z-index: var(--layer-flat);
    padding-block-start: var(--hero-padding-block-start, 100px);
    padding-block-end: var(--hero-padding-block-end, 32px);
    padding-inline-start: var(--hero-padding-inline-start, 40px);
    padding-inline-end: 40px;
  }

  .collection-all-hero__title {
    font-family: var(--font-primary--family, 'Montserrat', sans-serif);
    font-size: 19px;
    font-weight: 400;
    line-height: 1.2;
    color: #ffffff;
    margin: 0;
    text-wrap: pretty;
  }
{% endstylesheet %}

{% schema %}
{
  "name": "Shop All Hero",
  "tag": "section",
  "class": "collection-all-hero-wrapper",
  "settings": [
    {
      "type": "header",
      "content": "Hero Image"
    },
    {
      "type": "image_picker",
      "id": "background_image",
      "label": "Background image"
    },
    {
      "type": "checkbox",
      "id": "toggle_overlay",
      "label": "Show overlay",
      "default": true
    },
    {
      "type": "color",
      "id": "overlay_color",
      "label": "Overlay color",
      "default": "#000000"
    },
    {
      "type": "select",
      "id": "overlay_style",
      "label": "Overlay style",
      "options": [
        { "value": "solid", "label": "Solid" },
        { "value": "gradient", "label": "Gradient" }
      ],
      "default": "solid"
    },
    {
      "type": "header",
      "content": "Title"
    },
    {
      "type": "text",
      "id": "title",
      "label": "Title",
      "default": "Shop All"
    },
    {
      "type": "header",
      "content": "Color"
    },
    {
      "type": "color_scheme",
      "id": "color_scheme",
      "label": "Color scheme",
      "default": "scheme-5"
    },
    {
      "type": "header",
      "content": "Padding"
    },
    {
      "type": "range",
      "id": "padding-block-start",
      "label": "Top padding",
      "min": 0,
      "max": 200,
      "step": 4,
      "unit": "px",
      "default": 100
    },
    {
      "type": "range",
      "id": "padding-block-end",
      "label": "Bottom padding",
      "min": 0,
      "max": 100,
      "step": 4,
      "unit": "px",
      "default": 32
    },
    {
      "type": "range",
      "id": "padding-inline-start",
      "label": "Left padding",
      "min": 0,
      "max": 100,
      "step": 4,
      "unit": "px",
      "default": 40
    }
  ],
  "disabled_on": {
    "groups": ["header", "footer"]
  }
}
{% endschema %}
```

- [ ] **Step 2: Verify the file is valid — check schema JSON parses**

Run:
```bash
node -e "
const fs = require('fs');
const src = fs.readFileSync('sections/collection-all-hero.liquid', 'utf8');
const schemaMatch = src.match(/{%- schema -%}|{% schema %}([\s\S]*?){% endschema %}/);
if (!schemaMatch) { console.error('No schema found'); process.exit(1); }
const json = src.split('{% schema %}')[1].split('{% endschema %}')[0].trim();
JSON.parse(json);
console.log('Schema JSON valid');
" 2>&1
```
Expected output: `Schema JSON valid`

- [ ] **Step 3: Commit**

```bash
git add sections/collection-all-hero.liquid
git commit -m "feat: add collection-all-hero section for /collections/all hero"
```

---

### Task 2: Create `templates/collection.all.json`

**Files:**
- Create: `templates/collection.all.json`

Shopify resolves `/collections/all` using the template whose filename matches the collection handle. `collection.all.json` overrides the default `collection.json` for this specific URL.

- [ ] **Step 1: Create the template JSON**

Create `templates/collection.all.json` with the following content (the `main` section is copied verbatim from `collection.json` — both sections must be present for the page to render correctly):

```json
{
  "sections": {
    "hero": {
      "type": "collection-all-hero",
      "name": "Shop All Hero",
      "settings": {
        "title": "Shop All",
        "toggle_overlay": true,
        "overlay_color": "#000000",
        "overlay_style": "solid",
        "color_scheme": "scheme-5",
        "padding-block-start": 100,
        "padding-block-end": 32,
        "padding-inline-start": 40
      }
    },
    "main": {
      "type": "main-collection",
      "blocks": {
        "filters": {
          "type": "filters",
          "static": true,
          "settings": {
            "enable_filtering": true,
            "filter_style": "horizontal",
            "filter_width": "full-width",
            "text_label_case": "default",
            "show_swatch_label": false,
            "show_filter_label": false,
            "enable_sorting": true,
            "enable_grid_density": false,
            "inherit_color_scheme": false,
            "color_scheme": "scheme-4",
            "padding-block-start": 8,
            "padding-block-end": 8,
            "padding-inline-start": 40,
            "padding-inline-end": 40,
            "facets_margin_bottom": 0,
            "facets_margin_right": 0
          },
          "blocks": {}
        },
        "product-card": {
          "type": "_product-card",
          "static": true,
          "settings": {
            "product_card_gap": 0,
            "inherit_color_scheme": false,
            "color_scheme": "scheme-2",
            "border": "none",
            "border_width": 1,
            "border_opacity": 100,
            "border_radius": 0,
            "padding-block-start": 0,
            "padding-block-end": 0,
            "padding-inline-start": 0,
            "padding-inline-end": 0
          },
          "blocks": {
            "product_card_gallery": {
              "type": "_product-card-gallery",
              "name": "t:names.product_card_media",
              "settings": {
                "image_ratio": "square",
                "border": "solid",
                "border_width": 0,
                "border_opacity": 0,
                "border_radius": 0,
                "padding-block-start": 0,
                "padding-block-end": 0,
                "padding-inline-start": 0,
                "padding-inline-end": 0
              },
              "blocks": {}
            },
            "group_info": {
              "type": "_product-card-group",
              "name": "t:names.group",
              "settings": {
                "open_in_new_tab": false,
                "content_direction": "column",
                "vertical_on_mobile": true,
                "horizontal_alignment": "space-between",
                "vertical_alignment": "flex-start",
                "align_baseline": false,
                "horizontal_alignment_flex_direction_column": "flex-start",
                "vertical_alignment_flex_direction_column": "space-between",
                "gap": 4,
                "width": "fill",
                "custom_width": 100,
                "width_mobile": "fill",
                "custom_width_mobile": 100,
                "height": "fit",
                "custom_height": 100,
                "inherit_color_scheme": false,
                "color_scheme": "scheme-2",
                "background_media": "none",
                "video_position": "cover",
                "background_image_position": "cover",
                "border": "none",
                "border_width": 1,
                "border_opacity": 100,
                "border_radius": 0,
                "toggle_overlay": false,
                "overlay_color": "#00000026",
                "overlay_style": "solid",
                "gradient_direction": "to top",
                "padding-block-start": 12,
                "padding-block-end": 12,
                "padding-inline-start": 12,
                "padding-inline-end": 12
              },
              "blocks": {
                "group_title_price": {
                  "type": "_product-card-group",
                  "name": "t:names.group",
                  "settings": {
                    "open_in_new_tab": false,
                    "content_direction": "row",
                    "vertical_on_mobile": true,
                    "horizontal_alignment": "space-between",
                    "vertical_alignment": "center",
                    "align_baseline": false,
                    "horizontal_alignment_flex_direction_column": "flex-start",
                    "vertical_alignment_flex_direction_column": "flex-start",
                    "gap": 8,
                    "width": "fill",
                    "custom_width": 75,
                    "width_mobile": "fill",
                    "custom_width_mobile": 75,
                    "height": "fit",
                    "custom_height": 100,
                    "inherit_color_scheme": true,
                    "color_scheme": "",
                    "background_media": "none",
                    "video_position": "cover",
                    "background_image_position": "cover",
                    "border": "none",
                    "border_width": 1,
                    "border_opacity": 100,
                    "border_radius": 0,
                    "toggle_overlay": false,
                    "overlay_color": "#00000026",
                    "overlay_style": "solid",
                    "gradient_direction": "to top",
                    "padding-block-start": 0,
                    "padding-block-end": 0,
                    "padding-inline-start": 0,
                    "padding-inline-end": 0
                  },
                  "blocks": {
                    "product_title": {
                      "type": "product-title",
                      "name": "t:names.product_title",
                      "settings": {
                        "width": "fit-content",
                        "max_width": "normal",
                        "alignment": "left",
                        "type_preset": "h5",
                        "font": "var(--font-body--family)",
                        "font_size": "1rem",
                        "line_height": "normal",
                        "letter_spacing": "normal",
                        "case": "none",
                        "wrap": "pretty",
                        "color": "var(--color-foreground)",
                        "background": false,
                        "background_color": "#00000026",
                        "corner_radius": 0,
                        "padding-block-start": 0,
                        "padding-block-end": 0,
                        "padding-inline-start": 0,
                        "padding-inline-end": 0
                      },
                      "blocks": {}
                    },
                    "price": {
                      "type": "price",
                      "name": "t:names.product_price",
                      "settings": {
                        "show_sale_price_first": true,
                        "show_installments": false,
                        "show_tax_info": false,
                        "type_preset": "paragraph",
                        "width": "fit-content",
                        "alignment": "right",
                        "font": "var(--font-body--family)",
                        "font_size": "1rem",
                        "line_height": "normal",
                        "letter_spacing": "normal",
                        "case": "none",
                        "color": "var(--color-foreground)",
                        "padding-block-start": 0,
                        "padding-block-end": 0,
                        "padding-inline-start": 0,
                        "padding-inline-end": 0
                      },
                      "blocks": {}
                    }
                  },
                  "block_order": ["product_title", "price"]
                },
                "group_subtitle": {
                  "type": "_product-card-group",
                  "name": "t:names.group",
                  "settings": {
                    "open_in_new_tab": false,
                    "content_direction": "column",
                    "vertical_on_mobile": true,
                    "horizontal_alignment": "flex-start",
                    "vertical_alignment": "center",
                    "align_baseline": false,
                    "horizontal_alignment_flex_direction_column": "flex-start",
                    "vertical_alignment_flex_direction_column": "flex-start",
                    "gap": 12,
                    "width": "fit-content",
                    "custom_width": 100,
                    "width_mobile": "fill",
                    "custom_width_mobile": 25,
                    "height": "fit",
                    "custom_height": 100,
                    "inherit_color_scheme": true,
                    "color_scheme": "",
                    "background_media": "none",
                    "video_position": "cover",
                    "background_image_position": "cover",
                    "border": "none",
                    "border_width": 1,
                    "border_opacity": 100,
                    "border_radius": 0,
                    "toggle_overlay": false,
                    "overlay_color": "#00000026",
                    "overlay_style": "solid",
                    "gradient_direction": "to top",
                    "padding-block-start": 0,
                    "padding-block-end": 0,
                    "padding-inline-start": 0,
                    "padding-inline-end": 0
                  },
                  "blocks": {
                    "subtitle": {
                      "type": "text",
                      "name": "t:names.text",
                      "settings": {
                        "text": "<p>{{ closest.product.metafields.descriptors.subtitle.value }}</p>",
                        "width": "fit-content",
                        "max_width": "none",
                        "alignment": "left",
                        "type_preset": "custom",
                        "font": "var(--font-primary--family)",
                        "font_size": "var(--font-size--body-md)",
                        "line_height": "normal",
                        "letter_spacing": "normal",
                        "case": "none",
                        "wrap": "pretty",
                        "color": "",
                        "background": false,
                        "background_color": "#00000026",
                        "corner_radius": 0,
                        "padding-block-start": 0,
                        "padding-block-end": 0,
                        "padding-inline-start": 0,
                        "padding-inline-end": 0
                      },
                      "blocks": {}
                    }
                  },
                  "block_order": ["subtitle"]
                }
              },
              "block_order": ["group_title_price", "group_subtitle"]
            }
          },
          "block_order": ["product_card_gallery", "group_info"]
        }
      },
      "settings": {
        "layout_type": "grid",
        "product_card_size": "large",
        "mobile_product_card_size": "small",
        "product_grid_width": "full-width",
        "full_width_on_mobile": true,
        "columns_gap_horizontal": 1,
        "columns_gap_vertical": 1,
        "padding-inline-start": 0,
        "padding-inline-end": 0,
        "color_scheme": "",
        "padding-block-start": 0,
        "padding-block-end": 1
      }
    }
  },
  "order": ["hero", "main"]
}
```

- [ ] **Step 2: Verify the JSON is valid**

Run:
```bash
node -e "
const fs = require('fs');
const src = fs.readFileSync('templates/collection.all.json', 'utf8');
JSON.parse(src);
console.log('JSON valid');
"
```
Expected: `JSON valid`

- [ ] **Step 3: Verify section order is correct**

Run:
```bash
node -e "
const t = JSON.parse(require('fs').readFileSync('templates/collection.all.json','utf8'));
console.log('order:', t.order);
console.log('hero type:', t.sections.hero.type);
console.log('main type:', t.sections.main.type);
"
```
Expected:
```
order: [ 'hero', 'main' ]
hero type: collection-all-hero
main type: main-collection
```

- [ ] **Step 4: Commit**

```bash
git add templates/collection.all.json
git commit -m "feat: add collection.all.json template for /collections/all page"
```

---

### Task 3: Visual Verification

**Files:** No changes — verification only.

Since this is a Shopify Liquid theme with no automated template test runner, verification is done via the Shopify CLI preview or theme push.

- [ ] **Step 1: Push to development theme and open preview**

```bash
shopify theme dev --store your-store.myshopify.com
```

Then open: `http://localhost:9292/collections/all`

- [ ] **Step 2: Run QA checklist**

| Check | Pass? |
|---|---|
| `/collections/all` uses the new template (hero visible above grid) | |
| Hero shows background image with dark overlay | |
| Title renders as `<h1>Shop All</h1>` |  |
| Font is Montserrat (inspect: `font-family`) | |
| Title color is `#ffffff` | |
| Filters and product grid load below the hero | |
| No second `<h1>` on the page (inspect with DevTools) | |
| Overlay toggle in theme editor hides/shows overlay | |
| Image picker in theme editor updates the background | |
| Title field in theme editor updates the h1 text | |
| No layout break at 375px mobile width | |
| Focus ring visible when tabbing through the page | |

- [ ] **Step 3: Confirm no regression on `/collections` and other collection pages**

Open `/collections` (list-collections) and any individual collection URL — they should still use their own templates, unaffected.
