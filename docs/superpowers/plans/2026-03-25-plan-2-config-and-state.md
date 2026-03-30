# Plan 2: Config + State Manager

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement task-by-task.

**Goal:** Load `.env` into a typed Config object; track processed order IDs in a JSON file.

**Spec:** `docs/specs/2026-03-25-personalized-print-file-generator-design.md`
**Depends on:** Plan 1 complete

---

### Task 1: config.py

**Files:**
- Create: `config.py`
- Create: `tests/test_config.py`

- [ ] **Step 1: Write the failing test**

```python
# tests/test_config.py
import os, pytest
from unittest.mock import patch

def test_config_loads_required_vars():
    env = {
        "SHOPIFY_ACCESS_TOKEN": "shpat_test",
        "SHOPIFY_STORE": "test.myshopify.com",
        "FONT_PATH": "fonts/test.ttf",
        "OUTPUT_DIR": "output",
    }
    with patch.dict(os.environ, env, clear=True):
        from importlib import reload
        import config
        reload(config)
        cfg = config.load_config()
    assert cfg.shopify_access_token == "shpat_test"
    assert cfg.shopify_store == "test.myshopify.com"
    assert cfg.shopify_api_version == "2024-04"  # default
    assert cfg.font_path == "fonts/test.ttf"
    assert cfg.output_dir == "output"

def test_config_raises_on_missing_token():
    with patch.dict(os.environ, {}, clear=True):
        from importlib import reload
        import config
        reload(config)
        with pytest.raises(SystemExit):
            config.load_config()
```

- [ ] **Step 2: Run test — expect failure**

```bash
pytest tests/test_config.py -v
```
Expected: `ModuleNotFoundError: No module named 'config'`

- [ ] **Step 3: Create `config.py`**

```python
import os, sys, logging
from dataclasses import dataclass
from pathlib import Path
from dotenv import load_dotenv

load_dotenv()

@dataclass
class Config:
    shopify_access_token: str
    shopify_store: str
    shopify_api_version: str
    font_path: str
    output_dir: str

def load_config() -> Config:
    required = {
        "SHOPIFY_ACCESS_TOKEN": os.getenv("SHOPIFY_ACCESS_TOKEN"),
        "SHOPIFY_STORE": os.getenv("SHOPIFY_STORE"),
        "FONT_PATH": os.getenv("FONT_PATH"),
    }
    missing = [k for k, v in required.items() if not v]
    if missing:
        print(f"ERROR: Missing required environment variables: {', '.join(missing)}", file=sys.stderr)
        print("Copy .env.example to .env and fill in the values.", file=sys.stderr)
        sys.exit(1)

    output_dir = os.getenv("OUTPUT_DIR", "output")
    Path(output_dir).mkdir(parents=True, exist_ok=True)

    _configure_logging(output_dir)

    return Config(
        shopify_access_token=required["SHOPIFY_ACCESS_TOKEN"],
        shopify_store=required["SHOPIFY_STORE"],
        shopify_api_version=os.getenv("SHOPIFY_API_VERSION", "2024-04"),
        font_path=required["FONT_PATH"],
        output_dir=output_dir,
    )

def _configure_logging(output_dir: str):
    log_path = Path(output_dir) / "run.log"
    logging.basicConfig(
        level=logging.DEBUG,
        format="%(asctime)s %(levelname)s %(name)s: %(message)s",
        handlers=[
            logging.FileHandler(log_path, encoding="utf-8"),
            logging.StreamHandler(),   # WARNING+ to stderr via root logger level below
        ],
    )
    # Silence DEBUG/INFO on stderr — only WARNING+ to console
    logging.getLogger().handlers[1].setLevel(logging.WARNING)
```

- [ ] **Step 4: Run test — expect pass**

```bash
pytest tests/test_config.py -v
```
Expected: 2 passed

- [ ] **Step 5: Commit**

```bash
git add config.py tests/test_config.py
git commit -m "feat: add config loader with env validation and logging"
```

---

### Task 2: state_manager.py

**Files:**
- Create: `state_manager.py`
- Create: `tests/test_state_manager.py`

- [ ] **Step 1: Write the failing test**

```python
# tests/test_state_manager.py
import json, pytest
from pathlib import Path
from state_manager import StateManager

def test_new_state_file_created(tmp_path):
    sm = StateManager(tmp_path / "state.json")
    assert not sm.is_processed("1001")

def test_mark_and_check(tmp_path):
    sm = StateManager(tmp_path / "state.json")
    sm.mark_processed("1001")
    assert sm.is_processed("1001")
    assert not sm.is_processed("1002")

def test_state_persists_across_instances(tmp_path):
    path = tmp_path / "state.json"
    sm1 = StateManager(path)
    sm1.mark_processed("1001")
    sm2 = StateManager(path)
    assert sm2.is_processed("1001")

def test_corrupt_state_file_recovers(tmp_path):
    path = tmp_path / "state.json"
    path.write_text("NOT VALID JSON")
    sm = StateManager(path)   # should not raise
    assert not sm.is_processed("1001")

def test_multiple_orders(tmp_path):
    sm = StateManager(tmp_path / "state.json")
    for oid in ["1", "2", "3"]:
        sm.mark_processed(oid)
    assert sm.is_processed("2")
    assert not sm.is_processed("4")
```

- [ ] **Step 2: Run test — expect failure**

```bash
pytest tests/test_state_manager.py -v
```
Expected: `ModuleNotFoundError: No module named 'state_manager'`

- [ ] **Step 3: Create `state_manager.py`**

```python
import json, logging, os, tempfile
from pathlib import Path

logger = logging.getLogger(__name__)

class StateManager:
    def __init__(self, state_file_path):
        self._path = Path(state_file_path)
        self._processed: set = set()
        self._load()

    def _load(self):
        if not self._path.exists():
            return
        try:
            data = json.loads(self._path.read_text(encoding="utf-8"))
            self._processed = set(data)
        except (json.JSONDecodeError, ValueError):
            logger.warning("State file unreadable (%s), starting fresh", self._path)

    def is_processed(self, order_id: str) -> bool:
        return str(order_id) in self._processed

    def mark_processed(self, order_id: str):
        self._processed.add(str(order_id))
        self._write()

    def _write(self):
        data = json.dumps(sorted(self._processed), indent=2)
        # Atomic write on POSIX; direct write on Windows
        try:
            fd, tmp = tempfile.mkstemp(dir=self._path.parent, suffix=".tmp")
            with os.fdopen(fd, "w", encoding="utf-8") as f:
                f.write(data)
            os.replace(tmp, self._path)
        except Exception:
            self._path.write_text(data, encoding="utf-8")
```

- [ ] **Step 4: Run test — expect pass**

```bash
pytest tests/test_state_manager.py -v
```
Expected: 5 passed

- [ ] **Step 5: Commit**

```bash
git add state_manager.py tests/test_state_manager.py
git commit -m "feat: add state manager for processed order tracking"
```
