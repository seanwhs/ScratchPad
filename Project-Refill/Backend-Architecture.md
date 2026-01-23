# 📘 Backend Architecture

**2026 Enterprise Edition**
**Field-first, audit-safe, offline-tolerant backend architecture for Singapore LPG distribution**

> Designed around **physical asset accountability, regulatory traceability, operational correctness, and banking-grade transaction integrity**.

---

# **Executive Summary**

This document is the **definitive architectural, engineering, and operational reference** for the **HSH LPG Singapore Backend Platform**.

It formally defines:

* **Why** the system is architected this way
* **How** its subsystems operate
* **How** real-world LPG operations are executed **safely, atomically, and auditably**

The platform enforces **strict physical asset accountability** across the operational chain:

> **Inventory → Distribution → Transaction → Invoice → Meter → Audit**

Every workflow is engineered to be:

> **Atomic · Concurrency-safe · Fully auditable · Regulator-ready**

This backend is **not CRUD software**.

It is a **financial-grade, distributed transaction processing platform for physical asset accounting**, purpose-built for **high-volume LPG field operations under regulatory scrutiny**.

---

# **1. System Philosophy — Backend as Physical Truth**

---

## **1.1 Software Defines Responsibility**

In LPG operations, **software does not merely record reality — it defines accountability**.

Every physical movement must be:

* **Accounted**
* **Auditable**
* **Reversible (via compensating transactions)**
* **Legally defensible**

```
Physical Reality
      ↓
Transactional Event Ledger
      ↓
Derived State (Inventory, Balances, Invoices)
```

<img width="1536" height="1024" alt="image" src="https://github.com/user-attachments/assets/9fab89eb-4406-4d42-8cd4-b9b308a654cc" />

**State is never written directly. State is always derived from events.**

This ensures:

* Regulatory compliance
* Audit traceability
* Financial correctness
* Inventory reconciliation

---

## **1.2 Core System Guarantees (ACID + Audit)**

| Principle        | System Meaning                                   |
| ---------------- | ------------------------------------------------ |
| **Atomicity**    | All operations fully succeed or fully rollback   |
| **Consistency**  | Inventory, billing, and accounting never diverge |
| **Isolation**    | Concurrent workflows cannot corrupt shared state |
| **Durability**   | All committed actions are permanently recorded   |
| **Auditability** | Every critical action is reconstructable         |

These are **non-negotiable engineering constraints**, not optional features.

---

## **1.3 Foundational Engineering Principles**

### **Transaction-First Design**

All business-critical workflows execute inside:

```python
transaction.atomic()
```

Guaranteeing:

* Zero partial updates
* Strong rollback safety
* Deterministic state recovery

---

### **Audit-First Engineering**

Every meaningful state transition produces **immutable audit records**.

> If it isn’t audited — it didn’t happen.

Supports:

* Regulatory audits
* Dispute resolution
* Operational investigations
* Financial reconciliation

---

### **Strict Separation of Concerns**

| Layer     | Responsibility                                  |
| --------- | ----------------------------------------------- |
| ViewSets  | HTTP handling, auth, permissions, serialization |
| Services  | Business orchestration, workflows, transactions |
| Utilities | Low-level concurrency-safe primitives           |
| Models    | Persistence, constraints, invariants            |

Ensures:

* Modular design
* High testability
* Predictable behavior
* Maintainable evolution

---

### **Concurrency-Proof Numbering**

All business document numbers are generated using **database-locked sequences**, ensuring:

* No duplicates
* No gaps
* No race conditions
* Chronological traceability

---

# **2. High-Level System Architecture**

---

## **2.1 Runtime Execution Flow**

```
Client
  ↓
DRF ViewSets
  ↓
Service Layer
  ↓
Utilities + ORM
  ↓
PostgreSQL
  ↓
Audit Logging
  ↓
HTTP Response
```

---

## **2.2 Architectural Container Diagram**

```
+-------------------+        +----------------------+
|   Mobile Field    |  --->  |    API Gateway       |
|   Operations App  |        |  Django + DRF        |
+-------------------+        +----------+-----------+
                                       |
                                       v
                         +-------------+--------------+
                         |     Business Services      |
                         |----------------------------|
                         |  Auth / RBAC               |
                         |  Order Management          |
                         |  Distribution Engine       |
                         |  Inventory Ledger          |
                         |  Billing & Invoicing       |
                         |  Equipment Tracking        |
                         |  Audit & Compliance        |
                         +-------------+--------------+
                                       |
                                       v
                         +-------------+--------------+
                         |       PostgreSQL DB        |
                         |----------------------------|
                         |  Transaction Ledger        |
                         |  Inventory Views           |
                         |  Invoice Records           |
                         |  Audit Trails              |
                         +----------------------------+
```

---

## **2.3 Architectural Outcomes**

| Attribute          | Outcome                    |
| ------------------ | -------------------------- |
| Control            | Centralized business logic |
| Atomicity          | Guaranteed                 |
| Auditability       | System-wide                |
| Scalability        | Horizontally extensible    |
| Maintainability    | High                       |
| Operational Safety | Financial-grade            |

---

# **3. Core Domain Model & ERD**

---

## **3.1 ASCII ERD**

```
+-------------+       +--------------+       +----------------+
|  Customer   |       |   Order      |       |  Transaction   |
+-------------+       +--------------+       +----------------+
| id (PK)     |<----->| id (PK)      |<----->| id (PK)        |
| name        |       | customer_id  |       | order_id       |
| address     |       | status       |       | type           |
| credit_term |       | scheduled_at |       | quantity       |
+-------------+       +--------------+       | timestamp      |
                                              +----------------+
                                                       |
                                                       v
                                              +----------------+
                                              |  Inventory     |
                                              +----------------+
                                              | id (PK)        |
                                              | depot_id       |
                                              | customer_id    |
                                              | equipment_id   |
                                              | quantity       |
                                              +----------------+
                                                       |
                                                       v
                                              +----------------+
                                              |  Equipment     |
                                              +----------------+
                                              | id (PK)        |
                                              | serial_number  |
                                              | type           |
                                              | ownership      |
                                              +----------------+

+-----------+       +--------------+
|   Depot   |<----->| Distribution |
+-----------+       +--------------+
| id (PK)   |       | id (PK)      |
| name      |       | vehicle_id   |
| location  |       | driver_id    |
+-----------+       | date         |
                    +--------------+

+-------------+       +--------------+
|   Invoice   |<----->| Payment      |
+-------------+       +--------------+
| id (PK)     |       | id (PK)      |
| number      |       | amount       |
| total       |       | method       |
| status      |       | timestamp    |
+-------------+       +--------------+
```

---

## **3.2 Transaction-Driven State Model**

Instead of mutable counters:

```
Inventory = SUM(all transactions)
```

### Transaction Types

```
DELIVERY
COLLECTION
RETURN
TRANSFER
DUMP
ADJUSTMENT
WRITE_OFF
```

---

# **4. Inventory Movement Lifecycle**

---

## **4.1 Cylinder Delivery Flow**

```
Driver App → API → Ledger → Inventory View

Driver App        API              Ledger           Inventory
    |               |                 |                   |
    |--DELIVERY---->|                 |                   |
    |               |--create tx----->|                   |
    |               |                 |--recompute-------->|
    |               |<----200 OK------|                   |
```

**Effect:**

```
Depot Inventory   -= N
Customer Inventory += N
```

---

## **4.2 Cylinder Collection Flow**

```
Driver App → API → Ledger → Inventory View

Driver App        API              Ledger           Inventory
    |               |                 |                   |
    |--COLLECTION-->|                 |                   |
    |               |--create tx----->|                   |
    |               |                 |--recompute-------->|
    |               |<----200 OK------|                   |
```

**Effect:**

```
Customer Inventory -= N
Depot Inventory   += N
```

---

# **5. Distribution Engineering**

---

## **5.1 Two-Phase Distribution Workflow**

### **Phase 1 — Planning**

* Distribution created
* Items recorded
* No stock movement
* Audit logged

### **Phase 2 — Execution**

* Inventory locked
* Stock deducted atomically
* Distribution confirmed
* Final audit committed

---

## **5.2 Distribution Flow Diagram**

<img width="1536" height="1024" alt="image" src="https://github.com/user-attachments/assets/08e2e20b-9d7a-480a-b96f-e7c082485a72" />

```
Create Orders
     ↓
Route Planning
     ↓
Load Truck
     ↓
Field Execution
     ↓
Delivery / Collection
     ↓
Transaction Ledger
     ↓
Inventory Reconciliation
     ↓
Invoice Generation
     ↓
Email / Print / Audit
```

---

## **5.3 Distribution Execution Sequence**

```
Dispatcher        Driver App        API           Ledger       Invoice
     |                 |               |              |             |
     |--create run---->|               |              |             |
     |                 |               |              |             |
     |                 |--load truck-->|              |             |
     |                 |               |--TX LOAD---->|             |
     |                 |               |              |             |
     |                 |--deliver----->|              |             |
     |                 |               |--TX DEL----->|             |
     |                 |               |              |--generate-->| 
     |                 |<-------------status------------------------|
```

---

# **6. Transaction & Billing Services**

---

## **6.1 Transaction Pipeline**

```
Validate → Lock → Compute → Deduct → Record → Invoice → Audit → Commit
```

### Meter Billing – Optimistic Locking

```python
Customer.objects.filter(
    id=customer.id,
    version=customer.version
).update(...)
```

| Risk                      | Protection             |
| ------------------------- | ---------------------- |
| Concurrent meter readings | Version mismatch abort |
| Data overwrite            | Prevented              |
| Overbilling               | Impossible             |

---

# **7. Inventory Service – Authoritative Stock Ledger**

---

## **7.1 Lock Hierarchy**

```
Equipment → Owner (Depot / Customer) → Inventory Row
```

## **7.2 Atomic Deduction**

```python
qs.filter(quantity__gte=qty).update(
    quantity=F('quantity') - qty
)
```

```sql
UPDATE inventory
SET quantity = quantity - X
WHERE quantity >= X;
```

Guarantees:

* Fully atomic
* Concurrency safe
* No negative stock

---

# **8. Invoice & Billing Architecture**

---

## **8.1 Lifecycle**

```
Draft → Generated → Stored → Emailed → Archived
          ↘
           Regenerated → Re-emailed
```

## **8.2 PDF Generation**

* Engine: **WeasyPrint**
* Output: Archival-grade PDFs

```
MEDIA_ROOT/invoices/YYYYMM/TRX-YYYYMMDD-XXXXXX.pdf
```

## **8.3 Legal Compliance**

| Feature                 | Purpose             |
| ----------------------- | ------------------- |
| Deterministic filenames | Legal traceability  |
| Immutable PDFs          | Audit protection    |
| Regeneration audit      | Legal defensibility |

---

# **9. Offline-First Field Architecture**

```
Driver App (Offline)
      |
      v
[ Event Queue ]
      |
      v
[ Sync Engine ]
      |
      v
[ Central API ]
```

**Conflict Resolution**: Server ledger always wins.
Client sends immutable events, server validates causality.

---

# **10. Audit & Compliance Design**

---

## **10.1 Audit Philosophy**

> If it isn’t logged, it never happened.

## **10.2 Audit Scope & Schema**

```
+------------------+
|   AuditEvent     |
+------------------+
| id               |
| user_id          |
| event_type       |
| object_type      |
| object_id        |
| before_state     |
| after_state      |
| timestamp        |
+------------------+
```

Domains:

| Domain       | Events          |
| ------------ | --------------- |
| Distribution | Create, confirm |
| Inventory    | All adjustments |
| Transactions | Full lifecycle  |
| Invoices     | Generate, email |
| Meter        | Readings        |

---

# **11. Security Architecture**


<img width="1536" height="1024" alt="image" src="https://github.com/user-attachments/assets/c899abea-68b6-4327-9727-0aa886325803" />

```
Authentication: JWT + Refresh tokens
Authorization: Role-based access + Object-level + Field-level
```

| Risk                 | Control              |
| -------------------- | -------------------- |
| Unauthorized access  | JWT + RBAC           |
| Privilege escalation | Object permissions   |
| Data tampering       | Immutable audit logs |
| Replay attacks       | Token expiry         |

---

# **12. Deployment & Scalability**

---

## **12.1 Topology**

```
Mobile Clients
     ↓
Load Balancer
     ↓
Django API Cluster
     ↓
PostgreSQL Primary + Replica

PDF Worker Pool
Email Gateway
```

## **12.2 Scalability Strategy**

* Stateless API → horizontal scaling
* Worker pool scaling for PDF & email
* Read replicas for reporting

---

# **13. Disaster Recovery & Business Continuity**

| Component   | Strategy                   |
| ----------- | -------------------------- |
| Database    | PITR + daily snapshots     |
| Media Files | Object storage replication |
| Audit Logs  | Immutable archival         |

| Metric | Target       |
| ------ | ------------ |
| RTO    | < 15 minutes |
| RPO    | < 1 minute   |

---

# **14. System Quality Attributes**

| Attribute            | Rating           |
| -------------------- | ---------------- |
| Atomicity            | Excellent        |
| Concurrency safety   | Excellent        |
| Auditability         | Excellent        |
| Regulatory readiness | Very High        |
| Scalability          | High             |
| Maintainability      | High             |
| Security             | Enterprise-grade |

---

# **15. Engineering Philosophy**

Most systems follow:

> CRUD → Save → Hope → Reconcile

HSH LPG Platform follows:

> Lock → Validate → Transact → Audit → Prove

**Banking-grade transaction engineering applied to physical asset logistics.**

---

# **16. Final Architectural Assessment**

> **A production-grade, compliance-ready, distributed transaction processing platform for physical asset accounting.**

Suitable for:

* High-volume field operations
* Regulatory scrutiny
* Financial audits
* High-concurrency environments
* Multi-year enterprise deployment

