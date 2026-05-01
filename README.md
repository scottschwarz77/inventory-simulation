# ShelfSync — Inventory Simulation

Real-time inventory visibility and allocation engine. Tracks stock positions across stores, warehouses, and distribution centres, manages reservations, and fires low-stock webhook notifications.

> **Purpose:** This is a documentation simulation prototype. It is intentionally minimal — just enough to be real and runnable so that developer and operations documentation can be written against actual behaviour.

---

## Quick Start

### 1. Install dependencies

```bash
pip install -r inventory-src/requirements.txt
```

### 2. Configure environment

```bash
cp inventory-src/.env.example inventory-src/.env
# Edit .env if you want to change the database URL or set a webhook target
```

### 3. Start the server

```bash
cd inventory-src
uvicorn main:app --reload
```

### 4. Explore the API

- **Interactive docs (Swagger):** http://localhost:8000/docs
- **ReDoc:** http://localhost:8000/redoc
- **Health check:** http://localhost:8000/health

### 5. Load demo data (optional)

```bash
cd inventory-src
python seed.py
```

---

## API Overview

### Locations
| Method | Path | Description |
|--------|------|-------------|
| `POST` | `/v1/locations` | Register a store, warehouse, or DC |
| `GET` | `/v1/locations` | List all locations |
| `GET` | `/v1/locations/{id}` | Get a single location |
| `GET` | `/v1/locations/{id}/inventory` | Full inventory at one location |

### SKUs
| Method | Path | Description |
|--------|------|-------------|
| `POST` | `/v1/skus` | Register a new SKU |
| `GET` | `/v1/skus` | List all SKUs |
| `GET` | `/v1/skus/{sku_code}` | Get a single SKU |
| `PATCH` | `/v1/skus/{sku_code}` | Update SKU metadata or low-stock threshold |

### Inventory
| Method | Path | Description |
|--------|------|-------------|
| `POST` | `/v1/inventory/events` | Submit a stock-in, stock-out, adjustment, or transfer event |
| `GET` | `/v1/inventory/positions` | Current positions across all locations/SKUs |
| `GET` | `/v1/inventory/positions/{sku_code}` | Positions for one SKU across all locations |
| `GET` | `/v1/inventory/history` | Event history (filterable) |

### Reservations
| Method | Path | Description |
|--------|------|-------------|
| `POST` | `/v1/reservations` | Reserve units (reduces available-to-promise stock) |
| `DELETE` | `/v1/reservations/{id}` | Release a reservation |

### Webhooks
| Method | Path | Description |
|--------|------|-------------|
| `POST` | `/v1/webhooks` | Register a low-stock notification URL |
| `GET` | `/v1/webhooks` | List registered webhooks |
| `DELETE` | `/v1/webhooks/{id}` | Remove a webhook |

---

## Key Concepts

### Inventory Position
Each `(location, SKU)` pair maintains three quantities:

| Field | Meaning |
|-------|---------|
| `quantity_on_hand` | Physical units present at the location |
| `quantity_reserved` | Units held for pending orders (not yet shipped) |
| `quantity_available` | `on_hand − reserved` — available to promise (ATP) |

### Inventory Events
Events are the source of truth. The position table is a derived view.

| Event Type | Delta | Typical use |
|------------|-------|-------------|
| `stock_in` | positive | Receiving a purchase order |
| `stock_out` | negative | Fulfilling a customer order |
| `adjustment` | positive or negative | Cycle count correction |
| `transfer_in` | positive | Receiving stock from another location |
| `transfer_out` | negative | Shipping stock to another location |

### Idempotency
Supply an `idempotency_key` on `POST /v1/inventory/events` to prevent duplicate processing. Re-submitting the same key returns `HTTP 409` without applying the event again.

### Low-Stock Webhooks
When `quantity_available` drops to or below a SKU's `low_stock_threshold` after any event or reservation, ShelfSync POSTs to all registered webhook URLs:

```json
{
  "event": "low_stock",
  "sku_code": "WIDGET-RED-LG",
  "location_id": 1,
  "location_name": "Store #1 - Austin Downtown",
  "quantity_available": 4,
  "threshold": 10,
  "triggered_at": "2024-05-01T14:32:00Z"
}
```

If a `secret` is configured, the request includes:
```
X-ShelfSync-Signature: sha256=<hmac-hex>
```

---

## Configuration

All settings are read from environment variables (prefixed `SHELFSYNC_`) or from a `.env` file.

| Variable | Default | Description |
|----------|---------|-------------|
| `SHELFSYNC_DEBUG` | `false` | Enable verbose SQL logging |
| `SHELFSYNC_DATABASE_URL` | `sqlite+aiosqlite:///./shelfsync.db` | SQLAlchemy async database URL |
| `SHELFSYNC_WEBHOOK_URL` | _(none)_ | Default webhook POST target |
| `SHELFSYNC_WEBHOOK_SECRET` | _(none)_ | HMAC signing secret |
| `SHELFSYNC_WEBHOOK_TIMEOUT_SECONDS` | `5` | Webhook POST timeout |
| `SHELFSYNC_DEFAULT_LOW_STOCK_THRESHOLD` | `10` | Threshold applied to new SKUs |

---

## Project Structure

```
inventory-src/
├── main.py               # FastAPI app, lifespan, middleware, health endpoint
├── config.py             # Pydantic settings (reads from env / .env)
├── database.py           # Async engine, session factory, table init
├── models.py             # SQLAlchemy ORM models
├── schemas.py            # Pydantic request/response schemas
├── seed.py               # Demo data loader
├── routers/
│   ├── locations.py      # Location CRUD + inventory view
│   ├── skus.py           # SKU CRUD
│   ├── inventory.py      # Events + position queries
│   ├── reservations.py   # Reserve / release
│   └── webhooks.py       # Webhook registration
└── services/
    ├── inventory.py      # Core inventory logic (event application, position query)
    └── webhooks.py       # HTTP delivery of low-stock payloads
```

---

## Tech Stack

- **[FastAPI](https://fastapi.tiangolo.com/)** — async REST framework
- **[SQLAlchemy 2](https://docs.sqlalchemy.org/)** — async ORM (SQLite by default, PostgreSQL-compatible)
- **[Pydantic v2](https://docs.pydantic.dev/)** — request/response validation
- **[uvicorn](https://www.uvicorn.org/)** — ASGI server
- **[httpx](https://www.python-httpx.org/)** — async HTTP client for webhook delivery
