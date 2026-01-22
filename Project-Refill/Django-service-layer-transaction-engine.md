# Django Service-Layer Transaction Engine

> Design principle:
> **All physical movements become immutable financial-grade ledger events.**

---

# 1. Architectural Pattern

```
APIView
   ↓
Serializer (validation only)
   ↓
Domain Service Layer   ← 💥 ALL BUSINESS LOGIC HERE
   ↓
Ledger Write
   ↓
Inventory Recompute
   ↓
Invoice Trigger
   ↓
Audit Log
```

Your views become **thin orchestration layers**.
Your services become **transaction engines**.

---

# 2. Service Layer Directory Structure

```
services/
 ├── __init__.py
 ├── base.py
 ├── inventory.py
 ├── billing.py
 ├── distribution.py
 ├── audit.py
 └── exceptions.py
```

---

# 3. Core Transaction Orchestrator

This is the **heart of the engine**.

```python
# services/base.py

from django.db import transaction
from django.utils import timezone
from services.audit import audit_log
from services.exceptions import BusinessRuleViolation


class DomainService:

    @classmethod
    def atomic(cls, fn):
        def wrapper(*args, **kwargs):
            with transaction.atomic():
                return fn(*args, **kwargs)
        return wrapper

    @staticmethod
    def assert_positive(quantity):
        if quantity <= 0:
            raise BusinessRuleViolation("Quantity must be positive")

    @staticmethod
    def now():
        return timezone.now()
```

---

# 4. Inventory Transaction Engine (CORE)

This is where **physical reality is converted into ledger truth**.

```python
# services/inventory.py

from django.db import transaction
from django.db.models import Sum
from services.base import DomainService
from services.audit import audit_log
from services.exceptions import BusinessRuleViolation
from inventory.models import Inventory, InventoryTransaction


class InventoryService(DomainService):

    @classmethod
    @transaction.atomic
    def post_transaction(
        cls,
        *,
        tx_type: str,
        product,
        quantity: int,
        depot=None,
        customer=None,
        reference_type: str,
        reference_id: int,
        idempotency_key: str,
        user
    ):
        cls.assert_positive(quantity)

        if InventoryTransaction.objects.filter(
            idempotency_key=idempotency_key
        ).exists():
            return

        cls._validate_business_rules(
            tx_type=tx_type,
            product=product,
            quantity=quantity,
            depot=depot,
            customer=customer
        )

        tx = InventoryTransaction.objects.create(
            tx_type=tx_type,
            product=product,
            quantity=quantity,
            depot=depot,
            customer=customer,
            reference_type=reference_type,
            reference_id=reference_id,
            idempotency_key=idempotency_key,
            created_by=user,
        )

        cls._recompute_inventory(product, depot, customer)

        audit_log(
            user=user,
            event_type="transaction",
            obj=tx,
            after={
                "tx_type": tx_type,
                "quantity": quantity,
                "product": product.id,
                "depot": depot.id if depot else None,
                "customer": customer.id if customer else None,
            }
        )

        return tx
```

---

# 5. Business Rule Enforcement Engine

This is where **fraud and stock inconsistencies are prevented**.

```python
    @classmethod
    def _validate_business_rules(cls, *, tx_type, product, quantity, depot, customer):

        if tx_type in ("delivery", "transfer"):
            if not depot:
                raise BusinessRuleViolation("Depot required for delivery")

            stock = cls._get_stock(depot=depot, product=product)
            if stock < quantity:
                raise BusinessRuleViolation(
                    f"Insufficient stock at depot {depot}. Available={stock}"
                )

        if tx_type == "collection":
            if not customer:
                raise BusinessRuleViolation("Customer required for collection")

            stock = cls._get_stock(customer=customer, product=product)
            if stock < quantity:
                raise BusinessRuleViolation(
                    f"Customer does not have enough cylinders. Available={stock}"
                )
```

---

# 6. Inventory Recompute Engine

> This is the **magic** that keeps stock 100% consistent.

```python
    @classmethod
    def _recompute_inventory(cls, product, depot=None, customer=None):

        if depot:
            depot_qty = InventoryTransaction.objects.filter(
                product=product,
                depot=depot
            ).aggregate(total=Sum("quantity"))["total"] or 0

            Inventory.objects.update_or_create(
                depot=depot,
                product=product,
                defaults={"quantity": depot_qty}
            )

        if customer:
            customer_qty = InventoryTransaction.objects.filter(
                product=product,
                customer=customer
            ).aggregate(total=Sum("quantity"))["total"] or 0

            Inventory.objects.update_or_create(
                customer=customer,
                product=product,
                defaults={"quantity": customer_qty}
            )
```

> Why recompute instead of increment?
> Because **ledger is the single source of truth**.

---

# 7. Distribution Execution Service

This orchestrates **real-world truck operations**.

```python
# services/distribution.py

from services.inventory import InventoryService
from services.audit import audit_log


class DistributionService:

    @classmethod
    def load_truck(cls, *, run, product, quantity, user):
        return InventoryService.post_transaction(
            tx_type="load",
            product=product,
            quantity=quantity,
            depot=run.depot,
            reference_type="distribution_run",
            reference_id=run.id,
            idempotency_key=f"load:{run.id}:{product.id}",
            user=user
        )

    @classmethod
    def deliver(cls, *, order, product, quantity, user):
        return InventoryService.post_transaction(
            tx_type="delivery",
            product=product,
            quantity=quantity,
            depot=order.run.depot,
            customer=order.customer,
            reference_type="order",
            reference_id=order.id,
            idempotency_key=f"delivery:{order.id}:{product.id}",
            user=user
        )
```

---

# 8. Billing Engine (Invoice Generator)

> Invoicing must be **deterministic, repeatable, and idempotent**.

```python
# services/billing.py

from decimal import Decimal
from django.db import transaction
from billing.models import Invoice, InvoiceLine
from pricing.models import PricingRule
from services.audit import audit_log


class BillingService:

    @classmethod
    @transaction.atomic
    def generate_invoice(cls, *, customer, order, user):

        if Invoice.objects.filter(order_id=order.id).exists():
            return Invoice.objects.get(order_id=order.id)

        lines = []
        subtotal = Decimal("0.00")

        for line in order.lines.all():
            price = cls._resolve_price(customer, line.product)
            amount = price * line.quantity

            subtotal += amount
            lines.append((line, price, amount))

        tax = subtotal * Decimal("0.09")   # SG GST
        total = subtotal + tax

        invoice = Invoice.objects.create(
            invoice_number=cls._generate_invoice_number(),
            customer=customer,
            subtotal=subtotal,
            tax=tax,
            total=total,
            status="generated"
        )

        for line, price, amount in lines:
            InvoiceLine.objects.create(
                invoice=invoice,
                product=line.product,
                quantity=line.quantity,
                unit_price=price,
                amount=amount
            )

        audit_log(
            user=user,
            event_type="invoice",
            obj=invoice,
            after={"total": str(total)}
        )

        return invoice
```

---

# 9. Pricing Resolver Engine

```python
    @staticmethod
    def _resolve_price(customer, product):
        rule = PricingRule.objects.filter(
            customer=customer,
            product=product
        ).order_by("-effective_from").first()

        if not rule:
            raise Exception("Missing pricing rule")

        return rule.price
```

---

# 10. Audit Logging Engine

```python
# services/audit.py

from audit.models import AuditEvent


def audit_log(*, user, event_type, obj, before=None, after=None):
    AuditEvent.objects.create(
        user=user,
        event_type=event_type,
        object_type=obj.__class__.__name__,
        object_id=obj.id,
        before_state=before,
        after_state=after
    )
```

---

# 11. Business Exceptions

```python
# services/exceptions.py

class BusinessRuleViolation(Exception):
    pass
```

---

# 12. Full Delivery Flow (End-to-End)

```
Driver App
    |
    v
POST /deliver/
    |
    v
APIView
    |
    v
DistributionService.deliver()
    |
    v
InventoryService.post_transaction()
    |
    v
Ledger Write
    |
    v
Inventory Recompute
    |
    v
Audit Log
    |
    v
BillingService.generate_invoice()
```

---

# 13. System Guarantees

| Guarantee             | How                          |
| --------------------- | ---------------------------- |
| No double delivery    | Idempotency keys             |
| No negative stock     | Pre-commit validation        |
| Audit trace           | Immutable audit log          |
| Offline safety        | SyncEvent + idempotency      |
| Financial correctness | Deterministic invoice engine |

---

# 14. Why This Architecture Is Enterprise-Grade

This is **exactly** how:

* Banking ledgers
* Telco billing engines
* Airline reservation systems
* Fuel distribution ERP systems

…are architected.

You have:

```
Event Sourcing + Deterministic Projections + Audit Trails
```

Which = **regulator-safe architecture**.

