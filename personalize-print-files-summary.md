# Personalize Print Files — Project Summary

**Date:** March 31, 2026

---

## What We Built

A two-mode tool for generating print-ready PDFs from personalized Shopify tableware orders:

1. **Embedded web app** — lives inside Shopify Admin, lets you select orders and download ZIPs from the browser
2. **Batch CLI** (`run_batch.py`) — runs from the terminal or a cron job, processes all new orders automatically

Both modes use the same pipeline: fetch order → match product to template → composite name onto image → export PDF at 300 DPI → zip per order.

---

## Repository

**GitHub:** `talumo/personalize-print-files` (branch: `main`)

---

## What Was Implemented

### Core Modules
| File | Purpose |
|---|---|
| `models.py` | `Order`, `LineItem`, `GenerationResult` dataclasses |
| `config.py` | Env-based config loader for CLI mode |
| `state_manager.py` | Tracks processed orders (JSON file, CLI mode) |
| `template_manager.py` | JSON-driven theme/product matching (CLI mode) |
| `shopify_client.py` | Shopify Admin API — fetch orders by status or ID |
| `text_renderer.py` | Composites name onto template image with auto font-sizing |
| `file_generator.py` | PDF export (300 DPI via Pillow + ReportLab) + ZIP packaging |
| `run_batch.py` | CLI entry point (`--dry-run`, `--order`, `--since` flags) |

### Flask Web App
| File | Purpose |
|---|---|
| `app.py` | Flask app factory, registers all blueprints |
| `db.py` | SQLite CRUD — settings, templates, jobs, processed orders |
| `job_queue.py` | Background thread queue for async PDF generation |
| `web_template_manager.py` | Template matching from DB (web app mode) |
| `routes/auth.py` | Shopify OAuth 2.0 — install + callback with HMAC verification |
| `routes/orders.py` | Order listing + job creation |
| `routes/jobs.py` | Job status polling API |
| `routes/downloads.py` | Job history + ZIP download |
| `routes/templates_routes.py` | Template CRUD with image upload |
| `routes/settings_routes.py` | Font path, output dir, API version settings |

### UI Templates
| File | Purpose |
|---|---|
| `html_templates/base.html` | Bootstrap 5 base with Orders / Downloads / Templates / Settings nav |
| `html_templates/orders.html` | Order list with checkboxes, generate button, JS polling |
| `html_templates/downloads.html` | Job history and ZIP download links |
| `html_templates/template_list.html` | Template grid grouped by theme with edit/delete |
| `html_templates/template_editor.html` | Create/edit form — image upload, text box coords, typography |
| `html_templates/settings.html` | Font path, output dir, API version form |
| `static/orders.js` | Frontend polling for job status |

### Supporting Files
| File | Purpose |
|---|---|
| `Procfile` | `waitress-serve --port=$PORT --call app:create_app` — Railway/Heroku startup |
| `requirements.txt` | Pillow, ReportLab, Flask, waitress, requests, python-dotenv, PyJWT, pytest |
| `.env.example` | Template for all required environment variables |
| `template_config.json` | Scaffold with `bunny-love` example theme |
| `README.md` | Full setup guide for both CLI and web app modes |

### Tests
**89 tests, all passing.**

| Test file | Coverage |
|---|---|
| `test_models.py` | Dataclass construction and defaults |
| `test_config.py` | Env var loading and validation |
| `test_state_manager.py` | Processed-order tracking |
| `test_template_manager.py` | Theme/product keyword matching |
| `test_shopify_client.py` | API calls (mocked) |
| `test_text_renderer.py` | Font sizing and compositing |
| `test_file_generator.py` | PDF + ZIP generation |
| `test_run_batch.py` | CLI flags, dry-run, skip logic, exit codes |
| `test_auth.py` | HMAC verification, OAuth flow, CSRF protection |
| `test_db.py` | SQLite CRUD helpers |
| `test_job_queue.py` | Enqueue, worker, status transitions |
| `test_downloads.py` | Download routes and ZIP serving |
| `test_settings_routes.py` | Settings page render, save, validation |

---

## Deployment

**Platform:** Railway
**URL:** `https://web-production-7e79.up.railway.app`

### Environment Variables (set in Railway)

| Variable | Notes |
|---|---|
| `SHOPIFY_API_KEY` | Client ID from Shopify Partner Dashboard |
| `SHOPIFY_API_SECRET` | Client Secret from Shopify Partner Dashboard |
| `SECRET_KEY` | Random 32-byte hex — generated once, never changes |
| `APP_URL` | `https://web-production-7e79.up.railway.app` |
| `FONT_PATH` | `fonts/your-font.ttf` (file committed to repo) |
| `DB_PATH` | `/app/data/app.db` (on Railway persistent volume) |
| `OUTPUT_DIR` | `/app/data/output` (on Railway persistent volume) |
| `SHOPIFY_API_VERSION` | `2024-04` (update quarterly) |

> `SHOPIFY_ACCESS_TOKEN` is **not** set manually — it is obtained automatically via OAuth when you install the app.

### Railway Volume
Mount path: `/app/data`
Stores: `app.db` (database) and `output/` (generated PDFs and ZIPs).
Required so data survives redeploys.

### Shopify Partner Dashboard Configuration
- **App URL:** `https://web-production-7e79.up.railway.app/`
- **Allowed redirect URLs:** `https://web-production-7e79.up.railway.app/auth/callback`
- **Admin API scopes:** `read_orders`

---

## How to Install the App on Your Store

Visit this URL in your browser while logged into Shopify Admin:

```
https://web-production-7e79.up.railway.app/install?shop=khd-kids.myshopify.com
```

Shopify will ask you to authorize. After approval you land on the Orders page inside Shopify Admin.

---

## How to Use

### Web App
1. Open the app from Shopify Admin
2. **Orders** — shows all paid, unfulfilled orders with a Personalization property
3. Select one or more orders → click **Generate Print Files**
4. **Downloads** — monitor job progress, download ZIP when complete

### CLI (for batch/cron use)
```bash
python run_batch.py              # process all new orders
python run_batch.py --dry-run   # preview without generating
python run_batch.py --order 123 # reprocess a specific order
python run_batch.py --since 2026-03-01
```

---

## How to Add Templates

1. Export each product PSD as a full-canvas PNG at 300 DPI (hide the name text layer first)
2. In the app, go to **Templates → Add Template**
3. Fill in:
   - **Theme key** — e.g. `bunny-love`
   - **Product key** — e.g. `plate`, `bowl`, `mug`, `placemat`, `spoon_fork`
   - Upload the PNG
   - Enter the text box coordinates (x, y, width, height in pixels from top-left)
   - Set font size range, color, letter spacing
4. Add one row per product type per theme

---

## How to Update the App

```bash
# Make changes locally
pytest tests/ -q        # confirm 89+ tests pass
git add <files>
git commit -m "description"
git push                # Railway redeploys automatically
```

To add a new Shopify API permission: update `SHOPIFY_SCOPES` in `routes/auth.py`, update scopes in Partner Dashboard, then reinstall the app so the merchant re-authorizes.

---

## Known Limitations

- **In-memory job queue** — jobs in progress are lost if the server restarts (marked failed on startup)
- **Single name per order** — all products share one name; multi-name bundles not supported
- **No OpenType kerning** — compensate with `letter_spacing` in template settings
- **SQLite** — fine for single-instance Railway deployment; not suitable for multiple concurrent servers
- **`--since` is UTC** — orders near midnight in non-UTC timezones may behave unexpectedly
