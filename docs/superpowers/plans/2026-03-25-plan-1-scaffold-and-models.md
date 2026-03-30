# Plan 1: Project Scaffold + Models

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement task-by-task.

**Goal:** Set up the project structure, dependencies, and shared dataclasses.

**Spec:** `docs/specs/2026-03-25-personalized-print-file-generator-design.md`

---

### Task 1: Create project files

**Files:**
- Create: `requirements.txt`
- Create: `.gitignore`
- Create: `.env.example`
- Create: `template_config.json`
- Create: `tests/__init__.py`

- [ ] **Step 1: Create `requirements.txt`**

```
Pillow>=10.0.0,<11.0.0
reportlab>=4.0.0,<5.0.0
requests>=2.31.0,<3.0.0
python-dotenv>=1.0.0,<2.0.0
pytest>=7.0.0,<9.0.0
```

- [ ] **Step 2: Create `.gitignore`**

```
.env
output/
processed_orders.json
__pycache__/
*.pyc
.pytest_cache/
*.egg-info/
dist/
fonts/
templates/
```

- [ ] **Step 3: Create `.env.example`**

```env
SHOPIFY_ACCESS_TOKEN=shpat_xxxxxxxxxxxxxxxxxxxx
SHOPIFY_STORE=your-store.myshopify.com
SHOPIFY_API_VERSION=2024-04
FONT_PATH=fonts/personalization.ttf
OUTPUT_DIR=output
```

- [ ] **Step 4: Create `template_config.json`**

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
    "bunny-love": ["bunny", "bunny love"]
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

- [ ] **Step 5: Create `tests/__init__.py`** (empty file)

- [ ] **Step 6: Install dependencies**

```bash
pip install -r requirements.txt
```

- [ ] **Step 7: Commit**

```bash
git add requirements.txt .gitignore .env.example template_config.json tests/
git commit -m "feat: project scaffold"
```

---

### Task 2: models.py

**Files:**
- Create: `models.py`
- Create: `tests/test_models.py`

- [ ] **Step 1: Write the failing test**

```python
# tests/test_models.py
from models import LineItem, Order, TemplateConfig, GenerationResult

def test_line_item_fields():
    item = LineItem(title="Bunny Love Plate Set", name="Emma")
    assert item.title == "Bunny Love Plate Set"
    assert item.name == "Emma"

def test_order_fields():
    item = LineItem(title="Bunny Love Plate Set", name="Emma")
    order = Order(order_id="1234", order_number="#1234",
                  created_at="2026-03-25T10:00:00Z", line_items=[item])
    assert order.order_id == "1234"
    assert len(order.line_items) == 1

def test_template_config_fields():
    cfg = TemplateConfig(
        template_path="templates/bunny-love/plate.png",
        product_key="plate",
        text_box={"x": 210, "y": 380, "width": 480, "height": 80},
        max_font_size=72, min_font_size=18,
        font_color="#5A3E2B", letter_spacing=4, dpi=300
    )
    assert cfg.product_key == "plate"
    assert cfg.dpi == 300

def test_generation_result_defaults():
    result = GenerationResult(order_id="1234",
                               files_generated=2, files_skipped=1, files_failed=0)
    assert result.files_generated == 2
    assert result.files_failed == 0
```

- [ ] **Step 2: Run test — expect failure**

```bash
pytest tests/test_models.py -v
```
Expected: `ModuleNotFoundError: No module named 'models'`

- [ ] **Step 3: Create `models.py`**

```python
from dataclasses import dataclass

@dataclass
class LineItem:
    title: str   # Shopify product title, e.g. "Bunny Love Plate Set"
    name: str    # Personalization value, e.g. "Emma"

@dataclass
class Order:
    order_id: str
    order_number: str
    created_at: str          # ISO 8601 UTC string from Shopify
    line_items: list          # list[LineItem]

@dataclass
class TemplateConfig:
    template_path: str
    product_key: str          # e.g. "plate" — set by TemplateManager.resolve()
    text_box: dict            # {x, y, width, height} in pixels
    max_font_size: int
    min_font_size: int
    font_color: str           # hex e.g. "#5A3E2B"
    letter_spacing: int       # pixels between chars; 0 = no extra spacing
    dpi: int                  # print DPI, e.g. 300

@dataclass
class GenerationResult:
    order_id: str
    files_generated: int
    files_skipped: int
    files_failed: int
    # orders_processed is NOT here — aggregated by run_batch.py across all results
```

- [ ] **Step 4: Run test — expect pass**

```bash
pytest tests/test_models.py -v
```
Expected: 4 passed

- [ ] **Step 5: Commit**

```bash
git add models.py tests/test_models.py
git commit -m "feat: add shared dataclasses (models.py)"
```
