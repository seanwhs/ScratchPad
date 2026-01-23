# **Backend Code Walkthrough (2026)**

This code walkthourgh document covers **architecture, Django implementation, service logic, utilities, audit mechanisms, and full operational flows**, with explanations of **why and how** each part works.

The backend is designed to **safely digitize field operations**, including:

* **Inventory management** at depots and customer sites
* **Distribution creation and confirmation**
* **Transaction processing**
* **Invoice generation and delivery**
* **Meter reading updates**
* **Audit logging for operational and legal traceability**

---

## **1️⃣ Architectural Overview**

### **1.1 Backend Philosophy**

In LPG operations, software is **more than a database**:

* It **defines accountability** for physical assets (cylinders, meters, valves).
* It ensures that **transactions and inventory updates are atomic**: either everything succeeds, or nothing happens.
* It **logs every critical action**, ensuring **traceability and compliance**.

**Operational Principle:**

```
Inventory → Distribution → Transaction → Invoice → Customer → Audit
```

This flow guarantees:

1. **Physical and financial consistency**: Every transaction reflects real cylinders moved or sold.
2. **Auditability**: Regulatory or internal audits can trace every action to a user and timestamp.
3. **Concurrency safety**: Multiple drivers, depots, and admins can operate simultaneously without inconsistencies.

---

### **1.2 Key Design Principles**

| Principle              | Implementation                                                                              |
| ---------------------- | ------------------------------------------------------------------------------------------- |
| Atomicity              | Use `transaction.atomic()` for multi-model updates                                          |
| Audit Logging          | Record every critical action with `AuditLog`                                                |
| Service Layer          | All business logic lives in services, keeping ViewSets thin                                 |
| Utilities              | Shared helper functions for inventory, numbering, PDF generation                            |
| Thread-Safe Numbering  | Ensure unique TXN, INV, DIST numbers even under high concurrency                            |
| Separation of Concerns | ViewSets handle HTTP; Services handle business rules; Utilities handle low-level operations |

> **Why separation matters:** A ViewSet should not know how inventory is deducted or how PDFs are generated. It only orchestrates services.

---

## **2️⃣ Service Layer – Core Business Logic**

The **service layer** centralizes all **business-critical workflows**. It ensures:

* **Atomicity**: Partial updates cannot corrupt inventory or transactions
* **Audit logging**: All critical actions are logged
* **Reusability**: Services can be called from multiple ViewSets or background tasks
* **Testability**: Business rules are decoupled from HTTP logic

---

### **2.1 Distribution Services**

Distributions represent **physical delivery of cylinders or equipment from depot to customer or another depot**.

**Two-step process:**

1. **Create Distribution**: Records what will be distributed, but **inventory is not affected yet**.
2. **Confirm Distribution**: Deducts inventory, marks the distribution as confirmed, and logs the operation.

---

#### **2.1.1 `create_distribution()`**

```python
from django.db import transaction
from distribution.models import Distribution, DistributionItem
from core.utils.numbering import generate_number
from audit.models import AuditLog

def create_distribution(user, depot, items, remarks):
    """
    Create a distribution record.
    - Items are recorded but inventory not deducted yet.
    - Logs the creation in the AuditLog.
    
    Arguments:
        user: User performing the action
        depot: Depot from which items are distributed
        items: List of dicts {"equipment_id": int, "quantity": int}
        remarks: Optional notes
    
    Returns:
        Distribution object
    """
    # Ensure atomic operation
    with transaction.atomic():
        # Generate unique distribution number
        distribution_number = generate_number(prefix="DIST")
        
        # Create distribution record
        distribution = Distribution.objects.create(
            distribution_number=distribution_number,
            depot=depot,
            user=user,
            remarks=remarks
        )
        
        # Create DistributionItem records
        for item in items:
            DistributionItem.objects.create(
                distribution=distribution,
                equipment_id=item["equipment_id"],
                quantity=item["quantity"],
                direction="OUT"
            )
        
        # Audit log captures full payload
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

* `transaction.atomic()` ensures that if any `DistributionItem` fails, the entire operation rolls back.
* `generate_number()` provides **thread-safe, sequential IDs**.
* Audit logging records **who created the distribution and what items were included**.

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
    Confirm a distribution:
    - Deduct inventory atomically
    - Prevent double-confirmation
    - Log all inventory adjustments
    
    Arguments:
        distribution_id: Distribution to confirm
        user: User confirming
    
    Returns:
        Confirmed Distribution object
    """
    with transaction.atomic():
        # Lock the distribution record to prevent race conditions
        distribution = Distribution.objects.select_for_update().get(id=distribution_id)
        if distribution.confirmed_at:
            raise ValueError("Distribution already confirmed")
        
        # Deduct inventory for each item
        for item in distribution.items.all():
            safe_deduct_inventory(
                Inventory.objects.filter(depot=distribution.depot, equipment=item.equipment),
                quantity=item.quantity
            )
        
        # Mark as confirmed
        distribution.confirmed_at = timezone.now()
        distribution.save()
        
        # Audit log full snapshot
        AuditLog.objects.create(
            user=user,
            action="CONFIRM_DISTRIBUTION",
            entity_type="Distribution",
            entity_id=distribution.id,
            payload={"items": list(distribution.items.values("equipment_id", "quantity"))}
        )
    return distribution
```

**Notes:**

* `select_for_update()` **locks rows**, preventing **concurrent confirmations** from corrupting inventory.
* `safe_deduct_inventory()` prevents **negative stock**.
* Audit logs capture a **full snapshot** for traceability.

---

### **2.2 Transaction & Invoice Services**

#### **2.2.1 `generate_transaction()`**

```python
def generate_transaction(user, customer, items, payment_method):
    """
    Create a transaction with items:
    - Deducts inventory for each item
    - Creates TransactionItem records
    - Logs audit
    """
    from inventory.models import Inventory
    from transactions.models import Transaction, TransactionItem
    
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
                item["quantity"]
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

**Why It Matters:**

* Inventory deduction happens **within the same transaction**, ensuring no partial updates.
* `TransactionItem` captures **per-item rates and type**, enabling billing flexibility.
* Audit log ensures **every sale is traceable**.

---

#### **2.2.2 `create_invoice()`**

```python
def create_invoice(transaction_id, user):
    """
    Generate invoice for a transaction:
    - Creates unique invoice number
    - Generates PDF
    - Saves invoice
    - Logs audit
    """
    from invoices.models import Invoice
    from transactions.models import Transaction
    from core.utils.pdf import generate_pdf
    from django.utils import timezone
    
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

**Notes:**

* Generates PDF using `WeasyPrint`.
* Audit log includes **PDF path**, useful for troubleshooting and compliance.

---

### **2.3 Customer & Meter Services**

```python
def update_meter_reading(meter_id, new_reading, user):
    """
    Update a customer meter reading safely:
    - Uses select_for_update to prevent race conditions
    - Stores old reading in audit
    """
    from customers.models import Meter
    
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

* Locks the meter row to **prevent concurrent overwrites**.
* Audit captures **old and new readings**, critical for billing accuracy.

---

### **2.4 Utility Layer**

#### **2.4.1 `safe_deduct_inventory()`**

* Locks inventory row using `select_for_update()`.
* Prevents **negative stock**.
* Returns updated inventory.

#### **2.4.2 `generate_number()`**

* Sequential numbering with **optional daily reset**.
* Thread-safe for **TXN, INV, DIST numbers**.

#### **2.4.3 `generate_pdf()`**

* Converts **HTML invoice template** into PDF.
* Saves to `default_storage` (supports local, S3, GCS, etc.)
* Keeps ViewSets thin.

---

### **2.5 Complex ViewSets**

* **TransactionViewSet**: Orchestrates **transaction creation + invoice generation**.
* **InvoiceViewSet**: Returns PDF, sends email, updates status, logs audit.
* **DistributionViewSet**: Supports **create and confirm**, enforcing inventory deduction rules.
* **InventoryViewSet**: Custom endpoint `update_inventory()` uses `InventoryService` to enforce safe updates.

**Key Principle:** ViewSets only **coordinate services**, do not implement business rules.

---

## **3️⃣ End-to-End Lifecycle**

```
[API Request] → ViewSet → Service Layer → Utilities → Models → DB → AuditLog → Response
```

**Example: Transaction + Invoice**

```
POST /transactions/
    └─ TransactionViewSet.create_transaction()
        ├─ generate_transaction()
        │   ├─ Deduct inventory
        │   ├─ Create TransactionItem
        │   └─ Audit CREATE_TRANSACTION
        └─ create_invoice()
            ├─ Generate invoice number
            ├─ Generate PDF
            └─ Audit CREATE_INVOICE
Commit → DB
Response → {"transaction_number": "TXN-000012", "invoice_number": "INV-000012"}
```

**Highlights:**

* **Atomicity** ensures either transaction + invoice succeed together, or rollback.
* **Audit logs** capture full snapshots at each step.
* **Thread-safe numbering** ensures no duplicates.

---

## **4️⃣ Developer Insights**

1. **Atomic transactions** are critical in multi-model workflows.
2. **Audit logging** guarantees **operational and legal traceability**.
3. **Service-centered design** keeps ViewSets thin and testable.
4. **Thread-safe numbering** prevents ID collisions even under high concurrency.
5. **Utilities** abstract recurring operations like inventory deduction, numbering, PDF generation.
6. **Separation of creation vs confirmation** allows safer concurrency and error recovery.


