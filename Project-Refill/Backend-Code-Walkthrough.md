# **Backend Code Walkthrough (2026)**

This document serves as a **developer, auditor, and operations reference** for the HSH LPG Singapore backend. It covers **models, services, viewsets, utilities, and workflow**, emphasizing **traceability, atomicity, and concurrency safety**.

---

## **📷 System Overview**

<img width="1536" height="1024" alt="image" src="https://github.com/user-attachments/assets/93f064ad-74fd-4972-ae25-bf4124f67350" />

**Key Features:**

* **Inventory management** at depots and customer sites
* **Distribution creation and confirmation**
* **Transaction processing** with item-level tracking
* **Invoice generation** with PDF output
* **Meter reading updates**
* **Audit logging** for operational and legal traceability
* **Thread-safe sequential numbering** for transactions, invoices, and distributions

**Backend Philosophy:** Every operation reflects a **physical truth**, meaning software **defines accountability for tangible LPG assets**, not just records them.

---

## **1️⃣ Architectural Overview**

<img width="1024" height="1536" alt="image" src="https://github.com/user-attachments/assets/4d9627ee-22bd-4f0d-a1e1-bde6efea98f8" />

### **1.1 Backend Philosophy**

* **Defines accountability** – Cylinders, meters, and invoices are legal and operational artifacts.
* **Atomic updates** – Either all steps succeed or the operation rolls back completely.
* **Auditability** – Every action is logged with `AuditLog`.
* **Concurrency-safe** – `select_for_update()` and thread-safe numbering prevent conflicts in high-concurrency scenarios.

**Operational Flow:**

```
Inventory → Distribution → Transaction → Invoice → Customer → Audit
```

**Benefits:**

* **Physical consistency:** Inventory matches real-world stock.
* **Financial accuracy:** Transactions align with actual deliveries.
* **Regulatory compliance:** Full traceability for audits.

---

### **1.2 Key Design Principles**

| Principle              | Implementation                                                                        |
| ---------------------- | ------------------------------------------------------------------------------------- |
| Atomicity              | Multi-model updates wrapped in `transaction.atomic()`                                 |
| Audit Logging          | Record all critical actions with `AuditLog`                                           |
| Service Layer          | Business logic lives in services; ViewSets remain thin                                |
| Utilities              | Shared functions for inventory, numbering, PDF generation                             |
| Thread-Safe Numbering  | Ensures unique TXN, INV, DIST numbers even under high concurrency                     |
| Separation of Concerns | ViewSets orchestrate services; Services handle rules; Utilities are low-level helpers |

> **Why separation matters:** ViewSets do not handle inventory deduction, numbering, or PDF generation—they only orchestrate services.

---

## **2️⃣ Service Layer – Core Business Logic**

The **service layer centralizes all business-critical workflows**, ensuring:

* **Atomicity:** Prevents partial updates to inventory, transactions, or invoices.
* **Audit logging:** Every critical action is recorded for traceability.
* **Reusability:** Services are callable from multiple endpoints or background tasks.
* **Testability:** Business logic is decoupled from HTTP handling.

---

### **2.1 Distribution Services**

**Distributions** represent the **physical movement of cylinders or equipment**.

#### **2.1.1 `create_distribution()`**

```python
from django.db import transaction
from distribution.models import Distribution, DistributionItem
from core.utils.numbering import generate_number
from audit.models import AuditLog

def create_distribution(user, depot, items, remarks):
    """
    Create a distribution record without affecting inventory.
    Logs the creation in AuditLog.
    """
    with transaction.atomic():  # Ensures rollback if any step fails
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

**Key Points:**

* Uses `transaction.atomic()` to **rollback the whole operation** if any step fails.
* `generate_number()` ensures **unique, sequential distribution numbers** even under concurrency.
* Audit logs preserve **full payload** for traceability.

---

#### **2.1.2 `confirm_distribution()`**

```python
from django.utils import timezone
from core.utils.inventory import safe_deduct_inventory
from distribution.models import Distribution
from inventory.models import Inventory
from audit.models import AuditLog

def confirm_distribution(distribution_id, user):
    """
    Confirm distribution and deduct inventory atomically.
    Logs inventory adjustments.
    """
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
    return distribution
```

**Critical Details:**

* `select_for_update()` **locks the row** to prevent concurrent confirmations.
* `safe_deduct_inventory()` ensures **inventory never goes negative**.
* Full **audit snapshot** is captured for compliance.

---

### **2.2 Transaction & Invoice Services**

#### **2.2.1 `generate_transaction()`**

```python
from django.db import transaction
from inventory.models import Inventory
from transactions.models import Transaction, TransactionItem
from audit.models import AuditLog
from core.utils.inventory import safe_deduct_inventory
from core.utils.numbering import generate_number

def generate_transaction(user, customer, items, payment_method):
    """
    Generate a full transaction:
    - Deduct inventory for each item
    - Create Transaction and TransactionItem records
    - Logs all actions in AuditLog
    """
    with transaction.atomic():
        txn_number = generate_number(prefix="TXN")
        txn = Transaction.objects.create(
            transaction_number=txn_number,
            customer=customer,
            user=user,
            payment_method=payment_method
        )

        for item in items:
            safe_deduct_inventory(
                Inventory.objects.filter(equipment_id=item["equipment_id"]),
                quantity=item["quantity"]
            )
            TransactionItem.objects.create(
                transaction=txn,
                equipment_id=item["equipment_id"],
                quantity=item["quantity"],
                rate=item.get("rate", 0),
                type=item.get("type", "SALES")
            )

        AuditLog.objects.create(
            user=user,
            action="CREATE_TRANSACTION",
            entity_type="Transaction",
            entity_id=txn.id,
            payload={"items": items, "customer_id": customer.id}
        )
    return txn
```

**Flow Diagram:**

```
generate_transaction()
      │
Transaction.objects.create()
      │
Loop items:
 ├─ safe_deduct_inventory()
 └─ TransactionItem.objects.create()
      │
AuditLog.objects.create()
      │
Return Transaction
```

---

#### **2.2.2 `create_invoice()`**

```python
from django.utils import timezone
from invoices.models import Invoice
from transactions.models import Transaction
from audit.models import AuditLog
from core.utils.numbering import generate_number
from core.utils.pdf import generate_pdf

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

    AuditLog.objects.create(
        user=user,
        action="CREATE_INVOICE",
        entity_type="Invoice",
        entity_id=invoice.id,
        payload={"transaction_id": txn.id, "pdf_path": pdf_path}
    )

    return invoice
```

**ASCII Flow:**

```
create_invoice()
      │
Transaction.objects.get()
      │
generate_number()
      │
generate_pdf()
      │
Invoice.objects.create()
      │
AuditLog.objects.create()
      │
Return Invoice
```

---

### **2.3 Customer & Meter Services**

```python
def update_meter_reading(meter_id, new_reading, user):
    """
    Safely update a customer's meter reading:
    - Locks the meter row
    - Saves old and new reading in audit
    """
    from customers.models import Meter
    from django.utils import timezone
    from audit.models import AuditLog

    meter = Meter.objects.select_for_update().get(id=meter_id)
    old_reading = meter.current_reading
    meter.current_reading = new_reading
    meter.last_updated = timezone.now()
    meter.save()

    AuditLog.objects.create(
        user=user,
        action="UPDATE_METER_READING",
        entity_type="Meter",
        entity_id=meter.id,
        payload={"old_reading": old_reading, "new_reading": new_reading}
    )
    return meter
```

**Key Insight:**
`select_for_update()` ensures **no two updates conflict**, and audit logs capture every change.

---

### **2.4 Utility Layer**

#### **2.4.1 `safe_deduct_inventory()`**

* Deducts inventory **without going negative**
* Locks rows via `select_for_update()`
* Works atomically

#### **2.4.2 `generate_number()`**

```python
from django.utils import timezone
from sequences.models import Sequence

def generate_number(prefix: str, length: int = 6, reset_daily: bool = True):
    today = timezone.now().date()
    seq, _ = Sequence.objects.get_or_create(
        prefix=prefix,
        defaults={"last_value": 0, "last_date": today}
    )
    if reset_daily and seq.last_date != today:
        seq.last_value = 0
        seq.last_date = today
    seq.last_value += 1
    seq.save()
    return f"{prefix}-{str(seq.last_value).zfill(length)}"
```

**Highlights:**

* Thread-safe sequential IDs
* Daily reset ensures invoice/transaction numbers start fresh every day

#### **2.4.3 `generate_pdf()`**

```python
from django.template.loader import render_to_string
from weasyprint import HTML
from django.core.files.storage import default_storage

def generate_pdf(transaction, invoice_number):
    html_content = render_to_string("invoices/invoice_template.html", {"transaction": transaction})
    pdf_file_path = f"invoices/{invoice_number}.pdf"
    HTML(string=html_content).write_pdf(pdf_file_path)

    if default_storage:
        with open(pdf_file_path, "rb") as f:
            default_storage.save(pdf_file_path, f)

    return pdf_file_path
```

* Renders invoice **template → PDF**
* Saves via **default storage** for future access
* Atomic and reusable utility

---

### **2.5 Complex ViewSets**

<img width="1536" height="1024" alt="image" src="https://github.com/user-attachments/assets/caa2d99b-07a5-4717-b9c1-05418c0c4a81" />

**Example – TransactionViewSet**

```python
class TransactionViewSet(viewsets.ModelViewSet):
    queryset = Transaction.objects.all()
    serializer_class = TransactionSerializer

    def create(self, request, *args, **kwargs):
        user = request.user
        customer = request.data["customer"]
        items = request.data["items"]
        payment_method = request.data.get("payment_method", "CASH")

        txn = generate_transaction(user, customer, items, payment_method)
        invoice = create_invoice(txn.id, user)

        return Response({
            "transaction_number": txn.transaction_number,
            "invoice_number": invoice.invoice_number
        }, status=201)
```

**Principle:**
ViewSets **orchestrate services**, never implement core business logic themselves. This ensures **separation of concerns**, testability, and audit compliance.

---

### **3️⃣ End-to-End Flow**

```
[API Request] -> ViewSet -> Service Layer -> Utilities -> Models -> DB -> AuditLog -> Response

Example: Transaction + Invoice

POST /transactions/
      │
TransactionViewSet.create()
      │
 ├─ generate_transaction()
 │    ├─ safe_deduct_inventory()
 │    ├─ TransactionItem.objects.create()
 │    └─ AuditLog CREATE_TRANSACTION
 └─ create_invoice()
      ├─ generate_number()
      ├─ generate_pdf()
      ├─ Invoice.objects.create()
      └─ AuditLog CREATE_INVOICE
      │
Commit -> DB
Response -> {"transaction_number": "TXN-000012", "invoice_number": "INV-000012"}
```

---

### **4️⃣ Summary Table of Services & Utilities**

| Layer / Module         | Function                | Purpose / Behavior                                    | Notes                        |
| ---------------------- | ----------------------- | ----------------------------------------------------- | ---------------------------- |
| Distribution Service   | create_distribution()   | Records planned distributions, logs audit             | Inventory not deducted yet   |
|                        | confirm_distribution()  | Deduct inventory, confirm distribution, audit log     | Uses select_for_update()     |
| Transaction Service    | generate_transaction()  | Deduct inventory, create transaction items, audit log | Atomic and traceable         |
| Invoice Service        | create_invoice()        | Generate invoice number, PDF, record, audit log       | PDF path logged              |
| Customer/Meter Service | update_meter_reading()  | Update meter safely, audit old/new reading            | select_for_update()          |
| Inventory Utilities    | safe_deduct_inventory() | Deduct inventory safely, prevent negative stock       | Locks row, atomic            |
| Numbering Utilities    | generate_number()       | Thread-safe sequential numbers                        | Optional daily reset         |
| PDF Utilities          | generate_pdf()          | Generate PDF from template                            | Supports default_storage     |
| ViewSets               | TransactionViewSet      | Orchestrates transaction + invoice                    | Services handle logic        |
|                        | DistributionViewSet     | Orchestrates create/confirm distribution              | Inventory handled in service |

---

### **5️⃣ Master Backend Workflow Diagram**

```
                                 ┌─────────────────────────────┐
                                 │        API Requests         │
                                 │ POST / GET / PUT / DELETE   │
                                 └─────────────┬──────────────┘
                                               │
                                 ┌─────────────────────────────┐
                                 │         ViewSets            │
                                 │-----------------------------│
                                 │ TransactionViewSet          │
                                 │ DistributionViewSet         │
                                 │ InvoiceViewSet              │
                                 │ CustomerViewSet             │
                                 │ MeterViewSet                │
                                 │ InventoryViewSet            │
                                 └─────────────┬──────────────┘
                                               │
        ┌──────────────────────────────────────┼──────────────────────────────────────┐
        │                                      │                                      │
        ▼                                      ▼                                      ▼
┌─────────────────────┐               ┌─────────────────────┐               ┌─────────────────────┐
│  Service Layer       │               │ Service Layer       │               │ Service Layer       │
│--------------------- │               │---------------------│               │---------------------│
│ create_distribution()│               │ confirm_distribution()│             │ generate_transaction()│
│ create_invoice()     │               │ update_meter_reading()│             │                     │
└─────────────┬────────┘               └─────────────┬────────┘               └─────────────┬────────┘
              │                                      │                                      │
              ▼                                      ▼                                      ▼
       ┌─────────────┐                         ┌─────────────┐                        ┌─────────────┐
       │ Utilities   │                         │ Utilities   │                        │ Utilities   │
       │-------------│                         │-------------│                        │-------------│
       │ safe_deduct_inventory()               │ generate_number()                     │ generate_pdf()│
       │ generate_number()                     │                                   │                 │
       │ generate_pdf()                        │                                   │                 │
       └───────┬────────┘                     └───────┬────────┘                     └───────┬───────┘
               │                                      │                                      │
               ▼                                      ▼                                      ▼
       ┌─────────────────────────┐           ┌─────────────────────────┐           ┌─────────────────────────┐
       │     Database Models      │           │     Database Models      │           │     Database Models      │
       │-------------------------│           │-------------------------│           │-------------------------│
       │ Distribution             │           │ Inventory               │           │ Transaction             │
       │ DistributionItem         │           │ Customer                │           │ TransactionItem         │
       │ Invoice                  │           │ Meter                   │           │                         │
       │ Equipment                │           │ Depot                   │           │                         │
       │ Customer                 │           │                         │           │                         │
       │ Transaction              │           │                         │           │                         │
       │ Sequence                 │           │                         │           │                         │
       └─────────────┬───────────┘           └─────────────┬───────────┘           └─────────────┬───────────┘
                     │                                     │                                      │
                     ▼                                     ▼                                      ▼
              ┌─────────────────────────────────────────────────────────────────────────────────┐
              │                           Audit Logging (Side Effect)                            │
              │─────────────────────────────────────────────────────────────────────────────────│
              │ Logs all critical actions:                                                     │
              │ - CREATE / CONFIRM DISTRIBUTION                                               │
              │ - CREATE TRANSACTION / TRANSACTION ITEMS                                       │
              │ - CREATE INVOICE                                                             │
              │ - INVENTORY DEDUCTION                                                        │
              │ - METER UPDATES                                                             │
              │ Stores: user, timestamp, entity type, entity id, payload                      │
              └─────────────────────────────────────────────────────────────────────────────────┘
```

---


