# Plan 6: File Generator

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement task-by-task.

**Goal:** Orchestrate per-order PDF generation — RGBA→RGB flatten, reportlab embed, zip output folder.

**Spec:** `docs/specs/2026-03-25-personalized-print-file-generator-design.md`
**Depends on:** Plans 1–5 complete (models, template_manager, text_renderer)

---

### Task 1: file_generator.py

**Files:**
- Create: `file_generator.py`
- Create: `tests/test_file_generator.py`

- [ ] **Step 1: Write the failing tests**

```python
# tests/test_file_generator.py
import zipfile
from pathlib import Path
from unittest.mock import patch, MagicMock
from PIL import Image
from models import Order, LineItem, TemplateConfig, GenerationResult
from file_generator import sanitize_name, generate_order

# --- sanitize_name tests ---

def test_sanitize_normal_name():
    assert sanitize_name("Emma") == "Emma"

def test_sanitize_strips_unsafe_chars():
    result = sanitize_name("Chloë/test")
    assert "/" not in result
    assert len(result) > 0

def test_sanitize_truncates_to_40():
    long_name = "A" * 50
    assert len(sanitize_name(long_name)) <= 40

def test_sanitize_empty_fallback():
    assert sanitize_name("!!!") == "unknown"
    assert sanitize_name("   ") == "unknown"

def test_sanitize_strips_edge_underscores():
    result = sanitize_name("!!Emma!!")
    assert not result.startswith("_")
    assert not result.endswith("_")

# --- generate_order tests ---

def _make_template_config(tmp_path):
    # Create a real small base image for the test
    img_path = tmp_path / "plate.png"
    Image.new("RGBA", (100, 100), (255, 255, 255, 255)).save(img_path)
    return TemplateConfig(
        template_path=str(img_path),
        product_key="plate",
        text_box={"x": 10, "y": 10, "width": 80, "height": 30},
        max_font_size=24, min_font_size=8,
        font_color="#000000", letter_spacing=0, dpi=300,
    )

def test_generate_order_creates_pdf_and_zip(tmp_path):
    cfg_mock = MagicMock()
    cfg_mock.font_path = ""
    cfg_mock.output_dir = str(tmp_path / "output")

    tmpl_cfg = _make_template_config(tmp_path)
    tm = MagicMock()
    tm.resolve.return_value = tmpl_cfg

    order = Order(
        order_id="1001", order_number="#1001",
        created_at="2026-03-25T10:00:00Z",
        line_items=[LineItem(title="Bunny Love Plate Set", name="Emma")]
    )

    result = generate_order(order, tm, cfg_mock)

    assert result.files_generated == 1
    assert result.files_skipped == 0
    assert result.files_failed == 0

    # Check PDF exists
    output_root = Path(cfg_mock.output_dir)
    pdfs = list(output_root.rglob("*.pdf"))
    assert len(pdfs) == 1
    assert pdfs[0].name == "plate.pdf"

    # Check zip exists
    zips = list(output_root.rglob("*.zip"))
    assert len(zips) == 1
    with zipfile.ZipFile(zips[0]) as z:
        assert any(n.endswith("plate.pdf") for n in z.namelist())

def test_unresolved_product_counted_as_skipped(tmp_path):
    cfg_mock = MagicMock()
    cfg_mock.font_path = ""
    cfg_mock.output_dir = str(tmp_path / "output")

    tm = MagicMock()
    tm.resolve.return_value = None   # no template match

    order = Order(
        order_id="1002", order_number="#1002",
        created_at="2026-03-25T10:00:00Z",
        line_items=[LineItem(title="Unknown Sticker", name="Emma")]
    )
    result = generate_order(order, tm, cfg_mock)
    assert result.files_skipped == 1
    assert result.files_generated == 0

def test_differing_names_uses_first_logs_warning(tmp_path, caplog):
    import logging
    cfg_mock = MagicMock()
    cfg_mock.font_path = ""
    cfg_mock.output_dir = str(tmp_path / "output")

    tmpl_cfg = _make_template_config(tmp_path)
    tm = MagicMock()
    tm.resolve.return_value = tmpl_cfg

    order = Order(
        order_id="1003", order_number="#1003",
        created_at="2026-03-25T10:00:00Z",
        line_items=[
            LineItem(title="Bunny Love Plate Set", name="Emma"),
            LineItem(title="Bunny Love Mug", name="Sophie"),  # different name
        ]
    )
    with caplog.at_level(logging.WARNING):
        result = generate_order(order, tm, cfg_mock)
    assert "differing" in caplog.text.lower() or "different" in caplog.text.lower()
```

- [ ] **Step 2: Run tests — expect failure**

```bash
pytest tests/test_file_generator.py -v
```
Expected: `ModuleNotFoundError: No module named 'file_generator'`

- [ ] **Step 3: Create `file_generator.py`**

```python
import logging, re, zipfile
from datetime import datetime
from pathlib import Path
from PIL import Image
from reportlab.pdfgen import canvas as rl_canvas
from reportlab.lib.units import inch
from models import Order, GenerationResult
from text_renderer import render

logger = logging.getLogger(__name__)

def sanitize_name(name: str) -> str:
    name = name.strip()
    name = re.sub(r"[^A-Za-z0-9 _-]", "_", name)
    name = name[:40]
    name = name.strip("_").strip()
    return name if name else "unknown"

def generate_order(order: Order, template_manager, config) -> GenerationResult:
    # Determine folder name from first line item's name
    first_name = order.line_items[0].name if order.line_items else "unknown"
    names = {item.name for item in order.line_items}
    if len(names) > 1:
        logger.warning(
            "Order %s has differing personalization names %r — using first: %r",
            order.order_id, names, first_name
        )

    safe_name = sanitize_name(first_name)
    date_str = datetime.utcnow().strftime("%Y-%m-%d")
    folder_name = f"ORDER-{order.order_id}_{safe_name}"
    order_dir = Path(config.output_dir) / date_str / folder_name
    order_dir.mkdir(parents=True, exist_ok=True)

    generated = skipped = failed = 0

    for item in order.line_items:
        tmpl = template_manager.resolve(item.title)
        if tmpl is None:
            logger.warning("Order %s: no template for %r — skipping",
                           order.order_id, item.title)
            skipped += 1
            continue
        try:
            composited = render(tmpl.template_path, item.name, tmpl, config.font_path)
            rgb = _flatten_to_rgb(composited)
            pdf_path = order_dir / f"{tmpl.product_key}.pdf"
            _save_as_pdf(rgb, pdf_path, tmpl.dpi)
            generated += 1
        except Exception as e:
            logger.error("Order %s: failed to generate %r — %s",
                         order.order_id, item.title, e)
            failed += 1

    # Zip the order folder
    zip_path = order_dir.parent / f"{folder_name}.zip"
    with zipfile.ZipFile(zip_path, "w", zipfile.ZIP_DEFLATED) as zf:
        for f in order_dir.rglob("*"):
            zf.write(f, arcname=f.relative_to(order_dir.parent))

    return GenerationResult(
        order_id=order.order_id,
        files_generated=generated,
        files_skipped=skipped,
        files_failed=failed,
    )

def _flatten_to_rgb(img: Image.Image) -> Image.Image:
    """Flatten RGBA onto white background for reportlab compatibility."""
    background = Image.new("RGB", img.size, (255, 255, 255))
    background.paste(img, mask=img.split()[3])
    return background

def _save_as_pdf(img: Image.Image, path: Path, dpi: int):
    width_pt = img.width / dpi * 72
    height_pt = img.height / dpi * 72
    c = rl_canvas.Canvas(str(path), pagesize=(width_pt, height_pt))
    c.drawInlineImage(img, 0, 0, width=width_pt, height=height_pt)
    c.save()
```

- [ ] **Step 4: Run tests — expect pass**

```bash
pytest tests/test_file_generator.py -v
```
Expected: 8 passed

- [ ] **Step 5: Commit**

```bash
git add file_generator.py tests/test_file_generator.py
git commit -m "feat: add file generator with PDF output, RGBA flatten, and zip"
```
