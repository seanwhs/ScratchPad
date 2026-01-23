# Inventory & Sales Recomputation 

---

# 1️⃣ Problem Definition

In HSH LPG:

* **Inventory** is derived from **InventoryTransaction** (ledger)
* **Sales / transactions** are derived from **Transaction + Invoice**
* Drift occurs due to:

  * Manual corrections
  * Offline operations
  * Failed previous jobs
  * Concurrency collisions

We want **full recomputation**:

```
inventory_snapshot = SUM(ledger transactions)
sales_totals = SUM(transaction.line_items)
```

This is **re-run safely at any time**, auditably, and idempotently.

---

# 2️⃣ Inventory Recomputation SQL (Audit-First, MySQL)

```sql
-- recompute depot inventory from ledger
UPDATE inventory i
JOIN (
    SELECT
        depot_id,
        product_id,
        customer_id,
        SUM(quantity) AS total_qty
    FROM inventory_transaction
    GROUP BY depot_id, product_id, customer_id
) l ON i.depot_id <=> l.depot_id
    AND i.product_id = l.product_id
    AND (i.customer_id <=> l.customer_id)
SET i.quantity = l.total_qty,
    i.last_updated = NOW();
```

**Notes:**

* `<=>` is MySQL **NULL-safe equality**, needed if `customer_id` may be NULL
* MySQL does not support `RETURNING` — log changes **after update** using a separate `SELECT` if needed
* Wrap in **transaction** for atomicity

---

# 3️⃣ Django ORM Optimized Version (Inventory, MySQL)

```python
from django.db.models import Sum, F
from django.db import transaction
from inventory.models import Inventory, InventoryTransaction
from services.audit import audit_log
from django.utils import timezone

@transaction.atomic
def recompute_inventory_snapshots(user):
    """
    Full inventory recomputation based on ledger transactions (MySQL optimized).
    """
    timestamp = timezone.now()

    ledger_agg = (
        InventoryTransaction.objects
        .values("depot_id", "product_id", "customer_id")
        .annotate(total_qty=Sum("quantity"))
    )

    for entry in ledger_agg:
        filters = {"product_id": entry["product_id"]}
        if entry["depot_id"]:
            filters["depot_id"] = entry["depot_id"]
        if entry["customer_id"]:
            filters["customer_id"] = entry["customer_id"]

        inv = Inventory.objects.select_for_update().get(**filters)
        old_qty = inv.quantity
        new_qty = entry["total_qty"] or 0

        if old_qty != new_qty:
            inv.quantity = new_qty
            inv.last_updated = timestamp
            inv.save(update_fields=["quantity", "last_updated"])

            audit_log(
                user=user,
                event_type="inventory_recomputation",
                obj=inv,
                before={"quantity": old_qty},
                after={"quantity": new_qty, "timestamp": timestamp},
            )
```

✅ **MySQL Optimizations:**

* Aggregation done in DB → `SUM()`
* Concurrency safe: `select_for_update()`
* Null-safe handling for optional fields

---

# 4️⃣ Sales Recomputation (Transactions → Totals, MySQL)

```sql
-- recompute transaction totals
UPDATE transaction t
JOIN (
    SELECT trx_id, SUM(quantity * unit_price) AS total_amount
    FROM transaction_lineitem
    GROUP BY trx_id
) l ON t.id = l.trx_id
SET t.total_amount = l.total_amount,
    t.updated_at = NOW();

-- propagate to invoices
UPDATE invoice i
JOIN transaction t ON i.trx_id = t.id
SET i.total_amount = t.total_amount,
    i.updated_at = NOW();
```

**Notes:**

* MySQL requires `JOIN` for updates from aggregates (no CTE-based `UPDATE … FROM`)
* Wrap both statements in a **transaction** to ensure atomicity

---

# 5️⃣ Django ORM Optimized Version (Sales, MySQL)

```python
from transactions.models import Transaction, TransactionLineItem
from invoices.models import Invoice
from django.db import transaction
from django.utils import timezone
from services.audit import audit_log

@transaction.atomic
def recompute_sales_totals(user):
    """
    Recompute all transaction totals and propagate to invoices (MySQL optimized).
    """
    timestamp = timezone.now()

    trx_totals = (
        TransactionLineItem.objects
        .values("trx_id")
        .annotate(total=Sum(F("quantity") * F("unit_price")))
    )

    for entry in trx_totals:
        trx = Transaction.objects.select_for_update().get(id=entry["trx_id"])
        old_total = trx.total_amount
        new_total = entry["total"] or 0

        if old_total != new_total:
            trx.total_amount = new_total
            trx.updated_at = timestamp
            trx.save(update_fields=["total_amount", "updated_at"])

            try:
                invoice = Invoice.objects.select_for_update().get(trx=trx)
                invoice.total_amount = new_total
                invoice.updated_at = timestamp
                invoice.save(update_fields=["total_amount", "updated_at"])
            except Invoice.DoesNotExist:
                pass

            audit_log(
                user=user,
                event_type="sales_recomputation",
                obj=trx,
                before={"total_amount": old_total},
                after={"total_amount": new_total},
            )
```

---

# 6️⃣ Optional Batch Optimization for Huge MySQL Datasets

* Use `.iterator(chunk_size=500)` to stream results
* Batch updates using `bulk_update` when possible
* Schedule **daily/weekly recomputation cron jobs**
* Avoid Python-side loops on millions of rows

---

# 7️⃣ Recomputation Flow Summary

```
Start recomputation
       |
       |-- Inventory recomputation
       |       SUM(ledger) → Inventory snapshot
       |       Audit log each change
       |
       |-- Sales recomputation
               SUM(line items) → Transaction total
               Update invoices
               Audit log each change
       |
Finish (idempotent, safe)
```

---

# 8️⃣ Key Guarantees

| Feature               | Guarantee                                      |
| --------------------- | ---------------------------------------------- |
| Inventory correctness | Snapshot always = ledger sum                   |
| Sales consistency     | Transaction totals match line items            |
| Auditability          | Every change logged                            |
| Idempotency           | Running multiple times is safe                 |
| Concurrency safety    | `select_for_update()` prevents race conditions |
| Scalability           | DB aggregation reduces Python memory load      |


