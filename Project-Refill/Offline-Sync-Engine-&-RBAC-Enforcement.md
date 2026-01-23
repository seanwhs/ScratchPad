# Offline Sync Engine & RBAC Enforcement

## 1️⃣ Overview

**Objective:** Enable field users (drivers, sales agents, depot managers) to **operate offline** and later **sync safely with the central database**, preserving:

* **RBAC permissions**
* **Depot/customer scope**
* **Audit trail integrity**

**Core Features:**

| Feature                             | Description                                                                 |
| ----------------------------------- | --------------------------------------------------------------------------- |
| **Local Storage**                   | Mobile/Tablet caches subset of DB (Depot, assigned Transactions, Inventory) |
| **Operation Queue**                 | Stores offline CRUD actions as “intent”                                     |
| **Sync Engine**                     | Pushes queued operations when network is available                          |
| **Conflict Detection & Resolution** | Detects concurrent updates and resolves automatically or flags for review   |
| **RBAC Enforcement**                | Validates offline operations against role & scope on sync                   |
| **Audit Trail**                     | Logs offline actions, sync outcomes, and conflict resolutions               |

---

## 2️⃣ Offline Data Model

**Local Tables (subset of main DB):**

| Table                 | Notes                                                                           |
| --------------------- | ------------------------------------------------------------------------------- |
| Inventory             | Depot-scoped snapshot                                                           |
| Transaction           | Customer-scoped snapshot                                                        |
| Distribution          | Depot-scoped snapshot                                                           |
| Invoice               | Customer-scoped snapshot                                                        |
| OfflineOperationQueue | Stores queued operations: object, action, payload, timestamp, user_id, local_id |

**OfflineOperationQueue Example:**

```json
{
  "local_id": "uuid",
  "object_type": "Inventory",
  "action": "update",
  "payload": {"quantity": 15},
  "user_id": 123,
  "timestamp": "2026-01-23T12:34:56Z",
  "status": "pending"
}
```

---

## 3️⃣ Offline → Online Sync Flow

```
Local Device / Offline Cache
   │ Queue Operations (CREATE / UPDATE / CONFIRM)
   ▼
+-------------------------+
| OfflineOperationQueue   |
+-------------------------+
   │ Sync Trigger (manual / periodic / network)
   ▼
+-------------------------+
| Sync Engine / Server    |
|------------------------|
| 1. Pull latest server state             |
| 2. Compare queued operations           |
| 3. Detect conflicts                     |
| 4. Resolve conflicts (auto/manual)     |
| 5. Apply allowed operations            |
| 6. Update local DB & queue status      |
| 7. Enforce RBAC & depot/customer scope |
+-------------------------+
   │
   ▼
+-------------------------+
| Service Layer           |
|------------------------|
| InventoryService        | ← READ/WRITE/COMPUTE(depot)
| TransactionService      | ← CREATE/READ/PROPAGATE
| DistributionService     | ← CREATE/CONFIRM(depot)
| InvoiceService          | ← READ/GENERATE
| AuditService            | ← append-only logs
| Features: atomic, idempotent |
+-------------------------+
   │
   ▼
+-------------------------+
| Database                |
|------------------------|
| Inventory / Ledger      |
| Transaction / LineItems |
| Distribution / Items    |
| Invoice                 |
| AuditLog                |
| Accounts / Roles / Permissions |
+-------------------------+
   │
   ▼
+-------------------------+
| Audit Logs              |
|------------------------|
| Records offline + online actions |
| Conflict resolution info          |
| Timestamps, scope, user, role    |
+-------------------------+
   │
   ▼
Client / UI Response
  200 OK / 403 Forbidden / Conflict Details
```

---

## 4️⃣ Conflict Detection & Resolution

| Conflict Type                     | Detection                                        | Resolution Strategy                           |
| --------------------------------- | ------------------------------------------------ | --------------------------------------------- |
| **Concurrent update**             | Local timestamp < Server timestamp               | Last-write-wins / Field merge / Manual review |
| **Deletion vs update**            | Server object deleted, local update exists       | Reject local update / Notify user             |
| **Out-of-scope operation**        | Role/scope violation at sync                     | Reject & log audit                            |
| **Dependent operation conflicts** | E.g., distribution confirmed, inventory modified | Block dependent op; notify user               |

**Inventory Conflict Example:**

1. Compare `last_updated` timestamps:

   * Local newer → apply if RBAC allows
   * Server newer → overwrite local snapshot
2. Merge non-overlapping fields automatically.
3. Irreconcilable → queue for **manual review** by Admin/DepotManager

---

## 5️⃣ RBAC Integration

* Offline operations are **tagged with user role**
* Sync engine validates **RBAC & depot/customer scope**
* Audit logs capture **offline actions + conflict outcomes**

**Offline Audit Example:**

```json
{
  "actor_id": 123,
  "role": "depot_manager",
  "action": "update",
  "object_type": "Inventory",
  "entity_id": 456,
  "before": {"quantity": 10},
  "after": {"quantity": 15},
  "offline": true,
  "resolved_conflict": false,
  "timestamp": "2026-01-23T13:05:00Z",
  "scope": {"depot_id": 7}
}
```

---

## 6️⃣ Implementation Notes

**Client (Mobile / Tablet / Desktop)**

* Local storage: SQLite / IndexedDB / CoreData
* Queue-based offline CRUD architecture
* Background service / scheduled sync

**Server (Django + DRF)**

* `/sync/` endpoint receives bulk operations (POST)
* Each operation validated against:

  * Role permissions
  * Depot/customer scope
  * Last-update timestamps
* Returns operation results: applied / rejected / conflict

**Sync Response Example:**

```json
[
  {"local_id": "uuid-1", "status": "applied", "server_id": 101},
  {"local_id": "uuid-2", "status": "conflict", "server_state": {"quantity": 20, "last_updated": "2026-01-23T12:50:00Z"}}
]
```

---

## 7️⃣ Action Legend (RBAC + Offline)

```
R       : READ / SELECT
W       : WRITE / UPDATE/DELETE
C       : CREATE / add intent
CONFIRM : Irreversible mutation
G       : Generate / PDF / Invoice / Report
(depot) : Scoped to assigned depot (middleware enforced)
(own)   : Scoped to user's customer account (middleware enforced)
```

---

## 8️⃣ Offline + Audit Highlights

* Local audit of offline operations
* Server records final logs with:

  * `offline=True`
  * Conflict resolution details
  * Original payload + server result
* Full traceability, multi-device, multi-user safe

---

## 9️⃣ Summary

* Field agents work **offline without network dependency**
* **RBAC + scope enforcement** is preserved
* Conflicts are **detectable, resolvable, auditable**
* Audit trail captures **offline + online merges**
* Scalable across multiple devices per depot/customer
* Fully integrated with **RBAC → Middleware → Service → DB → Audit**

---

## 🔹 RBAC + Offline Sync Master Flow (ASCII One Page)

```
          ┌───────────────┐
          │     Roles     │
          │───────────────│
          │ SuperAdmin    │
          │ Admin         │
          │ DepotManager  │
          │ SalesAgent    │
          │ Driver        │
          │ Auditor       │
          │ CustomerUser  │
          └─────┬─────────┘
                │ request/action
                ▼
        ┌─────────────────────┐
        │ RBAC Middleware      │
        │────────────────────│
        │ ✔ Authenticate user  │
        │ ✔ Check role/object  │
        │ ✔ Validate action    │
        │ ✔ Enforce scope      │
        │ ✔ Allow / Deny       │
        └─────┬───────────────┘
              │ allowed actions
              ▼
  ┌───────────────────────────────┐
  │ Offline Device / Local Cache   │
  │──────────────────────────────│
  │ - Local snapshot of Depot/Customer data  │
  │ - OfflineOperationQueue stores queued ops │
  │ - Queued actions: CREATE / UPDATE / CONFIRM│
  │ - Applies RBAC & depot/customer scope locally│
  └─────┬───────────────────────┘
        │ Sync Trigger (network online)
        ▼
  ┌───────────────────────────────┐
  │ Sync Engine / Server Endpoint  │
  │──────────────────────────────│
  │ 1. Pull latest server state   │
  │ 2. Compare queued operations  │
  │ 3. Detect conflicts           │
  │ 4. Resolve conflicts          │
  │    - Last-write-wins          │
  │    - Field-level merge        │
  │    - Manual review if needed  │
  │ 5. Apply allowed operations   │
  │ 6. Update local cache & queue │
  │ 7. Enforce RBAC & scope       │
  └─────┬───────────────────────┘
        │ validated & merged actions
        ▼
┌───────────────────────────┐
│      Service Layer        │
│─────────────────────────│
│ InventoryService          │ ← READ/WRITE/COMPUTE(depot)
│ TransactionService        │ ← CREATE/READ/PROPAGATE
│ DistributionService       │ ← CREATE/CONFIRM(depot)
│ InvoiceService            │ ← READ/GENERATE
│ AuditService              │ ← append-only logs
│ Features: atomic, idempotent│
└─────┬─────────────────────┘
      │ writes/updates
      ▼
┌───────────────────────────┐
│         Database          │
│─────────────────────────│
│ Inventory / Ledger        │
│ Transaction / LineItems   │
│ Distribution / Items      │
│ Invoice                   │
│ AuditLog                  │
│ Accounts / Roles / Perms  │
└─────┬─────────────────────┘
      │ append-only, scoped
      ▼
┌───────────────────────────┐
│       Audit Logs          │
│─────────────────────────│
│ user, role, action        │
│ object_type, entity_id    │
│ offline/online flag       │
│ before/after snapshots    │
│ conflict resolution info  │
│ timestamp, scope          │
└─────┬─────────────────────┘
      │
      ▼
┌───────────────────────────┐
│ Response → Client / UI    │
│ 200 OK / 403 Forbidden    │
│ Sync results: applied/conflict │
└───────────────────────────┘
```

---
