# Inventory Reconciliation Engine

**Daily Closing Algorithm & Audit-Safe Stock Control**

---

# 1. Why Reconciliation Is Mandatory

In physical logistics systems:

* Drivers miscount
* Cylinders leak
* Returns get forgotten
* Offline sync duplicates occur
* Manual corrections happen

**Drift is inevitable.**

What matters is:

> **Your system must detect, explain, correct, and audit that drift.**

---

# 2. Reconciliation Philosophy

The ledger is **absolute truth**.

The snapshot inventory table is **a cached projection**.

Reconciliation validates:

```
SUM(ledger movements)  ==  snapshot inventory
```

If not:

> **You do not fix silently — you generate correction transactions.**

---

# 3. Reconciliation Architecture

```
CRON (00:30 daily)
     |
     v
Reconciliation Engine
     |
     v
Ledger Aggregation
     |
     v
Snapshot Compare
     |
     v
Drift Detection
     |
     v
Correction Transaction (if needed)
     |
     v
Audit Log + Reconciliation Report
```

---

# 4. Core Algorithm (High Level)

For each:

* Depot × Equipment
* Customer × Equipment

Do:

```
ledger_sum = SUM(inventory_transactions)
snapshot_qty = inventory.quantity

IF ledger_sum != snapshot_qty:
    delta = ledger_sum - snapshot_qty
    generate adjustment transaction(delta)
    log reconciliation event
```

---

# 5. Django Reconciliation Service (Production-Grade)

```python
# services/reconciliation.py

from django.db import transaction
from django.db.models import Sum
from django.utils import timezone
from inventory.models import Inventory, InventoryTransaction
from services.audit import audit_log
from services.exceptions import BusinessRuleViolation


class InventoryReconciliationService:

    @classmethod
    @transaction.atomic
    def daily_close(cls, *, executed_by):

        timestamp = timezone.now()

        cls._reconcile_depots(executed_by, timestamp)
        cls._reconcile_customers(executed_by, timestamp)

    # ---------- DEPOT STOCK ----------

    @classmethod
    def _reconcile_depots(cls, executed_by, timestamp):

        inventory_rows = Inventory.objects.filter(
            depot__isnull=False
        ).select_for_update()

        for inv in inventory_rows:

            ledger_qty = InventoryTransaction.objects.filter(
                product=inv.product,
                depot=inv.depot
            ).aggregate(total=Sum("quantity"))["total"] or 0

            if ledger_qty != inv.quantity:
                cls._correct_inventory(
                    inv=inv,
                    ledger_qty=ledger_qty,
                    snapshot_qty=inv.quantity,
                    executed_by=executed_by,
                    timestamp=timestamp
                )

    # ---------- CUSTOMER STOCK ----------

    @classmethod
    def _reconcile_customers(cls, executed_by, timestamp):

        inventory_rows = Inventory.objects.filter(
            customer__isnull=False
        ).select_for_update()

        for inv in inventory_rows:

            ledger_qty = InventoryTransaction.objects.filter(
                product=inv.product,
                customer=inv.customer
            ).aggregate(total=Sum("quantity"))["total"] or 0

            if ledger_qty != inv.quantity:
                cls._correct_inventory(
                    inv=inv,
                    ledger_qty=ledger_qty,
                    snapshot_qty=inv.quantity,
                    executed_by=executed_by,
                    timestamp=timestamp
                )
```

---

# 6. Correction Transaction Engine

> Corrections are **ledger transactions**, not silent updates.

```python
    @classmethod
    def _correct_inventory(
        cls,
        *,
        inv,
        ledger_qty,
        snapshot_qty,
        executed_by,
        timestamp
    ):
        delta = ledger_qty - snapshot_qty

        InventoryTransaction.objects.create(
            tx_type="reconciliation_adjustment",
            product=inv.product,
            quantity=delta,
            depot=inv.depot,
            customer=inv.customer,
            reference_type="daily_close",
            reference_id=int(timestamp.strftime("%Y%m%d")),
            idempotency_key=f"recon:{timestamp.date()}:{inv.id}",
            created_by=executed_by,
        )

        inv.quantity = ledger_qty
        inv.save(update_fields=["quantity", "last_updated"])

        audit_log(
            user=executed_by,
            event_type="inventory_reconciliation",
            obj=inv,
            before={"quantity": snapshot_qty},
            after={
                "quantity": ledger_qty,
                "ledger": ledger_qty,
                "delta": delta
            }
        )
```

---

# 7. Reconciliation Ledger Entry Types

| tx_type                     | Meaning           |
| --------------------------- | ----------------- |
| `delivery`                  | Outgoing stock    |
| `collection`                | Incoming stock    |
| `transfer`                  | Depot transfer    |
| `load`                      | Truck load        |
| `unload`                    | Truck unload      |
| `manual_adjustment`         | Admin correction  |
| `reconciliation_adjustment` | Daily closing fix |

This makes **regulatory review trivial**.

---

# 8. Reconciliation Report Model

Every closing generates a **forensic report**.

```python
# inventory/models.py

class ReconciliationReport(models.Model):
    date = models.DateField(unique=True)
    executed_by = models.ForeignKey(
        "accounts.Account",
        on_delete=models.PROTECT
    )
    mismatches_found = models.IntegerField()
    corrections_applied = models.IntegerField()
    created_at = models.DateTimeField(auto_now_add=True)
```

---

# 9. Reconciliation Reporting Service

```python
    @classmethod
    def generate_report(cls, *, date, executed_by):

        txs = InventoryTransaction.objects.filter(
            reference_type="daily_close",
            reference_id=int(date.strftime("%Y%m%d"))
        )

        return ReconciliationReport.objects.create(
            date=date,
            executed_by=executed_by,
            mismatches_found=txs.count(),
            corrections_applied=txs.count()
        )
```

---

# 10. Daily Closing Timeline

```
23:59 — Cut-off
00:00 — Lock writes (optional)
00:01 — Run reconciliation
00:10 — Generate report
00:15 — Release locks
```

---

# 11. Safety Guarantees

| Risk               | Protection                |
| ------------------ | ------------------------- |
| Drift accumulation | Daily forced correction   |
| Silent loss        | Ledger-driven recompute   |
| Audit gaps         | Mandatory correction logs |
| Double adjustment  | Idempotency keys          |
| Concurrent writes  | select_for_update         |

---

# 12. Real-World Example

**Depot A:**

| Source     | Quantity |
| ---------- | -------- |
| Ledger sum | 1,238    |
| Snapshot   | 1,240    |

Delta = **-2**

System auto-generates:

```
tx_type = reconciliation_adjustment
quantity = -2
reason = daily close
```

This:

* Explains loss
* Balances books
* Preserves forensic history

---

# 13. Field Reality Handling

If daily reconciliation repeatedly finds **same negative drift**:

→ triggers **investigation workflow**

You can auto-escalate:

* Supervisor alert
* CCTV review
* Driver retraining
* Process correction



