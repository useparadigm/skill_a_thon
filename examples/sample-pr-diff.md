# Sample PR: Rename `amount` to `total_amount` in OrderService

**Repository:** `acme-corp/order-service`
**PR #247:** "Standardize pricing fields across order model"
**Author:** @dev-maria
**Tests:** All passing | Linter: Clean | Coverage: 94%

---

```diff
diff --git a/app/models/order.py b/app/models/order.py
index 3a1f2d8..7b2e9c4 100644
--- a/app/models/order.py
+++ b/app/models/order.py
@@ -1,6 +1,7 @@
 from sqlalchemy import Column, Integer, String, Float, DateTime, ForeignKey
 from sqlalchemy.orm import relationship
 from app.database import Base
+from datetime import datetime


 class Order(Base):
@@ -9,7 +10,8 @@ class Order(Base):
     id = Column(Integer, primary_key=True, index=True)
     customer_id = Column(Integer, ForeignKey("customers.id"), nullable=False)
     status = Column(String, default="pending")
-    amount = Column(Float, nullable=False)
+    total_amount = Column(Float, nullable=False)
+    discount_amount = Column(Float, default=0.0)
     currency = Column(String(3), default="USD")
     created_at = Column(DateTime, default=datetime.utcnow)
     updated_at = Column(DateTime, default=datetime.utcnow, onupdate=datetime.utcnow)

diff --git a/app/services/order_service.py b/app/services/order_service.py
index 8c3d1a2..4f5e7b1 100644
--- a/app/services/order_service.py
+++ b/app/services/order_service.py
@@ -23,15 +23,16 @@ class OrderService:
     async def create_order(self, customer_id: int, items: list[dict]) -> Order:
         """Create a new order and publish event."""
-        total = sum(item["price"] * item["quantity"] for item in items)
+        subtotal = sum(item["price"] * item["quantity"] for item in items)
+        discount = await self._calculate_discount(customer_id, subtotal)

         order = Order(
             customer_id=customer_id,
-            amount=total,
+            total_amount=subtotal - discount,
+            discount_amount=discount,
             status="pending",
         )
         self.db.add(order)
         await self.db.commit()
         await self.db.refresh(order)

-        await self._publish_order_event(order)
+        await self._publish_order_event(order, items)
         await self._notify_payment_service(order)

         return order

-    async def _publish_order_event(self, order: Order):
+    async def _publish_order_event(self, order: Order, items: list[dict]):
         """Publish order created event to Kafka."""
         event = {
             "event_type": "order.created",
             "order_id": order.id,
             "customer_id": order.customer_id,
-            "amount": order.amount,
+            "total_amount": order.total_amount,
+            "discount_amount": order.discount_amount,
             "currency": order.currency,
+            "items": [{"sku": i["sku"], "qty": i["quantity"]} for i in items],
             "timestamp": datetime.utcnow().isoformat(),
         }
         await self.kafka_producer.send(
@@ -56,7 +58,7 @@ class OrderService:
     async def _notify_payment_service(self, order: Order):
         """Call payment service to initiate payment processing."""
         payload = {
             "order_id": order.id,
-            "amount": order.amount,
+            "total_amount": order.total_amount,
             "currency": order.currency,
             "customer_id": order.customer_id,
         }
@@ -65,3 +67,13 @@ class OrderService:
             json=payload,
             timeout=5.0,
         )
+
+    async def _calculate_discount(self, customer_id: int, subtotal: float) -> float:
+        """Calculate discount based on customer tier."""
+        customer = await self.db.execute(
+            select(Customer).where(Customer.id == customer_id)
+        )
+        customer = customer.scalar_one_or_none()
+        if customer and customer.tier == "premium":
+            return subtotal * 0.1
+        return 0.0

diff --git a/app/api/orders.py b/app/api/orders.py
index 2d4f8a3..9e1c5b2 100644
--- a/app/api/orders.py
+++ b/app/api/orders.py
@@ -15,7 +15,8 @@ class OrderResponse(BaseModel):
     id: int
     customer_id: int
     status: str
-    amount: float
+    total_amount: float
+    discount_amount: float = 0.0
     currency: str
     created_at: datetime
     updated_at: datetime

diff --git a/alembic/versions/003_rename_amount.py b/alembic/versions/003_rename_amount.py
new file mode 100644
index 0000000..a1b2c3d
--- /dev/null
+++ b/alembic/versions/003_rename_amount.py
@@ -0,0 +1,22 @@
+"""Rename amount to total_amount and add discount_amount
+
+Revision ID: 003
+"""
+from alembic import op
+import sqlalchemy as sa
+
+
+def upgrade():
+    op.alter_column('orders', 'amount', new_column_name='total_amount')
+    op.add_column('orders', sa.Column('discount_amount', sa.Float(), server_default='0.0'))
+
+
+def downgrade():
+    op.drop_column('orders', 'discount_amount')
+    op.alter_column('orders', 'total_amount', new_column_name='amount')
```

---

**PR Description:**
> Standardizes pricing fields in the order model. Renames `amount` to `total_amount` for clarity and adds `discount_amount` to support the new premium customer discount feature. Updated the Kafka event schema and payment service integration accordingly.
>
> Changes:
> - Renamed `Order.amount` → `Order.total_amount`
> - Added `Order.discount_amount` column
> - Updated order creation to calculate and apply discounts
> - Updated Kafka event payload with new field names + item details
> - Updated payment service notification payload
> - Updated API response schema
> - Added Alembic migration
>
> Tests: All 47 tests passing. Added 3 new tests for discount calculation.
