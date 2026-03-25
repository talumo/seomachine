# Personalized Product Print File Generator — Design Spec

**Date:** 2026-03-25
**Status:** Approved
**Scope:** MVP — batch CLI, PDF output to local/shared folder

---

## Problem

Kelly Hughes sells personalized kids tableware bundles on Shopify. Each bundle contains up to 6
product types (plate, bowl, mug, placemat, spoon+fork). The customer enters one name at purchase;
that name appears as a Shopify line item property ("Personalization:"). Currently, staff manually
open up to 6 Photoshop PSD files per order and edit the name text layer individually. This is
slow, error-prone, and doesn't scale.

This tool automates that workflow: run one command, get print-ready PDFs for every pending order.

---

## Solution Overview

A standalone Python CLI tool that:

1. Fetches paid, unfulfilled Shopify orders via the Admin API
2. Skips already-processed orders (state tracked in a local JSON file)
3. For each eligible order, reads the personalization name and determines which tableware
   products were ordered and which design theme they belong to
4. Composites the name onto the pre-exported base image for each product using Pillow,
   with configurable letter spacing and font auto-sizing to fit within a defined text box
5. Embeds the composited image into a print-ready PDF at 300 DPI
6. Saves output to `output/YYYY-MM-DD/ORDER-{id}_{name}/` with one PDF per product
7. Zips each order folder for easy sharing/download

No web server. No database. Runs on any Mac or Windows machine with Python 3.9+.

---

## Project Location

This is a **new, standalone project** separate from the SEO Machine repository.
Create as: `print-generator/` (a new directory/repo).

---

## Directory Structure

```
print-generator/
├── run_batch.py               # CLI entry point
├── config.py                  # Loads .env, exposes typed config values
├── shopify_client.py          # Shopify Admin API — fetch and parse orders
├── template_manager.py        # Maps theme+product → base image + text config
├── text_renderer.py           # Font auto-sizing, letter spacing, PDF compositing
├── file_generator.py          # Orchestrates per-order file generation
├── state_manager.py           # Tracks processed order IDs to prevent re-runs
├── templates/
│   ├── bunny-love/
│   │   ├── plate.png          # PSD exported without name text layer, 300 DPI
│   │   ├── bowl.png
│   │   ├── mug.png
│   │   ├── placemat.png
│   │   └── spoon_fork.png
│   └── {theme-name}/          # One folder per additional theme, same structure
├── fonts/
│   └── personalization.ttf    # Font file provided by user (ttf or otf)
├── template_config.json       # All template settings — text boxes, font, spacing
├── .env                       # Credentials and paths (not committed)
├── .env.example               # Template for .env setup
├── processed_orders.json      # Runtime state — auto-created on first run
├── output/                    # Generated PDFs — gitignored
│   └── 2026-03-25/
│       └── ORDER-1234_Emma/
│           ├── plate.pdf
│           └── mug.pdf
├── requirements.txt
└── README.md                  # Setup guide including Shopify Custom App steps
```

---

## Configuration: `template_config.json`

All template settings live here. Text box coordinates come from the PSD file (measured once).
Settings cascade: product-level overrides theme-level overrides defaults.

```json
{
  "defaults": {
    "max_font_size": 72,
    "min_font_size": 18,
    "font_color": "#5A3E2B",
    "letter_spacing": 0
  },
  "themes": {
    "bunny-love": {
      "plate": {
        "template": "templates/bunny-love/plate.png",
        "text_box": { "x": 210, "y": 380, "width": 480, "height": 80 },
        "letter_spacing": 4
      },
      "bowl": {
        "template": "templates/bunny-love/bowl.png",
        "text_box": { "x": 180, "y": 310, "width": 420, "height": 70 }
      },
      "mug": {
        "template": "templates/bunny-love/mug.png",
        "text_box": { "x": 150, "y": 260, "width": 380, "height": 65 }
      },
      "placemat": {
        "template": "templates/bunny-love/placemat.png",
        "text_box": { "x": 300, "y": 520, "width": 600, "height": 90 }
      },
      "spoon_fork": {
        "template": "templates/bunny-love/spoon_fork.png",
        "text_box": { "x": 80, "y": 200, "width": 200, "height": 50 }
      }
    }
  },
  "product_mapping": {
    "plate": ["plate"],
    "bowl": ["bowl"],
    "mug": ["mug"],
    "placemat": ["placemat", "place mat"],
    "spoon_fork": ["spoon", "fork", "cutlery", "utensil"]
  }
}
```

**Notes:**
- Coordinates are in pixels at the base image's native resolution
- `letter_spacing` is in pixels; positive = more open, negative = tighter
- Theme name must match the folder name under `templates/`
- Product title matching (Shopify → product key) is case-insensitive keyword search

---

## Environment Variables: `.env`

```env
SHOPIFY_ACCESS_TOKEN=shpat_xxxxxxxxxxxxxxxxxxxx
SHOPIFY_STORE=your-store.myshopify.com
FONT_PATH=fonts/personalization.ttf
OUTPUT_DIR=output
```

---

## Module Responsibilities

### `config.py`
Loads `.env` via `python-dotenv`. Provides a `Config` dataclass with typed fields.
Raises a clear error at startup if required variables are missing.

### `shopify_client.py`

**`fetch_pending_orders(since_date=None) → list[Order]`**
- Calls `/admin/api/2024-01/orders.json?financial_status=paid&fulfillment_status=unfulfilled`
- Paginates using the `Link` header until all results are retrieved
- Optional `since_date` parameter adds `created_at_min` filter
- For each order, extracts:
  - `order_id`, `order_number`, `created_at`
  - Per line item: `title`, `properties` (looks for key `"Personalization"` or `"Name"`,
    case-insensitive)
- Returns only orders where at least one line item has a personalization value
- Skips line items where no personalization is found (logs a warning)

### `template_manager.py`

**`TemplateManager(config_path)`**
- Loads and validates `template_config.json` on init
- Raises `ConfigError` if a referenced template image file doesn't exist

**`resolve(product_title, theme_hint=None) → TemplateConfig | None`**
- Matches Shopify product title to a product key via `product_mapping` keyword rules
- Infers theme from product title keywords (e.g. "Bunny Love" → `bunny-love`)
- Returns merged config (defaults + theme + product) or `None` if no match (non-tableware)

### `text_renderer.py`

**`render(base_image_path, name, template_config) → PIL.Image`**

1. Open base image with Pillow
2. Load font at `max_font_size` using `PIL.ImageFont.truetype`
3. Measure total text width: sum of individual glyph widths + `(len(name) - 1) * letter_spacing`
4. If width exceeds `text_box.width`, reduce font size by 2pt and re-measure; repeat until fit
   or `min_font_size` reached (at min, text is truncated with a warning logged)
5. Calculate draw origin to center the text block horizontally and vertically within the text box
6. Draw each character individually, advancing x by `glyph_width + letter_spacing`
7. Return the composited image

### `file_generator.py`

**`generate_order(order, template_manager, config) → GenerationResult`**
- For each personalized line item in the order:
  - Calls `template_manager.resolve(title)` — skips if `None`
  - Calls `text_renderer.render(base_image, name, template_config)`
  - Converts rendered PIL image to PDF using `reportlab` (image embedded at source DPI)
  - Saves to `output/YYYY-MM-DD/ORDER-{order_id}_{name}/{product_key}.pdf`
- After all files written, zips the order folder
- Returns `GenerationResult` with counts of success/skipped/failed

### `state_manager.py`

**`StateManager(state_file_path)`**
- Reads `processed_orders.json` on init (creates empty file if not found)
- `is_processed(order_id) → bool`
- `mark_processed(order_id)` — writes immediately (not batched) to avoid data loss on error

### `run_batch.py` (CLI)

```
python run_batch.py                         # Process all new paid orders
python run_batch.py --dry-run               # Show what would be processed, no files written
python run_batch.py --order 1234            # Reprocess a specific order (bypasses state)
python run_batch.py --since 2026-03-01      # Only fetch orders created on/after this date
```

Flow:
1. Load config, validate env vars
2. Load template manager (validates all template files exist)
3. Fetch pending orders from Shopify
4. Filter out already-processed orders (unless `--order` flag)
5. For each order: generate files, mark processed
6. Print summary: X orders processed, Y files generated, Z skipped

---

## PDF Generation

Pillow renders the composited image (base template + name text). `reportlab` then embeds
that image into a single-page PDF, preserving the image's native DPI so print dimensions
are correct. The PDF page size matches the image dimensions in points (pixels ÷ DPI × 72).

---

## Theme Detection

Shopify product titles contain both theme and product type, e.g.:
- `"Bunny Love Plate Set"` → theme: `bunny-love`, product: `plate`
- `"Dinosaur Mug"` → theme: `dinosaur`, product: `mug`

Theme detection uses keyword matching against theme names defined in `template_config.json`.
The matching is case-insensitive with hyphens/spaces normalized. If no theme is matched,
the line item is skipped and a warning is logged (not a fatal error).

---

## Error Handling

| Scenario | Behavior |
|---|---|
| Shopify API rate limit (429) | Retry with exponential backoff, up to 3 retries |
| Order has no personalization value | Skip line item, log warning, continue |
| Product title doesn't match any template | Skip line item silently |
| Name too long even at min_font_size | Use min size, log warning with order ID + name |
| Template image file missing | Fail at startup with clear error (not per-order) |
| Output directory not writable | Fail at startup |

---

## Template Setup Guide (for README)

For each PSD file:
1. Open in Photoshop
2. Hide the name text layer (eye icon off)
3. File → Export → Export As → PNG, 300 DPI
4. Save to `templates/{theme-name}/{product-key}.png`
5. Re-show the text layer, note the bounding box position (x, y, width, height in pixels)
6. Enter those values in `template_config.json`

---

## Shopify Custom App Setup (for README)

1. Shopify Admin → Settings → Apps and sales channels → Develop apps
2. Click "Create an app" → name it "Print File Generator"
3. Configuration → Admin API integration → edit access scopes → enable `read_orders`
4. API credentials → "Install app" → copy the Admin API access token
5. Add to `.env`:
   ```
   SHOPIFY_ACCESS_TOKEN=shpat_...
   SHOPIFY_STORE=your-store.myshopify.com
   ```

---

## Dependencies

```
Pillow>=10.0.0
reportlab>=4.0.0
requests>=2.31.0
python-dotenv>=1.0.0
```

Python 3.9+ required. No other dependencies.

---

## Verification Plan

1. **Renderer test**: Run `text_renderer.py` standalone with a short name ("Em") and long name
   ("Bartholomew") against a sample template — inspect output image for correct fit and centering
2. **Letter spacing test**: Set `letter_spacing` to 0, 4, and -2 — verify visual difference
3. **Dry run**: `python run_batch.py --dry-run` — confirm Shopify connection, order count printed,
   no files written
4. **Single order**: `python run_batch.py --order {test_order_id}` — open output PDFs, verify
   name position, font size, and PDF opens correctly in Acrobat/Preview
5. **State test**: Run batch twice — confirm second run reports 0 new orders
6. **Mixed order test**: Use a test order containing both tableware and non-tableware line items —
   confirm only tableware products generate files
7. **Full batch**: Process all pending orders, review folder structure, open a sample PDF,
   confirm before sending to fulfillment

---

## Future Work (Out of Scope for MVP)

- **Order Desk integration**: swap disk-write adapter for Order Desk API upload
- **Shopify webhook**: trigger on `orders/paid` event instead of batch poll
- **Admin dashboard**: simple web UI to view pending orders and trigger generation per order
- **Email notification**: send generated zip to staff email after each batch run
