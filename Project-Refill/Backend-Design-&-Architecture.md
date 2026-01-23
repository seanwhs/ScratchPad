# 📘 HSH LPG Sales & Logistics System (2026)

**Design & Technical Architecture**

> A field-first, audit-safe, offline-tolerant backend architecture for Singapore LPG distribution — designed around **physical asset accountability, regulatory traceability, and operational correctness**.

---

# 1. System Philosophy — Backend as Physical Truth

In LPG operations, **software does not merely record reality — it defines responsibility.**

Every physical movement must be:

* **Accounted**
* **Auditable**
* **Reversible (via compensating transactions)**
* **Legally defensible**

Therefore:

```
Physical Reality
      ↓
Transactional Event Ledger
      ↓
Derived State (Inventory, Balances, Invoices)
```

<img width="1536" height="1024" alt="image" src="https://github.com/user-attachments/assets/9fab89eb-4406-4d42-8cd4-b9b308a654cc" />

**State is never written directly.
State is always derived from events.**

This guarantees:

* Regulatory compliance
* Audit traceability
* Financial correctness
* Inventory reconciliation

---

# 2. High-Level System Architecture

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

# 3. Core Domain Model (ASCII ERD)

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

# 4. Core Design Principle — Transaction-Driven State

## 4.1 Ledger Model

Instead of:

```
Inventory = mutable numbers
```

We use:

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

# 5. Inventory Movement Lifecycle (ASCII Sequence)

## 5.1 Cylinder Delivery Flow

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

## 5.2 Cylinder Collection Flow

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

# 6. Distribution Workflow — End-to-End

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

# 7. Distribution Execution (ASCII Sequence)

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

# 8. Billing & Invoicing Model

## 8.1 Invoice Generation Trigger

Invoice creation occurs when:

```
Delivery + Customer Pricing Rules + Tax Rules → Invoice
```

### Invoice ERD (ASCII)

```
+-------------+
|  Invoice    |
+-------------+
| id          |
| number      |
| customer_id |
| subtotal    |
| tax         |
| total       |
| status      |
+-------------+
       |
       v
+-------------+
| InvoiceLine |
+-------------+
| id          |
| invoice_id  |
| product     |
| quantity    |
| price       |
| amount      |
+-------------+
```

---

## 8.2 Invoice Generation Flow

```
Ledger → Invoice Service → PDF Generator → Email Service
```

ASCII Sequence:

```
Ledger      Billing      PDF         Email
   |            |           |             |
   |--tx------->|           |             |
   |            |--calc---->|             |
   |            |           |--generate-->|
   |            |           |             |--send-->
```

---

# 9. Offline-First Field Architecture

## 9.1 Offline Strategy

```
Local Mobile Store
     ↓
Event Queue
     ↓
Sync Engine
     ↓
Central API
```

ASCII:

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

---

## 9.2 Conflict Resolution Strategy

```
Primary Rule: Server ledger always wins.

Client:
  - Sends immutable events
  - Never overwrites state

Server:
  - Validates causality
  - Rejects impossible transitions
```

---

# 10. Security Architecture

## 10.1 Authentication

```
JWT + Refresh Tokens
Role-Based Access Control (RBAC)
```

## 10.2 Authorization Layers

```
User → Role → Permissions → Object-level checks
```

ASCII:

```
User
  |
  v
Role (driver, dispatcher, finance, admin)
  |
  v
Permission Set
  |
  v
Object Ownership + Operational Scope
```

---

# 11. Audit & Compliance Design

## 11.1 Immutable Audit Log

Every:

* Login
* Transaction
* Invoice
* Inventory adjustment
* Export

→ produces **AuditEvent**

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

---

# 12. Data Integrity Guarantees

| Risk             | Mitigation                       |
| ---------------- | -------------------------------- |
| Double delivery  | Unique delivery tokens           |
| Duplicate sync   | Idempotency keys                 |
| Stock drift      | Ledger recomputation             |
| Manual tampering | Immutable audit trails           |
| Offline fraud    | Signature + timestamp validation |

---

# 13. Deployment Architecture (Singapore Ops)

```
+----------------------+
|   Load Balancer      |
+----------+-----------+
           |
           v
+----------------------+
|   Django API Cluster |
+----------+-----------+
           |
           v
+----------------------+
| PostgreSQL Primary  |
+----------------------+
           |
           v
+----------------------+
|  Read Replica       |
+----------------------+
```

---

# 14. Scalability Strategy

```
< 10 drivers  → Single node
< 50 drivers  → API horizontal scaling
< 500 drivers → Queue + event processing workers
```

---

# 15. Failure Scenarios & Recovery

| Failure          | Handling                      |
| ---------------- | ----------------------------- |
| Network loss     | Offline queue + delayed sync  |
| Partial delivery | Partial transaction commit    |
| Driver app crash | Local transaction persistence |
| Power loss       | DB WAL recovery               |
| Double invoicing | Invoice idempotency key       |

---

# 16. Implementation Blueprint (Django)

## 16.1 Apps Layout

```
apps/
 ├── auth/
 ├── customers/
 ├── orders/
 ├── distribution/
 ├── inventory/
 ├── equipment/
 ├── billing/
 ├── audit/
```

---

## 16.2 Transaction Processing Flow

```
APIView
   ↓
Serializer validation
   ↓
Domain service
   ↓
Ledger write
   ↓
Inventory recompute
   ↓
Invoice trigger
```

ASCII:

```
Request → Serializer → Service → Ledger → Inventory → Response
```

---

# 17. Core System Guarantees

| Guarantee               | Method                           |
| ----------------------- | -------------------------------- |
| Inventory accuracy      | Ledger-first model               |
| Audit safety            | Immutable logs                   |
| Regulatory traceability | Event sourcing                   |
| Offline reliability     | Event sync                       |
| Financial correctness   | Deterministic invoice generation |

---

# 18. Final System Promise

> **Every cylinder. Every movement. Every dollar. Fully traceable. Fully defensible.**

