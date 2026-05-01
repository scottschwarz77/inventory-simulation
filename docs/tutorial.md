# ShelfSync Tutorial

ShelfSync is a real-time inventory visibility and allocation API. It tracks stock across stores, warehouses, and distribution centres, manages order reservations, and fires webhook alerts when stock runs low.

---

## Setup

**1. Install dependencies**

```bash
cd inventory-src
pip install -r requirements.txt
```

**2. Configure environment**

```bash
cp .env.example .env
```

The defaults work out of the box. The server uses a local SQLite file (`shelfsync.db`). Edit `.env` only if you want to change the database or point webhooks at a real URL.

**3. Start the server**

```bash
uvicorn main:app --reload
```

The server starts at `http://localhost:8000`. The API docs are at `http://localhost:8000/docs`.

**4. (Optional) Load demo data**

```bash
python seed.py
```

This creates 5 locations, 5 SKUs, and a realistic set of stock-in, transfer, and sales events so you have data to explore immediately.

---

## Key Concepts

Before making any API calls, understand the three numbers that define every `(location, SKU)` position:

| Field | Meaning |
|---|---|
| `quantity_on_hand` | Physical units present at the location |
| `quantity_reserved` | Units held for pending orders, not yet shipped |
| `quantity_available` | `on_hand − reserved` — what you can promise today |

**Events are the source of truth.** You never directly set quantities. Instead you submit *events* (`stock_in`, `stock_out`, `adjustment`, `transfer_in`, `transfer_out`) and the system derives current positions from the event log.

---

## Step-by-Step Walkthrough

The following curl examples show a complete lifecycle: register a location and SKU, receive stock, sell some, reserve units for a pending order, then release the reservation.

Set a base URL variable to keep things concise:

```bash
BASE=http://localhost:8000
```

### 1. Register a Location

Locations are stores, warehouses, or distribution centres.

```bash
curl -s -X POST $BASE/v1/locations \
  -H "Content-Type: application/json" \
  -d '{
    "name": "Store #1 - Austin Downtown",
    "type": "store",
    "region": "us-south"
  }' | jq
```

Valid `type` values: `store`, `warehouse`, `distribution_center`.

**Response:**
```json
{
  "id": 1,
  "name": "Store #1 - Austin Downtown",
  "type": "store",
  "region": "us-south",
  "address": null,
  "created_at": "2026-04-29T10:00:00Z"
}
```

Save the `id` — you need it for inventory events and reservations.

---

### 2. Register a SKU

```bash
curl -s -X POST $BASE/v1/skus \
  -H "Content-Type: application/json" \
  -d '{
    "sku_code": "WIDGET-RED-LG",
    "name": "Red Widget (Large)",
    "low_stock_threshold": 10
  }' | jq
```

`low_stock_threshold` controls when webhook alerts fire. The default is 10 if you omit it.

---

### 3. Receive Stock (`stock_in`)

Submit an inventory event to bring stock into the location.

```bash
curl -s -X POST $BASE/v1/inventory/events \
  -H "Content-Type: application/json" \
  -d '{
    "sku_code": "WIDGET-RED-LG",
    "location_id": 1,
    "event_type": "stock_in",
    "delta": 100,
    "reference": "PO-2024-001",
    "idempotency_key": "po-2024-001-line-1"
  }' | jq
```

- `delta` must be positive for `stock_in`.
- `reference` is a free-text field — use a PO number, order ID, or anything meaningful.
- `idempotency_key` prevents double-processing if the request is retried. Re-submitting the same key returns `HTTP 409` instead of applying the event again.

---

### 4. Check Inventory Position

```bash
curl -s "$BASE/v1/inventory/positions?sku_code=WIDGET-RED-LG" | jq
```

**Response:**
```json
{
  "items": [
    {
      "location_id": 1,
      "location_name": "Store #1 - Austin Downtown",
      "sku_code": "WIDGET-RED-LG",
      "quantity_on_hand": 100,
      "quantity_reserved": 0,
      "quantity_available": 100,
      "updated_at": "2026-04-29T10:05:00Z"
    }
  ],
  "total": 1
}
```

You can also filter by `location_id`, or omit all filters to see every position across all locations and SKUs.

---

### 5. Sell Stock (`stock_out`)

```bash
curl -s -X POST $BASE/v1/inventory/events \
  -H "Content-Type: application/json" \
  -d '{
    "sku_code": "WIDGET-RED-LG",
    "location_id": 1,
    "event_type": "stock_out",
    "delta": -25,
    "reference": "SALES-BATCH-042"
  }' | jq
```

`delta` must be **negative** for `stock_out`. After this event, `quantity_on_hand` drops to 75.

---

### 6. Transfer Stock Between Locations

To move 20 units from location 1 to location 2, submit two events:

```bash
# Step 1 — debit the source
curl -s -X POST $BASE/v1/inventory/events \
  -H "Content-Type: application/json" \
  -d '{"sku_code":"WIDGET-RED-LG","location_id":1,"event_type":"transfer_out","delta":-20,"reference":"TRF-007"}' | jq

# Step 2 — credit the destination
curl -s -X POST $BASE/v1/inventory/events \
  -H "Content-Type: application/json" \
  -d '{"sku_code":"WIDGET-RED-LG","location_id":2,"event_type":"transfer_in","delta":20,"reference":"TRF-007"}' | jq
```

Use the same `reference` on both events to link them.

---

### 7. Reserve Units for a Pending Order

A reservation immediately reduces `quantity_available` without touching `quantity_on_hand`.

```bash
curl -s -X POST $BASE/v1/reservations \
  -H "Content-Type: application/json" \
  -d '{
    "sku_code": "WIDGET-RED-LG",
    "location_id": 1,
    "quantity": 5,
    "reference": "ORD-9001"
  }' | jq
```

**Response:**
```json
{
  "reservation_id": "rsv_a3f2c1b09d4e",
  "sku_code": "WIDGET-RED-LG",
  "location_id": 1,
  "quantity": 5,
  "reference": "ORD-9001",
  "quantity_available_after": 50
}
```

**Save the `reservation_id`** — you need it to release the hold. ShelfSync does not store reservations in the database; you are responsible for tracking this ID.

---

### 8. Release a Reservation

When the order ships (or is cancelled), release the reservation to return units to ATP:

```bash
curl -s -X DELETE \
  "$BASE/v1/reservations/rsv_a3f2c1b09d4e?sku_code=WIDGET-RED-LG&location_id=1&quantity=5" | jq
```

All four parameters (`reservation_id`, `sku_code`, `location_id`, `quantity`) are required.

---

### 9. View Event History

```bash
# Last 50 events (default)
curl -s "$BASE/v1/inventory/history" | jq

# Filter to one SKU
curl -s "$BASE/v1/inventory/history?sku_code=WIDGET-RED-LG" | jq

# Filter to one location, most recent 10
curl -s "$BASE/v1/inventory/history?location_id=1&limit=10" | jq

# Filter by event type
curl -s "$BASE/v1/inventory/history?event_type=stock_out" | jq
```

---

## Low-Stock Webhooks

Register a URL to receive a POST whenever `quantity_available` drops to or below a SKU's `low_stock_threshold`:

```bash
curl -s -X POST $BASE/v1/webhooks \
  -H "Content-Type: application/json" \
  -d '{
    "url": "https://your-system.example.com/hooks/shelfsync",
    "secret": "my-signing-secret",
    "low_stock_enabled": true
  }' | jq
```

ShelfSync will POST a payload like this to the registered URL:

```json
{
  "event": "low_stock",
  "sku_code": "WIDGET-RED-LG",
  "location_id": 1,
  "location_name": "Store #1 - Austin Downtown",
  "quantity_available": 4,
  "threshold": 10,
  "triggered_at": "2026-04-29T14:32:00Z"
}
```

If you provided a `secret`, the request includes an HMAC-SHA256 signature header:
```
X-ShelfSync-Signature: sha256=<hmac-hex>
```

To remove a webhook:
```bash
curl -s -X DELETE $BASE/v1/webhooks/1
```

---

## Reference

### All Event Types

| `event_type` | `delta` sign | Typical use |
|---|---|---|
| `stock_in` | positive | Receiving a purchase order |
| `stock_out` | negative | Fulfilling a customer order |
| `adjustment` | either | Cycle count correction |
| `transfer_in` | positive | Receiving stock from another location |
| `transfer_out` | negative | Shipping stock to another location |

### Environment Variables

| Variable | Default | Description |
|---|---|---|
| `SHELFSYNC_DATABASE_URL` | SQLite file | Use `postgresql+asyncpg://...` for Postgres |
| `SHELFSYNC_DEBUG` | `false` | Enables verbose SQL logging |
| `SHELFSYNC_WEBHOOK_URL` | _(none)_ | Default webhook target |
| `SHELFSYNC_WEBHOOK_SECRET` | _(none)_ | HMAC signing secret |
| `SHELFSYNC_WEBHOOK_TIMEOUT_SECONDS` | `5` | Webhook POST timeout |
| `SHELFSYNC_DEFAULT_LOW_STOCK_THRESHOLD` | `10` | Applied to new SKUs |

### Useful URLs

| URL | Description |
|---|---|
| `http://localhost:8000/docs` | Swagger interactive API explorer |
| `http://localhost:8000/redoc` | ReDoc reference |
| `http://localhost:8000/health` | Liveness probe |
| `http://localhost:8000/ui` | Static UI |

---

## Known Limitations

These are intentional constraints of the prototype:

- **No authentication** — there is no API key or OAuth layer.
- **Reservations are not persisted** — the database has no reservations table; clients must store the `reservation_id` returned at creation time.
- **No webhook retry** — failed deliveries are logged but not retried automatically.
- **Single database** — no multi-region federation.
