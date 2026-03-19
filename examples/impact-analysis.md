# Example Output: `/impact-report`

> Input: the diff from `examples/sample-pr-diff.md` against repo `acme-corp/order-service`

---

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
  *(none)*

P2 — Direct callers:
  1. `OrderAPI.create_order` at `app/api/orders.py:28`
  2. `OrderService.create_bulk_orders` at `app/services/order_service.py:85`

P3 — Cross-file callers:
  1. `WebhookHandler.retry_order` at `app/webhooks/retry_handler.py:12`
  2. `ImportService.import_orders` at `app/services/import_service.py:34`

P4 — Transitive callers:
  1. `AdminAPI.bulk_import` at `app/api/admin.py:67`
  2. `ScheduledTasks.process_pending_imports` at `app/tasks/scheduled.py:23`

### Remote Calls Detected

| Method | Sink Type | Target | Confidence |
|--------|-----------|--------|------------|
| `_publish_order_event` | mq (Kafka) | topic: `order.events` | high (1.0) |
| `_notify_payment_service` | http | `POST /api/payments/initiate` | high (1.0) |

### Database Access

| Method | Table | Read/Write | Source | Confidence |
|--------|-------|------------|--------|------------|
| `create_order` | `orders` | write | ORM `.add()` | 0.95 |
| `_calculate_discount` | `customers` | read | ORM `select()` | 0.95 |

### Cross-System Impact Summary

| External System | Connection | Impact |
|----------------|------------|--------|
| **billing-service** | Kafka consumer on `order.events` | Receives renamed fields |
| **payment-service** | HTTP client at `/api/payments/initiate` | Receives renamed field in payload |
| **analytics-service** | Direct DB read on `orders` table | Column renamed |
| **reporting-dashboard** | Direct DB read on `orders` table | Column renamed |

---

**Next step:** Run `/blast-radius` for severity assessment and action plan.
