# **Backend Code Walkthrough (2026)**

This guide is a **comprehensive reference** for the HSH LPG Singapore backend. It covers architecture, Django implementation, services, utilities, business logic, audit mechanisms, and annotated code examples. Designed for **developer onboarding, audits, and long-term maintenance**.

The backend enforces **physical asset accountability**, ensuring **atomic operations, traceable workflows, and full auditability**:

**Inventory → Distribution → Transaction → Invoice → Customer/Meter → Audit**

---

## **1️⃣ Architectural Overview**

### **Backend Philosophy**

Software in LPG distribution does more than **record reality** — it **defines accountability**. Every cylinder, transaction, or invoice has operational and legal significance. The backend ensures:

* **Atomicity:** Operations complete fully or rollback entirely.
* **Traceability:** All changes are auditable.
* **Consistency:** Inventory, transactions, and invoices remain synchronized.
* **Resilience:** Supports concurrent users and unreliable network conditions.

### **Key Design Principles**

<img width="1024" height="1536" alt="image" src="https://github.com/user-attachments/assets/dbe7a876-d3b6-4fba-b630-381c190cd329" />

1. **Atomic Operations:** Multi-model updates happen inside `transaction.atomic()`.
2. **Audit Logging:** Critical actions logged with user, timestamp, entity, action, and payload.
3. **Separation of Concerns:**

   * ViewSets → HTTP/API handling
   * Services → Business workflows & atomicity
   * Utilities → Low-level operations (inventory, numbering)
   * Admin → Internal visibility without touching core logic
4. **Thread-Safe Numbering:** Unique sequential IDs for transactions, invoices, and distributions.

---

## **2️⃣ Services Layer – Core Business Logic**

The **Service Layer** centralizes business rules, making them **reusable, testable, and consistent**.

### **Distribution Services**

| Function                                           | Purpose & Behavior                                                                   |
| -------------------------------------------------- | ------------------------------------------------------------------------------------ |
| `create_distribution(user, depot, items, remarks)` | Creates a distribution; inventory **not deducted yet**; logs audit.                  |
| `confirm_distribution(distribution_id, user)`      | Confirms distribution, deducts inventory atomically, updates timestamps, logs audit. |

#### **Workflow**

1. **Create Distribution:** Items selected, `Distribution` and `DistributionItem` records created, inventory unchanged, payload captured in audit logs.
2. **Confirm Distribution:** Deducts inventory atomically, updates `confirmed_at`, logs item quantities in audit.

---

### **Example: create_distribution()**

```python
from django.db import transaction
from distribution.models import Distribution, DistributionItem
from core.utils.numbering import generate_number
from audit.models import AuditLog

def create_distribution(user, depot, items, remarks):
    with transaction.atomic():
        distribution_number = generate_number(prefix="DIST")
        distribution = Distribution.objects.create(
            distribution_number=distribution_number,
            depot=depot,
            user=user,
            remarks=remarks
        )
        for item in items:
            DistributionItem.objects.create(
                distribution=distribution,
                equipment_id=item["equipment_id"],
                quantity=item["quantity"],
                direction="OUT"
            )
        AuditLog.objects.create(
            user=user,
            action="CREATE_DISTRIBUTION",
            entity_type="Distribution",
            entity_id=distribution.id,
            payload={"items": items}
        )
    return distribution
```

**Notes:**

* `transaction.atomic()` ensures rollback if any step fails.
* `generate_number()` provides thread-safe sequential IDs.
* Audit logs preserve full payload.

---

### **Example: confirm_distribution()**

```python
from django.utils import timezone
from core.utils.inventory import safe_deduct_inventory
from distribution.models import Distribution
from inventory.models import Inventory
from audit.models import AuditLog

def confirm_distribution(distribution_id, user):
    with transaction.atomic():
        distribution = Distribution.objects.select_for_update().get(id=distribution_id)
        if distribution.confirmed_at:
            raise ValueError("Distribution already confirmed")

        for item in distribution.items.all():
            safe_deduct_inventory(
                Inventory.objects.filter(depot=distribution.depot, equipment=item.equipment),
                quantity=item.quantity
            )
        distribution.confirmed_at = timezone.now()
        distribution.save()
        AuditLog.objects.create(
            user=user,
            action="CONFIRM_DISTRIBUTION",
            entity_type="Distribution",
            entity_id=distribution.id,
            payload={"items": list(distribution.items.values("equipment_id", "quantity"))}
        )
```

**Notes:**

* `select_for_update()` prevents race conditions.
* Inventory deduction is atomic.
* Audit logs preserve a full snapshot.

---

## **3️⃣ Utilities – Safe Operations**

| Module                    | Function                                              | Purpose                                         |
| ------------------------- | ----------------------------------------------------- | ----------------------------------------------- |
| `core/utils/inventory.py` | `safe_deduct_inventory(qs, quantity)`                 | Deduct inventory safely, prevent negative stock |
| `core/utils/numbering.py` | `generate_number(prefix, length=6, reset_daily=True)` | Thread-safe sequential IDs                      |
| `core/utils/pdf.py`       | `generate_pdf(transaction, invoice_number)`           | Generates invoice PDF                           |

---

### **Example: safe_deduct_inventory()**

```python
def safe_deduct_inventory(qs, quantity: int):
    with transaction.atomic():
        inv = qs.select_for_update().first()
        if not inv:
            raise ValueError("Inventory not found")
        if inv.quantity < quantity:
            raise ValueError("Insufficient stock")
        inv.quantity -= quantity
        inv.save()
```

---

### **Example: generate_number()**

```python
from django.utils import timezone
from sequences.models import Sequence

def generate_number(prefix: str, length: int = 6, reset_daily: bool = True):
    today = timezone.now().date()
    seq, _ = Sequence.objects.get_or_create(prefix=prefix, defaults={"last_value": 0, "last_date": today})
    if reset_daily and seq.last_date != today:
        seq.last_value = 0
        seq.last_date = today
    seq.last_value += 1
    seq.save()
    return f"{prefix}-{str(seq.last_value).zfill(length)}"
```

---

## **4️⃣ Transactions & Invoices**

### **generate_transaction()**

```python
def generate_transaction(user, customer, items, payment_method):
    with transaction.atomic():
        txn_number = generate_number(prefix="TXN")
        txn = Transaction.objects.create(
            transaction_number=txn_number,
            customer=customer,
            user=user,
            payment_method=payment_method
        )
        for item in items:
            safe_deduct_inventory(Inventory.objects.filter(equipment_id=item["equipment_id"]), item["quantity"])
            TransactionItem.objects.create(transaction=txn, equipment_id=item["equipment_id"], quantity=item["quantity"])
        AuditLog.objects.create(user=user, action="CREATE_TRANSACTION", entity_type="Transaction", entity_id=txn.id, payload={"items": items, "customer_id": customer.id})
    return txn
```

### **create_invoice()**

```python
def create_invoice(transaction_id, user):
    txn = Transaction.objects.get(id=transaction_id)
    invoice_number = generate_number(prefix="INV")
    pdf_path = generate_pdf(transaction=txn, invoice_number=invoice_number)
    invoice = Invoice.objects.create(
        invoice_number=invoice_number,
        transaction=txn,
        pdf_path=pdf_path,
        generated_at=timezone.now(),
        status="generated"
    )
    AuditLog.objects.create(user=user, action="CREATE_INVOICE", entity_type="Invoice", entity_id=invoice.id, payload={"transaction_id": txn.id, "pdf_path": pdf_path})
    return invoice
```

---

## **5️⃣ Customer & Meter Operations**

```python
def update_meter_reading(meter_id, new_reading, user):
    meter = Meter.objects.select_for_update().get(id=meter_id)
    old_reading = meter.current_reading
    meter.current_reading = new_reading
    meter.last_updated = timezone.now()
    meter.save()
    AuditLog.objects.create(user=user, action="UPDATE_METER_READING", entity_type="Meter", entity_id=meter.id, payload={"old_reading": old_reading, "new_reading": new_reading})
    return meter
```

---

## **6️⃣ End-to-End Flow (Summary)**

```text
[API Request] → ViewSet → Service → Utilities → Models → DB → AuditLog → Response

Example: Transaction + Invoice
POST /transactions/
    │
    ▼
TransactionViewSet.create()
    │
    ▼
generate_transaction(user, customer, items)
    │
    ├─ Deduct inventory
    ├─ Create Transaction & Items
    └─ Audit CREATE_TRANSACTION
    │
    ▼
create_invoice(transaction.id)
    │
    ├─ Generate invoice number
    ├─ Generate PDF
    └─ Audit CREATE_INVOICE
    │
    ▼
Database commit
    │
    ▼
Response: {"transaction_number": "TXN-000012", "invoice_number": "INV-000012"}
```

---

## **7️⃣ Master Flow Diagram – Backend Lifecycle**

```text
                   ┌─────────────────────────┐
                   │       API Requests       │
                   │ POST / GET / PUT / DELETE│
                   └─────────────┬───────────┘
                                 │
                                 ▼
                   ┌─────────────────────────┐
                   │        ViewSets         │
                   │ DistributionViewSet     │
                   │ TransactionViewSet      │
                   │ InvoiceViewSet          │
                   │ CustomerViewSet         │
                   │ MeterViewSet            │
                   └─────────────┬───────────┘
                                 │
                                 ▼
                   ┌─────────────────────────┐
                   │      Service Layer      │
                   │ create_distribution()   │
                   │ confirm_distribution()  │
                   │ generate_transaction()  │
                   │ create_invoice()        │
                   │ update_meter_reading()  │
                   │ record_meter_service()  │
                   └───────┬─────────┬───────┘
                           │         │
                           ▼         ▼
             ┌────────────────┐  ┌────────────────┐
             │ Utilities      │  │ Utilities      │
             │ safe_deduct_   │  │ generate_number│
             │ inventory()    │  │ sequence       │
             │ generate_pdf() │  │                │
             └───────┬────────┘  └────────────────┘
                     │
                     ▼
             ┌─────────────────────────┐
             │      Database Models     │
             │ CustomUser              │
             │ Depot                   │
             │ Equipment               │
             │ Customer                │
             │ Inventory               │
             │ Transaction             │
             │ TransactionItem         │
             │ Distribution            │
             │ DistributionItem        │
             │ Invoice                 │
             │ Meter                   │
             │ AuditLog                │
             │ Sequence                │
             └───────┬─────────────────┘
                     │
                     ▼
             ┌─────────────────────────┐
             │ Audit Logging (Side Eff)│
             │ Records all critical    │
             │ operations:             │
             │ Distribution            │
             │ Transaction             │
             │ Invoice                 │
             │ Inventory updates       │
             │ Meter updates           │
             └─────────────────────────┘
```

---

✅ **Teaching Highlights**

* **Atomic transactions** prevent partial updates.
* **Audit logs** ensure **full operational and legal traceability**.
* **Service-centered design** keeps logic **modular and testable**.
* **Thread-safe numbering** guarantees unique IDs across concurrent operations.
* **Utilities** abstract reusable functionality like inventory deduction, numbering, and PDF generation.

---

This is a **full, polished, and reference-ready HSH LPG backend walkthrough**, covering **all major flows, service logic, utilities, audit, and end-to-end operational lifecycle**.

