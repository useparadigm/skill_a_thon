# Example Output: `/deep-check`

> Input: "Verify billing-service consumer for `order.events` topic schema change — field `amount` renamed to `total_amount` in producer"
> Consumer repo: `acme-corp/billing-service`

---

## Deep Check: billing-service consumer for `order.events` schema change

**Verdict: CONFIRMED**

### Consumer Code Analysis

- **Repository:** acme-corp/billing-service
- **File:** `billing/consumers/order_handler.py:31`
- **Function:** `OrderEventHandler.handle_order_created`

Paradigm's `analyze_task` found `OrderEventHandler` as the primary consumer. Method body:

```python
# billing/consumers/order_handler.py

class OrderEventHandler:
    """Handles order.* events from Kafka."""

    def __init__(self, billing_repo: BillingRepository, tax_service: TaxService):
        self.billing_repo = billing_repo
        self.tax_service = tax_service

    async def handle_order_created(self, data: dict):
        """Process a new order for billing."""
        order_id = data["order_id"]
        customer_id = data["customer_id"]
        amount = data["amount"]           # <-- LINE 47: KeyError after PR ships
        currency = data["currency"]

        tax_rate = await self.tax_service.get_rate(customer_id, currency)
        billable = amount * tax_rate

        invoice = Invoice(
            order_id=order_id,
            customer_id=customer_id,
            subtotal=amount,              # <-- Also uses the old field name
            tax=billable - amount,
            total=billable,
            currency=currency,
            status="pending",
        )

        await self.billing_repo.create_invoice(invoice)
        await self._schedule_payment_collection(invoice)
```

### Finding

**The consumer WILL break.** Confirmed on 3 points:

1. **Direct field access with no fallback:** Line 47 uses `data["amount"]` — a hard key access. After the upstream PR ships, the event payload will contain `total_amount` instead of `amount`. This raises `KeyError: 'amount'` on every message.

2. **No schema validation:** There is no Pydantic model, no Avro schema, no JSON Schema validation on the incoming event. The handler trusts the raw dict.

3. **No dead letter queue visible:** The Kafka consumer configuration (`billing/config/kafka.py`) shows `enable.auto.commit=true` with no DLQ configured. Failed messages are silently committed — **data loss**.

Additionally: the handler does not use the new `discount_amount` or `items` fields. These are additive and won't cause errors, but billing will not account for discounts.

### Recommended Fix

**Option A: Fix the consumer (preferred)**

```python
# billing/consumers/order_handler.py:47
# Before:
amount = data["amount"]

# After — backward-compatible during migration:
amount = data.get("total_amount", data.get("amount"))
if amount is None:
    raise ValueError(f"Order event {data.get('order_id')} missing amount field")
```

Also update the `Invoice` creation to use `discount_amount` if available:

```python
discount = data.get("discount_amount", 0.0)
invoice = Invoice(
    order_id=order_id,
    customer_id=customer_id,
    subtotal=amount + discount,
    discount=discount,
    tax=billable - amount,
    total=billable,
    currency=currency,
    status="pending",
)
```

**Option B: Add alias in producer (quick fix)**

In the upstream PR (`order-service`), temporarily send both fields:

```python
event = {
    "total_amount": order.total_amount,
    "amount": order.total_amount,  # deprecated — remove after billing-service deploys
    ...
}
```

### Suggested PR Comment

> **Cross-system impact detected:** This PR renames `amount` → `total_amount` in the Kafka event payload for topic `order.events`.
>
> The billing-service consumer (`billing/consumers/order_handler.py:47`) accesses `data["amount"]` directly — no fallback, no schema validation. This will raise `KeyError` on every order event after this PR ships.
>
> **Recommended:** Either (a) update the billing-service consumer first and deploy it before merging this PR, or (b) add a temporary backward-compatible alias (`"amount": order.total_amount`) in the event payload.
>
> See also: no dead letter queue is configured for the billing consumer — failed messages will be silently committed (data loss).

### Migration Strategy

1. **PR 1 (billing-service):** Update consumer to accept both `amount` and `total_amount` → merge and deploy
2. **PR 2 (order-service):** This PR as-is → merge and deploy
3. **PR 3 (billing-service follow-up):** Remove fallback for `amount`, add `discount_amount` support → merge and deploy
4. **Verify:** Monitor billing consumer error rate for 24h after each deploy. Check that invoice totals match order totals including discounts.
