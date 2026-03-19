# Example Output: `/blast-radius`

> Input: the diff from `examples/sample-pr-diff.md` against repo `acme-corp/order-service`

---

## Section A: Blast Radius Map

### Direct Changes

| Method | File | Lines Changed |
|--------|------|---------------|
| `OrderService.create_order` | `app/services/order_service.py:23` | Field rename, discount logic |
| `OrderService._publish_order_event` | `app/services/order_service.py:42` | Kafka payload schema changed |
| `OrderService._notify_payment_service` | `app/services/order_service.py:56` | HTTP payload schema changed |
| `OrderService._calculate_discount` | `app/services/order_service.py:67` | New method |
| `Order` (model) | `app/models/order.py:8` | Column renamed + added |
| `OrderResponse` (schema) | `app/api/orders.py:15` | Response field renamed + added |

### Propagation Map

**Seed: `OrderService.create_order` (app/services/order_service.py:23)**

P1 — Override children (must co-edit):
  *(none — not overridden)*

P2 — Direct callers (likely affected):
  1. `OrderAPI.create_order` at `app/api/orders.py:28`
     Called by: API route handler
  2. `OrderService.create_bulk_orders` at `app/services/order_service.py:85`
     Calls: `create_order` in a loop

P3 — Cross-file callers (check impact):
  1. `WebhookHandler.retry_order` at `app/webhooks/retry_handler.py:12`
     Calls: `OrderService.create_order` for failed order retries
  2. `ImportService.import_orders` at `app/services/import_service.py:34`
     Calls: `OrderService.create_order` for CSV imports

P4 — Transitive callers:
  1. `AdminAPI.bulk_import` at `app/api/admin.py:67`
  2. `ScheduledTasks.process_pending_imports` at `app/tasks/scheduled.py:23`

### Remote Calls Detected

| Method | Sink Type | Target | Confidence |
|--------|-----------|--------|------------|
| `_publish_order_event` | **mq** (Kafka) | topic: `order.events` | high (1.0) — `kafka_producer.send()` known sink |
| `_notify_payment_service` | **http** | `POST {PAYMENT_SERVICE_URL}/api/payments/initiate` | high (1.0) — `httpx.post()` known sink |

### Database Access

| Method | Table | Read/Write | Source | Confidence |
|--------|-------|------------|--------|------------|
| `create_order` | `orders` | write | ORM `.add()` + `.commit()` | 0.95 |
| `_calculate_discount` | `customers` | read | ORM `select().where()` | 0.95 |
| `create_order` | `products` | read | ORM (via items validation) | 0.70 |

### Cross-System Impact Summary

| External System | Connection | Impact |
|----------------|------------|--------|
| **billing-service** | Kafka consumer on `order.events` | Receives renamed fields (`amount` → `total_amount`) |
| **payment-service** | HTTP client at `/api/payments/initiate` | Receives renamed field in payload |
| **analytics-service** | Direct DB read on `orders` table | Column `amount` renamed to `total_amount` |
| **reporting-dashboard** | Direct DB read on `orders` table | Same column rename |

---

## Section B: Severity Assessment

### BREAKING | Kafka topic `order.events` | Schema fields renamed

**What changed:** The event payload field `amount` was renamed to `total_amount`. New field `discount_amount` added. New field `items` added.

**Why this breaks:** The billing-service has a Kafka consumer that deserializes `order.events` messages. It accesses `data["amount"]` directly in `billing/consumers/order_handler.py:47`. There is no schema registry, no Avro/Protobuf enforcement, and no fallback logic (`data.get()` with default).

**Evidence:** Paradigm's propagation map shows `billing-service/consumers/order_handler.py` consuming from topic `order.events`. Method body shows:
```python
# billing-service/consumers/order_handler.py:47
amount = data["amount"]  # KeyError after this PR ships
billable = amount * tax_rate
```

**Blast radius:** The billing pipeline stops processing all new orders. `KeyError: 'amount'` on every message. Orders are created but never billed. If no dead letter queue is configured, messages are lost.

---

### BREAKING | HTTP call to payment-service | Payload field renamed

**What changed:** The `_notify_payment_service` method sends `total_amount` instead of `amount` in the JSON payload.

**Why this breaks:** The payment-service API expects `amount` in the request body. If it uses strict validation (Pydantic model with `amount: float`), requests will fail with 422. If it's flexible, it will silently ignore `total_amount` and process payments with no amount — or fail downstream.

**Evidence:** HTTP POST to `{PAYMENT_SERVICE_URL}/api/payments/initiate` with changed payload shape. Need to verify payment-service's request validation.

**Blast radius:** Payment processing fails or processes with incorrect amount. Direct revenue impact.

---

### CAUTION | API response schema | Field renamed in OrderResponse

**What changed:** `OrderResponse.amount` → `OrderResponse.total_amount`, added `discount_amount`.

**Why this may break:** Any frontend or API client that reads `response.amount` will get `undefined`/`KeyError`. This is a public API change.

**Evidence:** `app/api/orders.py:15` — Pydantic response model changed. No API versioning visible.

**Blast radius:** Frontend displays may show $0 or crash. Mobile apps on older versions will break. Third-party integrations expecting `amount` will fail.

---

### CAUTION | DB column rename | `orders.amount` → `orders.total_amount`

**What changed:** Alembic migration renames column `amount` to `total_amount` in `orders` table.

**Why this may break:** Any service or script that queries `SELECT amount FROM orders` will fail. The analytics-service and reporting-dashboard are known to read this table directly.

**Evidence:** Migration in `alembic/versions/003_rename_amount.py` uses `alter_column` — this is a live rename, not a copy-and-backfill.

**Blast radius:** Analytics queries fail immediately after migration. Reporting dashboards show errors. ETL pipelines that extract from `orders` table break.

---

### SAFE | New `_calculate_discount` method | Internal addition

**What changed:** New private method added to `OrderService`.

**Why it's safe:** This is a purely additive change. No existing interface is modified. The method is private and only called from `create_order`.

---

## Section C: Action Plan

### 1. [BLOCK MERGE] Update billing-service consumer before merging

The billing-service consumer will crash on every message after this PR ships.

- **File:** `billing-service/consumers/order_handler.py:47`
- **Change:** `data["amount"]` must handle both old and new field names during migration:
  ```python
  # Backward-compatible read
  amount = data.get("total_amount", data.get("amount"))
  ```
- **Alternative:** Add a backward-compatible alias in the producer (this PR) that sends BOTH fields:
  ```python
  event = {
      "total_amount": order.total_amount,
      "amount": order.total_amount,  # deprecated alias — remove after billing-service deploys
      ...
  }
  ```
- **Deploy order:** If fixing consumer → merge consumer PR first, then this PR. If adding alias → merge this PR with alias, then update consumer, then remove alias.

### 2. [BLOCK MERGE] Verify payment-service request validation

The payment-service receives `total_amount` instead of `amount`.

- **Action:** Run `/deep-check` on `acme-corp/payment-service` to verify how it validates the `/api/payments/initiate` request body
- **If strict validation:** Update payment-service to accept `total_amount` (or both) before merging this PR
- **If flexible:** Add `amount` alias in this PR's payload as safety measure

### 3. [BEFORE MERGE] Coordinate DB migration with downstream readers

The column rename will break any direct query using `amount`.

- **Analytics service:** Check all SQL queries against `orders` table
  - Run: `grep -r "orders.*amount\|amount.*orders" analytics-service/`
- **Reporting dashboard:** Check Metabase/Grafana queries
- **ETL pipelines:** Check Airflow DAGs or dbt models that read `orders`
- **Migration strategy:** Consider a two-phase migration:
  1. Add `total_amount` as a new column (copy from `amount`)
  2. Migrate all consumers
  3. Drop `amount` column

### 4. [BEFORE MERGE] Version the API or communicate breaking change

The `OrderResponse` schema change removes `amount` from the API.

- **Option A:** Add API versioning (`/v2/orders`) with the new schema
- **Option B:** Return both fields for backward compatibility:
  ```python
  class OrderResponse(BaseModel):
      total_amount: float
      amount: float = None  # deprecated, remove in v2

      @model_validator(mode='before')
      def backcompat(cls, values):
          values['amount'] = values.get('total_amount')
          return values
  ```
- **Option C:** Document as breaking change with migration guide for API consumers

### 5. [AFTER MERGE] Monitor for deserialization failures

- Set up alerts on billing-service consumer error rate
- Monitor payment-service 4xx/5xx spike after deploy
- Watch analytics query failure logs for 24h post-migration

---

**Suggested deep checks:**

Run `/deep-check` to verify:
- `acme-corp/billing-service` — Kafka consumer for `order.events` topic schema change
- `acme-corp/payment-service` — HTTP endpoint `/api/payments/initiate` request validation
- `acme-corp/analytics-service` — SQL queries against `orders` table column `amount`
