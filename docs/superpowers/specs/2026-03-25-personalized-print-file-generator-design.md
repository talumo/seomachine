# Personalized Product Print File Generator — Design Spec

**Date:** 2026-03-25
**Status:** Approved (v2 — post spec review)
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
5. Embeds the composited image into a print-ready PDF at the configured DPI
6. Saves output to `output/YYYY-MM-DD/ORDER-{id}_{sanitized_name}/` with one PDF per product
7. Zips each order folder as `ORDER-{id}_{sanitized_name}.zip` in the date folder

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
├── models.py                  # Shared dataclasses: Order, LineItem, TemplateConfig, GenerationResult
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
│       ├── ORDER-1234_Emma/
│       │   ├── plate.pdf
│       │   └── mug.pdf
│       └── ORDER-1234_Emma.zip
├── requirements.txt
└── README.md                  # Setup guide including Shopify Custom App steps
```

---

## Shared Dataclasses: `models.py`

```python
@dataclass
class LineItem:
    title: str           # Shopify product title, e.g. "Bunny Love Plate Set"
    name: str            # Personalization value, e.g. "Emma"
    # product_key is NOT on LineItem — resolution happens in file_generator via template_manager

@dataclass
class Order:
    order_id: str
    order_number: str
    created_at: str      # ISO 8601 UTC string from Shopify
    line_items: list[LineItem]   # Only items with a personalization value

@dataclass
class TemplateConfig:
    template_path: str
    text_box: dict       # {x, y, width, height} in pixels
    max_font_size: int
    min_font_size: int
    font_color: str      # Hex string e.g. "#5A3E2B"
    letter_spacing: int  # Pixels between characters; 0 = no extra spacing
    dpi: int             # Print DPI, e.g. 300

@dataclass
class GenerationResult:
    order_id: str
    files_generated: int
    files_skipped: int
    files_failed: int
    # orders_processed is NOT here — it is an aggregate count maintained by run_batch.py
    # across all GenerationResult instances in the batch loop
```

---

## Configuration: `template_config.json`

All template settings live here. Text box coordinates come from the PSD file (measured once).
Settings cascade: product-level overrides theme-level overrides defaults.

**New in v2:** Added `theme_mapping` for theme keyword detection, and explicit `dpi` per template.

```json
{
  "defaults": {
    "max_font_size": 72,
    "min_font_size": 18,
    "font_color": "#5A3E2B",
    "letter_spacing": 0,
    "dpi": 300
  },
  "theme_mapping": {
    "bunny-love": ["bunny", "bunny love"],
    "dinosaur": ["dinosaur", "dino"]
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
- Coordinates are in pixels at the base image's native resolution (export full canvas, no trimming)
- `dpi` is explicit in config — do not rely on PNG metadata, which is often stripped by Photoshop export
- `letter_spacing` is in pixels; positive = more open, negative = tighter
- Character-by-character drawing does not apply OpenType kerning pairs; compensate with `letter_spacing`
- Theme name in `theme_mapping` must match the key in `themes` and the folder name under `templates/`
- Product title matching is case-insensitive keyword search

---

## Environment Variables: `.env`

```env
SHOPIFY_ACCESS_TOKEN=shpat_xxxxxxxxxxxxxxxxxxxx
SHOPIFY_STORE=your-store.myshopify.com
SHOPIFY_API_VERSION=2024-04
FONT_PATH=fonts/personalization.ttf
OUTPUT_DIR=output
```

`SHOPIFY_API_VERSION` defaults to `2024-04` if not set but should be updated when Shopify
deprecates a version (check quarterly at shopify.dev/changelog).

---

## Module Responsibilities

### `config.py`
Loads `.env` via `python-dotenv`. Provides a `Config` dataclass with typed fields.
Raises a clear error at startup if required variables are missing.
Configures the `logging` module: WARNING+ to stderr, full log (DEBUG+) to `output/run.log`
(appended, not rotated, for MVP simplicity).

### `shopify_client.py`

**`fetch_pending_orders(since_date=None) → list[Order]`**
- Calls `/admin/api/{API_VERSION}/orders.json?financial_status=paid&fulfillment_status=unfulfilled`
- `since_date` appends `created_at_min` in ISO 8601 UTC format (e.g. `2026-03-01T00:00:00Z`);
  document in README that this filter is UTC-based
- Paginates using the `Link: rel="next"` header until all results are retrieved
- For each order, iterates line items and looks for a property with key matching
  `"Personalization"` or `"Name"` (case-insensitive); logs a WARNING if an item has no match
- Skips line items with no personalization value; includes all others regardless of product type
  (non-tableware items are filtered later in `template_manager.resolve()`)
- Returns `list[Order]` — may include orders with zero resolved line items (filtered in caller)

### `template_manager.py`

**`TemplateManager(config_path)`**
- Loads and validates `template_config.json` on init
- Raises `ConfigError` with a clear message if any referenced template image file doesn't exist

**`resolve(product_title) → TemplateConfig | None`**
- Step 1: Match theme — scan `theme_mapping` for any keyword that appears in `product_title`
  (case-insensitive, hyphens normalized to spaces). Returns `None` with a WARNING log if no
  theme matched.
- Step 2: Match product key — scan `product_mapping` for any keyword in `product_title`.
  Returns `None` with a WARNING log if no product key matched.
- Step 3: Merge and return — combine defaults + theme-level overrides + product-level overrides
  into a `TemplateConfig` dataclass

### `text_renderer.py`

**`render(base_image_path, name, config: TemplateConfig) → PIL.Image`**

1. Sanitize `name`: strip leading/trailing whitespace; if empty after strip, raise `ValueError`
2. Open base image with Pillow (`Image.open(...).convert("RGBA")`)
3. Load font at `config.max_font_size` using `PIL.ImageFont.truetype(font_path, size)`
4. Compute total text width: for each character, get width via `font.getbbox(char)[2]`;
   sum all widths + `(len(name) - 1) * config.letter_spacing`
5. While total width > `config.text_box["width"]` and font_size > `config.min_font_size`:
   reduce font_size by 2, reload font, recompute
6. If still over width at `min_font_size`: log WARNING with order context, proceed at min size
   (do not truncate the name — a too-large name is better than a truncated one for fulfillment)
7. Compute draw origin: `x = text_box.x + (text_box.width - total_width) / 2`,
   `y = text_box.y + (text_box.height - font_size) / 2`
8. Draw each character individually, advancing x by `char_width + letter_spacing`
9. Return composited RGBA image

**Unicode / missing glyphs**: Pillow silently renders a tofu box for missing glyphs. The spec
does not add a glyph check for MVP — log a note in README that the font must cover all
characters expected in customer names (including accented characters if applicable).

### `file_generator.py`

**`sanitize_name(name: str) → str`**
- Strip leading/trailing whitespace
- Replace any character not in `[A-Za-z0-9 _-]` with `_`
- Truncate to 40 characters
- Strip leading/trailing underscores from the result
- If the result is empty after all steps, fall back to `"unknown"` (prevents `ORDER-1234_.zip`)
- Used for folder and zip naming only; the original `name` is used for text rendering

**`generate_order(order: Order, template_manager, config) → GenerationResult`**
- Derive `sanitized_name` from the first line item's name (all items in one order share one name;
  if line items have differing names — unexpected but possible — log a WARNING and use the first)
- Create output folder: `output/YYYY-MM-DD/ORDER-{order.order_id}_{sanitized_name}/`
- For each line item in `order.line_items`:
  - Call `template_manager.resolve(line_item.title)` — if `None`, increment `files_skipped`,
    log WARNING (not silent), continue
  - Call `text_renderer.render(template_config.template_path, line_item.name, template_config)`
  - Flatten rendered RGBA image to RGB: `Image.new("RGB", img.size, (255,255,255)); rgb.paste(img, mask=img.split()[3])`
    — reportlab's `ImageReader` does not handle RGBA reliably; flattening to RGB on a white
    background produces correct, corruption-free print PDFs
  - Embed flattened RGB image into PDF via `reportlab`:
    - Page size in points: `(pixels_wide / dpi * 72, pixels_tall / dpi * 72)`
    - DPI comes from `template_config.dpi` (not PNG metadata)
    - Product key comes from the matched `TemplateConfig` (not from `LineItem`)
    - Save to `{order_folder}/{product_key}.pdf`
  - On success: increment `files_generated`; on exception: increment `files_failed`, log ERROR
- Zip entire order folder as `ORDER-{order_id}_{sanitized_name}.zip` in the date directory
- Return `GenerationResult`

### `state_manager.py`

**`StateManager(state_file_path)`**
- On init: attempt to load `processed_orders.json`; on `JSONDecodeError` or `FileNotFoundError`,
  log WARNING "state file unreadable, starting fresh" and initialize with empty set
- `is_processed(order_id: str) → bool`
- `mark_processed(order_id: str)` — appends to in-memory set and immediately rewrites the full
  JSON file (atomic via write-to-temp-file + rename on POSIX; on Windows, write directly)

### `run_batch.py` (CLI)

```
python run_batch.py                         # Process all new paid orders
python run_batch.py --dry-run               # Show what would be processed, no files written
python run_batch.py --order 1234            # Reprocess a specific order ID (bypasses state check;
                                            #   fetches that order directly; overwrites existing
                                            #   output; re-marks as processed)
python run_batch.py --since 2026-03-01      # Fetch orders on/after this date (UTC midnight)
```

Flow:
1. Load config, validate env vars — exit with clear error if missing
2. Load template manager — validates all template files exist at startup
3. Fetch orders from Shopify:
   - If `--order {id}`: use targeted endpoint `/orders/{id}.json` (single fetch, no pagination)
   - Otherwise: use list endpoint with `financial_status=paid&fulfillment_status=unfulfilled`
4. Filter out already-processed orders (unless `--order` flag bypasses)
5. For each order: call `generate_order`, then `state_manager.mark_processed`
6. Print summary to stdout: `X orders processed, Y files generated, Z skipped, W failed`
7. If any files failed: exit with code 1 so scripts/cron jobs can detect failures

---

## PDF Generation

Pillow renders the composited RGBA image (base template + name text). `reportlab` embeds
that image into a single-page PDF. Page dimensions are derived from
`template_config.dpi` (not PNG metadata) to ensure correct print sizing:

```
page_width_pt  = image.width  / dpi * 72
page_height_pt = image.height / dpi * 72
```

---

## Theme Detection

Shopify product titles contain both theme and product type, e.g.:
- `"Bunny Love Plate Set"` → theme: `bunny-love`, product: `plate`
- `"Dinosaur Mug"` → theme: `dinosaur`, product: `mug`

Theme detection uses keyword matching against `theme_mapping` in `template_config.json`.
Matching is case-insensitive with hyphens normalized to spaces. If no theme matched, the
line item is logged as a WARNING and skipped.

---

## Error Handling

| Scenario | Behavior |
|---|---|
| Shopify API rate limit (429) | Retry with exponential backoff, 3 retries (2s, 4s, 8s) |
| Shopify auth failure (401/403) | Exit immediately with clear error message |
| Order has no personalization value | Skip line item, log WARNING, continue |
| Product title doesn't match any template | Skip line item, log WARNING (not silent) |
| Name too long at min_font_size | Render at min size, log WARNING with order ID + name |
| Template image file missing | Fail at startup with clear error (not per-order) |
| Output directory not writable | Fail at startup |
| State file unreadable / corrupt JSON | Log WARNING, treat as empty, re-process all |
| Customer name with filesystem-unsafe chars | Sanitize for paths; use original for rendering |
| Line items in same order have different names | Use first name, log WARNING with order ID |

---

## Template Setup Guide (for README)

For each PSD file:
1. Open in Photoshop
2. Hide the name text layer (eye icon off)
3. File → Export → Export As → PNG
   - Set resolution to 300 DPI
   - **Important**: export the full canvas without trimming (disable "Canvas Size" crop)
4. Save to `templates/{theme-name}/{product-key}.png`
5. Re-show the text layer; measure the text box position in Photoshop's pixel coordinates
   (ruler origin = top-left corner of the canvas) — record x, y, width, height
6. Enter those values in `template_config.json` and set `dpi: 300`

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
   SHOPIFY_API_VERSION=2024-04
   ```
6. Update `SHOPIFY_API_VERSION` when Shopify notifies you of version deprecations

---

## Dependencies

```
Pillow>=10.0.0,<11.0.0
reportlab>=4.0.0,<5.0.0
requests>=2.31.0,<3.0.0
python-dotenv>=1.0.0,<2.0.0
```

Python 3.9+ required. No other dependencies.

---

## Verification Plan

**Happy path:**
1. **Renderer test**: Run `text_renderer.py` standalone with a short name ("Em") and long name
   ("Bartholomew") — inspect output image for correct fit and centering within text box
2. **Letter spacing test**: Set `letter_spacing` to 0, 4, and -2 — verify visible difference
3. **Dry run**: `python run_batch.py --dry-run` — Shopify connects, order count prints, no files written
4. **Single order**: `python run_batch.py --order {test_order_id}` — open PDFs, verify name
   position, font size, PDF dimensions correct in Acrobat/Preview (check actual print size)
5. **State test**: Run full batch twice — second run reports "0 orders to process"
6. **Mixed order test**: Test order with tableware + non-tableware line items — only tableware PDFs generated; WARNING logged for non-tableware items
7. **Full batch**: Process all pending orders, review folder/zip structure, open sample PDFs

**Negative / edge cases:**
8. **Corrupt state file**: Manually corrupt `processed_orders.json` → confirm WARNING logged and run completes successfully
9. **Wrong API token**: Set invalid `SHOPIFY_ACCESS_TOKEN` → confirm clear auth error (not Python traceback)
10. **Name with special characters**: Test an order with name "Chloë" or "Sofía" → confirm renders (or WARNING if font missing glyph) and folder name is safely sanitized
11. **British spelling**: If Shopify property key is `"personalisation"` → confirm WARNING logged with order ID

---

## Known Limitations (MVP)

- **No OpenType kerning**: character-by-character drawing bypasses font kerning pairs; compensate with `letter_spacing` config
- **Unicode coverage**: depends on the provided font; missing glyphs render silently as tofu boxes
- **`--since` is UTC**: orders placed near midnight in non-UTC timezones may be unexpectedly included/excluded
- **Single name per order**: all line items in one order use the same name; multi-name bundles are not supported

---

## Future Work (Out of Scope for MVP)

- **Order Desk integration**: swap disk-write adapter for Order Desk API upload
- **Shopify webhook**: trigger on `orders/paid` event instead of batch poll
- **Admin dashboard**: simple web UI to view pending orders and trigger generation per order
- **Email notification**: send generated zip to staff email after each batch run
