# RBAC Permission Matrix & Enforcement 

---

## 1️⃣ Core Roles & Responsibilities

| Role             | Scope / Responsibilities                                               |
| ---------------- | ---------------------------------------------------------------------- |
| **SuperAdmin**   | Full system access; override any operation                             |
| **Admin**        | Depot-level management; create/edit transactions & inventory           |
| **DepotManager** | Depot stock control; approve distributions; read/write depot inventory |
| **SalesAgent**   | Create transactions & invoices; confirm distributions                  |
| **Driver**       | Confirm physical deliveries; report stock movements                    |
| **Auditor**      | Read-only access to transactions, inventory, audit logs                |
| **CustomerUser** | View invoices & balances only                                          |

---

## 2️⃣ Roles × Objects × Actions (Permission Matrix)

```
+----------------+------------+--------------+--------------+------------+-----------+-----------+
| Role \ Object  | Inventory  | Distribution | Transaction  | Invoice    | AuditLog  | Equipment |
+----------------+------------+--------------+--------------+------------+-----------+-----------+
| SuperAdmin     | R/W/C/G    | R/W/C/G      | R/W/C/G      | R/W/C/G    | R         | R/W       |
| Admin          | R/W        | R            | R/W          | R/G        | R         | R/W       |
| DepotManager   | R/W(depot) | R/C(depot)   | R            | R(depot)   | -         | -         |
| SalesAgent     | -          | C            | C            | R/G        | -         | -         |
| Driver         | -          | C/CONFIRM    | -            | -          | -         | -         |
| Auditor        | R          | R            | R            | R          | R         | R         |
| CustomerUser   | -          | -            | -            | R(own)     | -         | -         |
+----------------+------------+--------------+--------------+------------+-----------+-----------+

Legend:
R       : READ / SELECT
W       : WRITE / UPDATE/DELETE
C       : CREATE / add intent
CONFIRM : Irreversible state mutation
G       : Generate / PDF / Invoice / Report
(depot) : Scoped to assigned depot (middleware enforced)
(own)   : Scoped to user's customer account (middleware enforced)
```

---

## 3️⃣ Object-Level Scoping Rules

* **Inventory / Distribution** → `depot_id == user.depot_id`
* **Transaction / Invoice** → `customer_id == user.customer_id` (CustomerUser) or `depot_id == user.depot_id` (depot-scoped roles)
* **AuditLog** → viewable if role in `[Auditor, Admin, SuperAdmin]`
* Unauthorized access → logged, returns `403 Forbidden`

---

## 4️⃣ RBAC Enforcement (Middleware + Decorator)

### Decorator Example

```python
def rbac(object_type: str, action: str):
    def decorator(func):
        func.rbac_object = object_type
        func.rbac_action = action
        return func
    return decorator
```

### Minimal ViewSet Example

```python
from rest_framework.viewsets import ModelViewSet
from inventory.models import Inventory
from inventory.serializers import InventorySerializer

class InventoryViewSet(ModelViewSet):
    queryset = Inventory.objects.all()
    serializer_class = InventorySerializer

    @rbac("Inventory", "view")
    def list(self, request, *args, **kwargs):
        return super().list(request, *args, **kwargs)

    @rbac("Inventory", "update")
    def update(self, request, *args, **kwargs):
        return super().update(request, *args, **kwargs)
```

> Full RBACMiddleware logic lives in dev docs; the decorator + viewset example suffices for reference.

---

## 5️⃣ Enforcement Flow (Request → DB → Audit)

```
Client / UI / API
        │
        ▼
+--------------------+
| RBAC Middleware    |
|--------------------|
| ✔ Authenticate      |
| ✔ Role / object     |
| ✔ Action validation |
| ✔ Enforce scope     |
| ✔ Allow / Deny      |
+---------┬----------+
          ▼
+--------------------+
|  Service Layer     |
|--------------------|
| InventoryService     → READ/WRITE/COMPUTE(depot)
| TransactionService   → CREATE/READ/PROPAGATE
| DistributionService  → CREATE/CONFIRM(depot)
| InvoiceService       → READ/GENERATE
| AuditService         → append-only logs
| Features: atomic, idempotent, select_for_update()
+---------┬----------+
          ▼
+--------------------+
| Database           |
|--------------------|
| Inventory / Ledger
| Transaction / LineItems
| Distribution / Items
| Invoice
| AuditLog
| Accounts / Roles / Permissions
+---------┬----------+
          ▼
+--------------------+
| Audit Logs         |
|--------------------|
| user, role, action
| object_type, entity_id
| before / after
| timestamp, scope
+---------┬----------+
          ▼
+--------------------+
| Client / UI        |
| 200 OK / 403 Forbidden |
+--------------------+
```

---

## 6️⃣ Role → Key Actions (Quick Reference)

```
SuperAdmin   : ALL → R/W/C/CONFIRM/GENERATE → ALL Services → ALL DB → Audit
Admin        : Inventory R/W, Transaction R/W, Invoice R/GEN, Audit R
DepotManager : Inventory R/W(depot), Distribution R/CONFIRM(depot), Invoice R(depot)
SalesAgent   : Transaction CREATE, Invoice R/GEN, Distribution CREATE
Driver       : Distribution CONFIRM
Auditor      : ALL READ
CustomerUser : Invoice R(own)
```

---

### 🔹 Enforcement Highlights

* Middleware: pre-flight role/action/scope validation
* Service Layer: atomic, scoped, idempotent, `select_for_update()`
* DB: transactional tables for inventory, transactions, distributions, invoices, audit
* Audit: logs all allowed & denied actions
* Client: receives 200 OK / 403 Forbidden


