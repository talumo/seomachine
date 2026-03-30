# Plan 7: run_batch CLI + README

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement task-by-task.

**Goal:** Wire all modules into a CLI entry point and write the README setup guide.

**Spec:** `docs/specs/2026-03-25-personalized-print-file-generator-design.md`
**Depends on:** Plans 1–6 complete

---

### Task 1: run_batch.py

**Files:**
- Create: `run_batch.py`
- Create: `tests/test_run_batch.py`

- [ ] **Step 1: Write the failing tests**

```python
# tests/test_run_batch.py
import sys
from unittest.mock import patch, MagicMock
from models import Order, LineItem, GenerationResult

def _make_order(order_id="1001", name="Emma"):
    return Order(
        order_id=order_id, order_number=f"#{order_id}",
        created_at="2026-03-25T10:00:00Z",
        line_items=[LineItem(title="Bunny Love Plate Set", name=name)]
    )

def test_dry_run_prints_orders_without_generating(capsys):
    with patch("run_batch.load_config") as mock_cfg, \
         patch("run_batch.TemplateManager"), \
         patch("run_batch.ShopifyClient") as mock_client_cls, \
         patch("run_batch.StateManager") as mock_state_cls:

        mock_cfg.return_value = MagicMock(output_dir="/tmp/out")
        mock_client = MagicMock()
        mock_client.fetch_pending_orders.return_value = [_make_order()]
        mock_client_cls.return_value = mock_client
        mock_state = MagicMock()
        mock_state.is_processed.return_value = False
        mock_state_cls.return_value = mock_state

        import run_batch
        run_batch.main(["--dry-run"])

    out = capsys.readouterr().out
    assert "1" in out  # 1 order found
    mock_state.mark_processed.assert_not_called()

def test_already_processed_orders_are_skipped(capsys):
    with patch("run_batch.load_config") as mock_cfg, \
         patch("run_batch.TemplateManager"), \
         patch("run_batch.ShopifyClient") as mock_client_cls, \
         patch("run_batch.StateManager") as mock_state_cls, \
         patch("run_batch.generate_order") as mock_gen:

        mock_cfg.return_value = MagicMock(output_dir="/tmp/out")
        mock_client = MagicMock()
        mock_client.fetch_pending_orders.return_value = [_make_order()]
        mock_client_cls.return_value = mock_client
        mock_state = MagicMock()
        mock_state.is_processed.return_value = True   # already done
        mock_state_cls.return_value = mock_state

        import run_batch
        run_batch.main([])

    mock_gen.assert_not_called()

def test_order_flag_uses_targeted_fetch():
    with patch("run_batch.load_config") as mock_cfg, \
         patch("run_batch.TemplateManager"), \
         patch("run_batch.ShopifyClient") as mock_client_cls, \
         patch("run_batch.StateManager"), \
         patch("run_batch.generate_order") as mock_gen:

        mock_cfg.return_value = MagicMock(output_dir="/tmp/out")
        mock_client = MagicMock()
        mock_client.fetch_order_by_id.return_value = [_make_order("9999")]
        mock_client_cls.return_value = mock_client
        mock_gen.return_value = GenerationResult("9999", 1, 0, 0)

        import run_batch
        run_batch.main(["--order", "9999"])

    mock_client.fetch_order_by_id.assert_called_once_with("9999")
    mock_client.fetch_pending_orders.assert_not_called()

def test_failed_files_exits_with_code_1():
    with patch("run_batch.load_config") as mock_cfg, \
         patch("run_batch.TemplateManager"), \
         patch("run_batch.ShopifyClient") as mock_client_cls, \
         patch("run_batch.StateManager") as mock_state_cls, \
         patch("run_batch.generate_order") as mock_gen:

        mock_cfg.return_value = MagicMock(output_dir="/tmp/out")
        mock_client = MagicMock()
        mock_client.fetch_pending_orders.return_value = [_make_order()]
        mock_client_cls.return_value = mock_client
        mock_state = MagicMock()
        mock_state.is_processed.return_value = False
        mock_state_cls.return_value = mock_state
        mock_gen.return_value = GenerationResult("1001", 0, 0, 1)  # 1 failed

        import run_batch
        import pytest
        with pytest.raises(SystemExit) as exc:
            run_batch.main([])
        assert exc.value.code == 1
```

- [ ] **Step 2: Run tests — expect failure**

```bash
pytest tests/test_run_batch.py -v
```
Expected: `ModuleNotFoundError: No module named 'run_batch'`

- [ ] **Step 3: Create `run_batch.py`**

```python
import argparse, sys, logging
from config import load_config
from shopify_client import ShopifyClient
from template_manager import TemplateManager, ConfigError
from file_generator import generate_order
from state_manager import StateManager

logger = logging.getLogger(__name__)

def main(argv=None):
    parser = argparse.ArgumentParser(
        description="Generate print-ready PDFs for personalized Shopify tableware orders."
    )
    parser.add_argument("--dry-run", action="store_true",
                        help="Show orders that would be processed without generating files")
    parser.add_argument("--order", metavar="ID",
                        help="Process a single order by ID (bypasses state check)")
    parser.add_argument("--since", metavar="DATE",
                        help="Only fetch orders on/after this UTC date (YYYY-MM-DD)")
    args = parser.parse_args(argv)

    config = load_config()

    try:
        tm = TemplateManager("template_config.json")
    except ConfigError as e:
        print(f"ERROR: {e}", file=sys.stderr)
        sys.exit(1)

    client = ShopifyClient(
        access_token=config.shopify_access_token,
        store=config.shopify_store,
        api_version=config.shopify_api_version,
    )
    state = StateManager("processed_orders.json")

    # Fetch orders
    if args.order:
        orders = client.fetch_order_by_id(args.order)
    else:
        orders = client.fetch_pending_orders(since_date=args.since)

    # Filter already-processed (unless --order forces reprocess)
    if not args.order:
        pending = [o for o in orders if not state.is_processed(o.order_id)]
    else:
        pending = orders

    if args.dry_run:
        print(f"Dry run: {len(pending)} order(s) would be processed")
        for o in pending:
            names = {i.name for i in o.line_items}
            print(f"  Order {o.order_number}: {len(o.line_items)} item(s), name(s): {names}")
        return

    orders_done = total_gen = total_skip = total_fail = 0
    for order in pending:
        result = generate_order(order, tm, config)
        state.mark_processed(order.order_id)
        orders_done += 1
        total_gen += result.files_generated
        total_skip += result.files_skipped
        total_fail += result.files_failed
        logger.info("Order %s: %d generated, %d skipped, %d failed",
                    order.order_id, result.files_generated,
                    result.files_skipped, result.files_failed)

    print(f"Done: {orders_done} order(s), "
          f"{total_gen} file(s) generated, "
          f"{total_skip} skipped, "
          f"{total_fail} failed")

    if total_fail > 0:
        sys.exit(1)

if __name__ == "__main__":
    main()
```

- [ ] **Step 4: Run tests — expect pass**

```bash
pytest tests/test_run_batch.py -v
```
Expected: 4 passed

- [ ] **Step 5: Run all tests**

```bash
pytest -v
```
Expected: All tests pass across all modules.

- [ ] **Step 6: Commit**

```bash
git add run_batch.py tests/test_run_batch.py
git commit -m "feat: add CLI entry point (run_batch.py)"
```

---

### Task 2: README.md

**Files:**
- Modify: `README.md`

- [ ] **Step 1: Replace README with full setup guide**

```markdown
# Personalize Print Files

Batch CLI tool that generates print-ready PDFs for personalized Shopify tableware orders.
Enter a name once per order — get one PDF per product, ready to send to your print partner.

## How It Works

1. Fetches paid, unfulfilled Shopify orders with a "Personalization" line item property
2. Matches each product to a design theme + product template (plate, bowl, mug, placemat, spoon+fork)
3. Composites the name onto the base template image with auto font-sizing and configurable letter spacing
4. Exports a print-ready PDF at 300 DPI for each product
5. Zips each order's files for easy sharing or download

## Requirements

- Python 3.9+
- Your font file (.ttf or .otf)
- Template base images (see Template Setup below)

## Installation

```bash
pip install -r requirements.txt
```

## Setup

### 1. Shopify Custom App

1. Shopify Admin → Settings → Apps and sales channels → Develop apps
2. Create an app named "Print File Generator"
3. Configuration → Admin API integration → enable `read_orders`
4. API credentials → Install app → copy the Admin API access token

### 2. Environment Variables

```bash
cp .env.example .env
```

Edit `.env`:
```
SHOPIFY_ACCESS_TOKEN=shpat_...
SHOPIFY_STORE=your-store.myshopify.com
SHOPIFY_API_VERSION=2024-04
FONT_PATH=fonts/YourFont.ttf
OUTPUT_DIR=output
```

Update `SHOPIFY_API_VERSION` when Shopify notifies you of deprecations (check quarterly at shopify.dev/changelog).

### 3. Template Setup

For each product PSD:
1. Open in Photoshop, hide the name text layer (eye icon off)
2. File → Export → Export As → PNG, 300 DPI, **full canvas, no trimming**
3. Save to `templates/{theme-name}/{product-key}.png`
   - Theme name: lowercase with hyphens (e.g. `bunny-love`)
   - Product keys: `plate`, `bowl`, `mug`, `placemat`, `spoon_fork`
4. Note the text box position in pixel coordinates (x, y, width, height from top-left)
5. Add to `template_config.json` under `themes → {theme-name} → {product-key}`

> **Important:** Export the full canvas without trimming. Coordinates must match the exported image dimensions, not the PSD artboard.

### 4. template_config.json

The scaffold `template_config.json` includes a `bunny-love` example. Update:
- `theme_mapping`: add keywords to identify each theme from a Shopify product title
- `themes`: add text box coordinates for each product
- `product_mapping`: keywords are already set for the 5 product types; add more if needed
- `defaults.letter_spacing`: adjust to taste (pixels between characters)
- `defaults.font_color`: hex color matching your design

### 5. Add Font File

Copy your `.ttf` or `.otf` file to the `fonts/` folder and set `FONT_PATH` in `.env`.

> **Unicode note:** The font must include all characters used in customer names. If a glyph is missing, Pillow renders a tofu box silently.

## Usage

```bash
# Process all new paid orders
python run_batch.py

# Preview without generating files
python run_batch.py --dry-run

# Reprocess a specific order
python run_batch.py --order 12345

# Only fetch orders since a date (UTC)
python run_batch.py --since 2026-03-01
```

Output is saved to `output/YYYY-MM-DD/ORDER-{id}_{name}/` with one PDF per product, plus a `.zip` of each order folder.

## Known Limitations

- **No OpenType kerning**: characters are drawn individually; compensate with `letter_spacing` in config
- **`--since` is UTC**: orders near midnight in non-UTC timezones may be included/excluded unexpectedly
- **Single name per order**: all products in one order share one name; multi-name bundles are not supported
```

- [ ] **Step 2: Commit**

```bash
git add README.md
git commit -m "docs: complete README with setup guide"
```

---

### Task 3: Push to GitHub

- [ ] **Step 1: Push all commits**

> **Before running:** substitute your actual GitHub Personal Access Token for `YOUR_TOKEN_HERE`. Do not run this command with the placeholder as-is.

```bash
git push https://YOUR_TOKEN_HERE@github.com/talumo/personalize-print-files.git main
```

Or set it as an env var first:
```bash
export GITHUB_TOKEN=ghp_...
git push https://$GITHUB_TOKEN@github.com/talumo/personalize-print-files.git main
```
