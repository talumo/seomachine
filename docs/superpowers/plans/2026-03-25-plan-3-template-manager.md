# Plan 3: Template Manager

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement task-by-task.

**Goal:** Load `template_config.json` and resolve a Shopify product title to a `TemplateConfig`.

**Spec:** `docs/specs/2026-03-25-personalized-print-file-generator-design.md`
**Depends on:** Plan 1 (models.py) complete

---

### Task 1: template_manager.py

**Files:**
- Create: `template_manager.py`
- Create: `tests/test_template_manager.py`

- [ ] **Step 1: Write the failing tests**

```python
# tests/test_template_manager.py
import json, pytest
from pathlib import Path
from template_manager import TemplateManager, ConfigError

MINIMAL_CONFIG = {
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
                "text_box": {"x": 210, "y": 380, "width": 480, "height": 80}
            },
            "mug": {
                "template": "templates/bunny-love/mug.png",
                "text_box": {"x": 150, "y": 260, "width": 380, "height": 65},
                "letter_spacing": 6
            }
        }
    },
    "product_mapping": {
        "plate": ["plate"],
        "mug": ["mug"]
    }
}

@pytest.fixture
def config_file(tmp_path):
    # Create dummy template image files so validation passes
    tmpl_dir = tmp_path / "templates" / "bunny-love"
    tmpl_dir.mkdir(parents=True)
    (tmpl_dir / "plate.png").write_bytes(b"")
    (tmpl_dir / "mug.png").write_bytes(b"")

    cfg = dict(MINIMAL_CONFIG)
    # Point templates to tmp_path
    cfg["themes"]["bunny-love"]["plate"]["template"] = str(tmpl_dir / "plate.png")
    cfg["themes"]["bunny-love"]["mug"]["template"] = str(tmpl_dir / "mug.png")

    path = tmp_path / "template_config.json"
    path.write_text(json.dumps(cfg))
    return path

def test_resolve_plate(config_file):
    tm = TemplateManager(config_file)
    result = tm.resolve("Bunny Love Plate Set")
    assert result is not None
    assert result.product_key == "plate"
    assert result.dpi == 300
    assert result.letter_spacing == 0  # default

def test_resolve_mug_with_override(config_file):
    tm = TemplateManager(config_file)
    result = tm.resolve("Bunny Love Mug")
    assert result is not None
    assert result.product_key == "mug"
    assert result.letter_spacing == 6  # product-level override

def test_resolve_unknown_theme_returns_none(config_file):
    tm = TemplateManager(config_file)
    result = tm.resolve("Dinosaur Plate")
    assert result is None

def test_resolve_unknown_product_returns_none(config_file):
    tm = TemplateManager(config_file)
    result = tm.resolve("Bunny Love Sticker Sheet")
    assert result is None

def test_resolve_case_insensitive(config_file):
    tm = TemplateManager(config_file)
    assert tm.resolve("BUNNY LOVE PLATE SET") is not None

def test_missing_template_file_raises(tmp_path):
    cfg = dict(MINIMAL_CONFIG)
    # Template file does NOT exist
    path = tmp_path / "template_config.json"
    path.write_text(json.dumps(cfg))
    with pytest.raises(ConfigError, match="template"):
        TemplateManager(path)
```

- [ ] **Step 2: Run tests — expect failure**

```bash
pytest tests/test_template_manager.py -v
```
Expected: `ModuleNotFoundError: No module named 'template_manager'`

- [ ] **Step 3: Create `template_manager.py`**

```python
import json, logging
from pathlib import Path
from models import TemplateConfig

logger = logging.getLogger(__name__)

class ConfigError(Exception):
    pass

class TemplateManager:
    def __init__(self, config_path):
        self._config_path = Path(config_path)
        data = json.loads(self._config_path.read_text(encoding="utf-8"))
        self._defaults = data.get("defaults", {})
        self._theme_mapping = data.get("theme_mapping", {})
        self._themes = data.get("themes", {})
        self._product_mapping = data.get("product_mapping", {})
        self._validate_templates()

    def _validate_templates(self):
        for theme, products in self._themes.items():
            for product_key, cfg in products.items():
                path = Path(cfg["template"])
                if not path.exists():
                    raise ConfigError(
                        f"Template file not found: {path} "
                        f"(theme={theme}, product={product_key})"
                    )

    def resolve(self, product_title: str):
        title_lower = product_title.lower().replace("-", " ")

        # Step 1: match theme
        theme_key = None
        for theme, keywords in self._theme_mapping.items():
            if any(kw.lower() in title_lower for kw in keywords):
                theme_key = theme
                break
        if theme_key is None:
            logger.warning("No theme matched for product title: %r", product_title)
            return None

        # Step 2: match product key
        product_key = None
        for key, keywords in self._product_mapping.items():
            if any(kw.lower() in title_lower for kw in keywords):
                product_key = key
                break
        if product_key is None:
            logger.warning("No product key matched for title: %r (theme=%s)",
                           product_title, theme_key)
            return None

        # Step 3: merge defaults + theme product config
        product_cfg = self._themes[theme_key].get(product_key, {})
        if not product_cfg:
            logger.warning("Product key %r not found in theme %r", product_key, theme_key)
            return None

        return TemplateConfig(
            template_path=product_cfg["template"],
            product_key=product_key,
            text_box=product_cfg["text_box"],
            max_font_size=product_cfg.get("max_font_size",
                          self._defaults.get("max_font_size", 72)),
            min_font_size=product_cfg.get("min_font_size",
                          self._defaults.get("min_font_size", 18)),
            font_color=product_cfg.get("font_color",
                       self._defaults.get("font_color", "#000000")),
            letter_spacing=product_cfg.get("letter_spacing",
                           self._defaults.get("letter_spacing", 0)),
            dpi=product_cfg.get("dpi", self._defaults.get("dpi", 300)),
        )
```

- [ ] **Step 4: Run tests — expect pass**

```bash
pytest tests/test_template_manager.py -v
```
Expected: 6 passed

- [ ] **Step 5: Commit**

```bash
git add template_manager.py tests/test_template_manager.py
git commit -m "feat: add template manager with theme+product resolution"
```
