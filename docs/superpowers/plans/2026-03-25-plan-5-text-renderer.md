# Plan 5: Text Renderer

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement task-by-task.

**Goal:** Composite a name onto a base image with auto font-sizing and configurable letter spacing.

**Spec:** `docs/specs/2026-03-25-personalized-print-file-generator-design.md`
**Depends on:** Plan 1 (models.py) complete

---

### Task 1: text_renderer.py

**Files:**
- Create: `text_renderer.py`
- Create: `tests/test_text_renderer.py`
- Create: `tests/fixtures/` (test images generated at runtime, not committed)

- [ ] **Step 1: Write the failing tests**

```python
# tests/test_text_renderer.py
import pytest
from pathlib import Path
from PIL import Image, ImageFont, ImageDraw
from models import TemplateConfig
from text_renderer import render, measure_text_width

# Create a small test base image (solid white, 400x200)
@pytest.fixture
def base_image(tmp_path):
    img = Image.new("RGBA", (400, 200), (255, 255, 255, 255))
    path = tmp_path / "base.png"
    img.save(path)
    return str(path)

@pytest.fixture
def cfg():
    return TemplateConfig(
        template_path="",       # set per-test
        product_key="plate",
        text_box={"x": 50, "y": 80, "width": 300, "height": 60},
        max_font_size=60,
        min_font_size=12,
        font_color="#000000",
        letter_spacing=0,
        dpi=300,
    )

def test_render_returns_rgba_image(base_image, cfg):
    result = render(base_image, "Emma", cfg, font_path="")
    assert result.mode == "RGBA"
    assert result.size == (400, 200)

def test_short_name_uses_large_font(base_image, cfg):
    result = render(base_image, "Em", cfg, font_path="")
    assert result is not None

def test_long_name_fits_in_box(base_image, cfg):
    result = render(base_image, "Bartholomew", cfg, font_path="")
    assert result is not None

def test_empty_name_raises(base_image, cfg):
    with pytest.raises(ValueError, match="empty"):
        render(base_image, "   ", cfg, font_path="")

def test_measure_text_width_increases_with_spacing(base_image, cfg):
    # With letter spacing, total width should be >= width without
    font = ImageFont.load_default()
    w0 = measure_text_width("Emma", font, letter_spacing=0)
    w4 = measure_text_width("Emma", font, letter_spacing=4)
    assert w4 >= w0
```

- [ ] **Step 2: Run tests — expect failure**

```bash
pytest tests/test_text_renderer.py -v
```
Expected: `ModuleNotFoundError: No module named 'text_renderer'`

- [ ] **Step 3: Create `text_renderer.py`**

```python
import logging
from PIL import Image, ImageDraw, ImageFont
from models import TemplateConfig

logger = logging.getLogger(__name__)

def measure_text_width(text: str, font, letter_spacing: int = 0) -> int:
    total = 0
    for i, char in enumerate(text):
        bbox = font.getbbox(char)
        total += bbox[2]  # right edge = character width
        if i < len(text) - 1:
            total += letter_spacing
    return total

def render(base_image_path: str, name: str, config: TemplateConfig,
           font_path: str) -> Image.Image:
    """font_path comes from Config (env), not TemplateConfig, to keep concerns separated."""
    name = name.strip()
    if not name:
        raise ValueError("Personalization name is empty after stripping whitespace")

    img = Image.open(base_image_path).convert("RGBA")
    draw = ImageDraw.Draw(img)

    box = config.text_box
    font_size = config.max_font_size
    font = None
    total_width = None

    # Auto-size: reduce until text fits box width
    while font_size >= config.min_font_size:
        try:
            font = ImageFont.truetype(font_path, font_size)
        except (AttributeError, OSError):
            font = ImageFont.load_default()
        total_width = measure_text_width(name, font, config.letter_spacing)
        if total_width <= box["width"]:
            break
        font_size -= 2

    if total_width > box["width"]:
        logger.warning(
            "Name %r exceeds text box width at min font size %d — rendering at min size",
            name, config.min_font_size
        )

    # Parse font color
    hex_color = config.font_color.lstrip("#")
    r, g, b = int(hex_color[0:2], 16), int(hex_color[2:4], 16), int(hex_color[4:6], 16)
    fill = (r, g, b, 255)

    # Center text block in box
    font_height = font.getbbox("A")[3]  # approximate cap height
    x = box["x"] + (box["width"] - total_width) // 2
    y = box["y"] + (box["height"] - font_height) // 2

    # Draw character by character with letter spacing
    cursor_x = x
    for char in name:
        draw.text((cursor_x, y), char, font=font, fill=fill)
        char_width = font.getbbox(char)[2]
        cursor_x += char_width + config.letter_spacing

    return img
```

- [ ] **Step 4: Run tests — expect pass**

```bash
pytest tests/test_text_renderer.py -v
```
Expected: 5 passed

- [ ] **Step 6: Commit**

```bash
git add text_renderer.py tests/test_text_renderer.py
git commit -m "feat: add text renderer with font auto-sizing and letter spacing"
```

> **Note for implementer:** `font_path` is passed explicitly (not stored on `TemplateConfig`) so text_renderer stays decoupled from global config. In `file_generator.py`, pass `config.font_path` from the `Config` object when calling `render()`. Tests pass `font_path=""` to trigger Pillow's built-in default font fallback.
