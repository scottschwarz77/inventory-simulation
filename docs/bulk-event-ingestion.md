# Bulk Inventory Event Ingestion

> Submit many inventory events in a single HTTP request. Designed for high-volume batch imports, nightly reconciliation jobs, and ERP/WMS sync pipelines that would otherwise generate thousands of individual API calls.

- **Endpoint:** `POST /v1/inventory/events/bulk`
- **Status:** Generally available
- **Tracking issue:** [KAN-14](https://scottschwarz77.atlassian.net/browse/KAN-14)
- **Resolved:** 2026-04-30

---

## When to use this endpoint

Use the bulk endpoint instead of looping over `POST /v1/inventory/events` when any of the following are true:

- You are ingesting data from a file, ERP export, or scheduled batch job.
- You expect to submit more than ~10 events back-to-back to the same caller.
- You need all-or-nothing visibility into which rows in a batch failed validation.
- HTTP overhead (TLS handshake, request framing) is materially impacting your throughput.

For interactive, single-event flows (a worker scanning a barcode, a checkout decrement) keep using the singular `POST /v1/inventory/events` endpoint — it has lower latency per event.

## Request

### Headers

| Header | Required | Description |
| --- | --- | --- |
| `Content-Type` | yes | Must be `application/json`. |
| `Idempotency-Key` | optional | Top-level idempotency key for the request envelope. Per-event idempotency keys (see below) are also honored. |

### Body

The request body is a JSON object containing an `events` array. Each element of the array is the same shape accepted by the singular event endpoint.

```json
{
  "events": [
    {
      "event_type": "stock_in",
      "location_id": "LOC-CHI-01",
      "sku": "WIDGET-RED-01",
      "quantity_delta": 50,
      "reference_id": "PO-2026-0418",
      "idempotency_key": "po-2026-0418-line-1"
    },
    {
      "event_type": "stock_out",
      "location_id": "LOC-CHI-01",
      "sku": "WIDGET-BLUE-02",
      "quantity_delta": -3,
      "reference_id": "ORD-99812",
      "idempotency_key": "ord-99812-line-1"
    },
    {
      "event_type": "adjustment",
      "location_id": "LOC-NYC-02",
      "sku": "WIDGET-RED-01",
      "quantity_delta": -2,
      "reason": "cycle_count_correction"
    }
  ]
}
```

### Limits

| Limit | Value |
| --- | --- |
| Maximum events per request | **500** |
| Allowed `event_type` values | `stock_in`, `stock_out`, `adjustment`, `transfer_in`, `transfer_out` |
| Sign rule for `stock_in` / `transfer_in` | `quantity_delta` must be positive |
| Sign rule for `stock_out` / `transfer_out` | `quantity_delta` must be negative |
| Sign rule for `adjustment` | either sign permitted |

Requests with more than 500 events are rejected with `400 Bad Request` before any event is processed.

## Response

The endpoint returns **`207 Multi-Status`** with one result entry per submitted event, in the same order the events were submitted.

```json
{
  "results": [
    {
      "index": 0,
      "status": "created",
      "event_id": "evt_01HV9ZKQ4P7N3M2X",
      "idempotency_key": "po-2026-0418-line-1"
    },
    {
      "index": 1,
      "status": "created",
      "event_id": "evt_01HV9ZKQ4P7N3M2Y",
      "idempotency_key": "ord-99812-line-1"
    },
    {
      "index": 2,
      "status": "error",
      "error": {
        "code": "sku_not_found",
        "message": "SKU 'WIDGET-RED-01' does not exist at location 'LOC-NYC-02'."
      }
    }
  ],
  "summary": {
    "submitted": 3,
    "created": 2,
    "errors": 1
  }
}
```

### Result fields

- `index` — zero-based position of the event in your submitted `events` array.
- `status` — `created` for accepted events, `duplicate` if the per-event idempotency key matched a prior event, or `error` for validation/processing failures.
- `event_id` — present on `created` and `duplicate` results.
- `idempotency_key` — echoed back when supplied.
- `error` — present only on `error` results; contains a stable machine-readable `code` and a human-readable `message`.

## Transactional behavior

All **valid** events in the batch are written inside a single database transaction. If any event fails validation:

- The invalid event is reported in `results` with `status: "error"`.
- The valid events in the same batch are still committed.
- The HTTP status remains `207`. Inspect `summary.errors` to detect partial failures.

If the underlying transaction itself fails (database error, deadlock, etc.), the API responds with `500 Internal Server Error` and **no** events from the batch are persisted. Safe to retry with the same idempotency keys.

## Idempotency

Per-event `idempotency_key` values are deduplicated independently — replaying a batch will return `status: "duplicate"` for any event whose key was already accepted, while still creating any events in that same batch whose keys are new.

This makes it safe to:

- Retry a batch that timed out mid-flight.
- Re-run an ERP nightly export against the same window.
- Resume a paginated import from any offset.

## Effects on derived state

Each successfully created event updates the relevant inventory position exactly as it would through the singular endpoint:

- `quantity_on_hand` is adjusted by the event's `quantity_delta`.
- `quantity_available` is recomputed as `on_hand − reserved`.
- If the recomputed `quantity_available` for a SKU drops to or below its `low_stock_threshold`, a webhook is dispatched (one webhook per SKU per batch, even if multiple events in the same batch crossed the threshold).

## Common errors

| HTTP | `error.code` | Cause |
| --- | --- | --- |
| 400 | `events_required` | The `events` array is missing or empty. |
| 400 | `batch_too_large` | More than 500 events submitted. |
| 400 | `invalid_event_type` | An event has an unrecognized `event_type`. |
| 400 | `sign_mismatch` | A `stock_in` had a negative delta, or similar. |
| 404 | `sku_not_found` | The `(location_id, sku)` pair has no inventory position. |
| 409 | `idempotency_conflict` | An idempotency key was reused with a different payload. |

## Example: cURL

```bash
curl -X POST https://api.shelfsync.local/v1/inventory/events/bulk \
  -H "Content-Type: application/json" \
  -d '{
    "events": [
      {"event_type":"stock_in","location_id":"LOC-CHI-01","sku":"WIDGET-RED-01","quantity_delta":50,"reference_id":"PO-2026-0418"},
      {"event_type":"stock_out","location_id":"LOC-CHI-01","sku":"WIDGET-BLUE-02","quantity_delta":-3,"reference_id":"ORD-99812"}
    ]
  }'
```

## Example: Python

```python
import requests

events = [
    {"event_type": "stock_in",  "location_id": "LOC-CHI-01", "sku": "WIDGET-RED-01",  "quantity_delta": 50},
    {"event_type": "stock_out", "location_id": "LOC-CHI-01", "sku": "WIDGET-BLUE-02", "quantity_delta": -3},
]

resp = requests.post(
    "https://api.shelfsync.local/v1/inventory/events/bulk",
    json={"events": events},
    timeout=30,
)
resp.raise_for_status()
body = resp.json()

for result in body["results"]:
    if result["status"] == "error":
        print(f"event {result['index']} failed: {result['error']['code']}")
```

## Related

- `POST /v1/inventory/events` — singular event endpoint.
- `GET /v1/inventory/positions` — read back the resulting on-hand / available numbers.
- Low-stock webhooks — fired when an event drives `quantity_available` to or below `low_stock_threshold`.
