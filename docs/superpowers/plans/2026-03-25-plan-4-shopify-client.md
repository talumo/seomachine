# Plan 4: Shopify Client

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement task-by-task.

**Goal:** Fetch paid/unfulfilled orders from Shopify Admin API with pagination and retry.

**Spec:** `docs/specs/2026-03-25-personalized-print-file-generator-design.md`
**Depends on:** Plan 1 (models.py) complete

---

### Task 1: shopify_client.py

**Files:**
- Create: `shopify_client.py`
- Create: `tests/test_shopify_client.py`

- [ ] **Step 1: Write the failing tests**

```python
# tests/test_shopify_client.py
import pytest
from unittest.mock import patch, MagicMock
from models import Order, LineItem
from shopify_client import ShopifyClient, ShopifyAuthError

MOCK_ORDER = {
    "id": 1001,
    "order_number": "#1001",
    "created_at": "2026-03-25T10:00:00Z",
    "line_items": [
        {
            "title": "Bunny Love Plate Set",
            "properties": [{"name": "Personalization", "value": "Emma"}]
        },
        {
            "title": "Bunny Love Mug",
            "properties": [{"name": "Personalization", "value": "Emma"}]
        },
        {
            "title": "Plain Sticker Sheet",  # no personalization
            "properties": []
        }
    ]
}

def make_client():
    return ShopifyClient(
        access_token="shpat_test",
        store="test.myshopify.com",
        api_version="2024-04"
    )

def mock_response(json_data, status=200, link_header=None):
    resp = MagicMock()
    resp.status_code = status
    resp.json.return_value = json_data
    resp.headers = {"Link": link_header} if link_header else {}
    return resp

def test_fetch_returns_orders_with_personalization():
    client = make_client()
    with patch("shopify_client.requests.get",
               return_value=mock_response({"orders": [MOCK_ORDER]})):
        orders = client.fetch_pending_orders()
    assert len(orders) == 1
    order = orders[0]
    assert order.order_id == "1001"
    assert len(order.line_items) == 2  # sticker sheet excluded (no personalization)
    assert order.line_items[0].name == "Emma"

def test_fetch_since_date():
    client = make_client()
    with patch("shopify_client.requests.get",
               return_value=mock_response({"orders": []})) as mock_get:
        client.fetch_pending_orders(since_date="2026-03-01")
    call_url = mock_get.call_args[0][0]
    assert "created_at_min=2026-03-01T00%3A00%3A00Z" in call_url or \
           "created_at_min" in call_url

def test_auth_error_raises():
    client = make_client()
    with patch("shopify_client.requests.get",
               return_value=mock_response({}, status=401)):
        with pytest.raises(ShopifyAuthError):
            client.fetch_pending_orders()

def test_fetch_single_order():
    client = make_client()
    with patch("shopify_client.requests.get",
               return_value=mock_response({"order": MOCK_ORDER})):
        orders = client.fetch_order_by_id("1001")
    assert len(orders) == 1
    assert orders[0].order_id == "1001"

def test_personalization_key_case_insensitive():
    order_data = {
        **MOCK_ORDER,
        "line_items": [{
            "title": "Bunny Love Plate Set",
            "properties": [{"name": "personalisation", "value": "Sofía"}]
        }]
    }
    client = make_client()
    with patch("shopify_client.requests.get",
               return_value=mock_response({"orders": [order_data]})):
        orders = client.fetch_pending_orders()
    assert orders[0].line_items[0].name == "Sofía"

def test_rate_limit_retries(monkeypatch):
    client = make_client()
    responses = [
        mock_response({}, status=429),
        mock_response({}, status=429),
        mock_response({"orders": [MOCK_ORDER]}, status=200),
    ]
    with patch("shopify_client.requests.get", side_effect=responses), \
         patch("shopify_client.time.sleep"):  # don't actually sleep in tests
        orders = client.fetch_pending_orders()
    assert len(orders) == 1
```

- [ ] **Step 2: Run tests — expect failure**

```bash
pytest tests/test_shopify_client.py -v
```
Expected: `ModuleNotFoundError: No module named 'shopify_client'`

- [ ] **Step 3: Create `shopify_client.py`**

```python
import logging, time
import requests
from urllib.parse import urlencode, quote
from models import Order, LineItem

logger = logging.getLogger(__name__)

PERSONALIZATION_KEYS = {"personalization", "personalisation", "name"}

class ShopifyAuthError(Exception):
    pass

class ShopifyClient:
    def __init__(self, access_token: str, store: str, api_version: str):
        self._token = access_token
        self._store = store
        self._version = api_version
        self._headers = {
            "X-Shopify-Access-Token": access_token,
            "Content-Type": "application/json",
        }

    def _base_url(self):
        return f"https://{self._store}/admin/api/{self._version}"

    def _get(self, url, retries=3):
        delay = 2
        for attempt in range(retries + 1):
            resp = requests.get(url, headers=self._headers)
            if resp.status_code == 200:
                return resp
            if resp.status_code in (401, 403):
                raise ShopifyAuthError(
                    f"Shopify auth failed ({resp.status_code}). "
                    "Check SHOPIFY_ACCESS_TOKEN and app permissions."
                )
            if resp.status_code == 429 and attempt < retries:
                logger.warning("Rate limited, retrying in %ds...", delay)
                time.sleep(delay)
                delay *= 2
                continue
            resp.raise_for_status()
        resp.raise_for_status()  # final attempt failed

    def fetch_pending_orders(self, since_date=None) -> list:
        params = {
            "financial_status": "paid",
            "fulfillment_status": "unfulfilled",
            "limit": 250,
        }
        if since_date:
            params["created_at_min"] = f"{since_date}T00:00:00Z"

        url = f"{self._base_url()}/orders.json?{urlencode(params)}"
        all_orders = []

        while url:
            resp = self._get(url)
            data = resp.json()
            for raw in data.get("orders", []):
                order = self._parse_order(raw)
                if order.line_items:
                    all_orders.append(order)
            # Pagination
            link = resp.headers.get("Link", "")
            url = self._next_url(link)

        return all_orders

    def fetch_order_by_id(self, order_id: str) -> list:
        url = f"{self._base_url()}/orders/{order_id}.json"
        resp = self._get(url)
        raw = resp.json().get("order", {})
        order = self._parse_order(raw)
        return [order] if order.line_items else []

    def _parse_order(self, raw: dict) -> Order:
        items = []
        for item in raw.get("line_items", []):
            name = self._extract_personalization(item.get("properties", []))
            if name:
                items.append(LineItem(title=item["title"], name=name))
            else:
                logger.warning(
                    "Order %s: line item %r has no personalization property",
                    raw.get("id"), item.get("title")
                )
        return Order(
            order_id=str(raw["id"]),
            order_number=str(raw.get("order_number", "")),
            created_at=raw.get("created_at", ""),
            line_items=items,
        )

    def _extract_personalization(self, properties: list):
        for prop in properties:
            if prop.get("name", "").lower() in PERSONALIZATION_KEYS:
                val = prop.get("value", "").strip()
                return val if val else None
        return None

    @staticmethod
    def _next_url(link_header: str):
        for part in link_header.split(","):
            if 'rel="next"' in part:
                return part.split(";")[0].strip().strip("<>")
        return None
```

- [ ] **Step 4: Run tests — expect pass**

```bash
pytest tests/test_shopify_client.py -v
```
Expected: 6 passed

- [ ] **Step 5: Commit**

```bash
git add shopify_client.py tests/test_shopify_client.py
git commit -m "feat: add Shopify client with pagination, retry, and order parsing"
```
