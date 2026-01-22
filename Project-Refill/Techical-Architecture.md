# **HSH Sales System (2026)**

## **Backend Architecture, Workflow & Implementation Blueprint**

A **field‑first LPG sales and logistics system** for Singapore operations, designed to safely digitize **physical asset movement, billing, and proof of delivery**, even under unreliable connectivity and regulatory scrutiny.

---

## **1. Architectural Philosophy: Backend as Physical Truth**

In LPG distribution, software does not merely *describe* reality — it **defines accountability for physical assets**.

Every cylinder collected, delivered, returned, billed, or serviced represents:

* Monetary value
* Regulated inventory
* Contractual obligation
* Audit liability

The HSH Sales System is built around one non‑negotiable mandate:

> **The backend is the authoritative source of truth for all physical and financial state.**

This ensures **risk containment**, preventing revenue leakage, stock disputes, and regulatory violations.

**Backend design principles:**

* Conservative and explicit
* Transactionally strict
* Fully auditable

---

### **1.1 Field‑First Architecture as a Risk Model**

Field staff operate in environments with poor connectivity:

* Industrial estates
* Basements and service corridors
* Roadside delivery points
* Customer back‑of‑house areas

Assuming “always‑online” behavior introduces silent failure modes:

* Dropped inventory updates
* Duplicated transactions
* Unverifiable cylinder movements
* Delayed or missing invoices

Therefore:

> **Offline‑first operation is the baseline. Connectivity is an optimization.**

All data must be:

* Captured at the point of physical action
* Safely queued on the device
* Deterministically reconciled under backend authority

---

### **1.2 Backend Discipline in Physical Operations**

Ambiguity is unacceptable. The backend must:

* Enforce **explicit ownership** of every state transition
* Reject partial or ambiguous commits
* Provide **immutable, timestamped evidence** for every action

This guides layering, transaction boundaries, locking, APIs, and auditing.

---

## **2. Core Users & Operational Goals**

### **Primary Users**

* Delivery drivers
* Field sales staff
* Tablets paired with mobile thermal printers

### **Operational Goals (Priority-Ordered)**

1. Record cylinder movements (Depot ↔ Truck)
2. Deliver cylinders, read meters, perform services
3. Generate **correct invoices immediately**
4. Print invoices on‑site
5. Auto-email PDF invoices
6. Track cylinder location (Depot / Truck / Client Site)
7. Track invoice lifecycle: **Generated → Printed → Emailed → Paid**

---

## **3. Canonical Workflows**

| Workflow     | Purpose                                 | Customer | Invoice | Print            | Email | Cylinder Tracking |
| ------------ | --------------------------------------- | -------- | ------- | ---------------- | ----- | ----------------- |
| Distribution | Depot ↔ Truck logistics                 | No       | No      | Optional receipt | No    | Depot inventory   |
| Transaction  | Customer sale / usage / service billing | Yes      | Yes     | Invoice          | Yes   | Client inventory  |

> These workflows are deliberately separated in **UI, API, services, and data models**.

---

## **4. Technical Foundation (2026)**

**Runtime & Framework:**

* Python 3.12
* Django 6 LTS
* Django REST Framework 3.15+

**Persistence & Integrity:**

* MySQL 8 (production, ACID, row-level locking)
* SQLite (development)

**Infrastructure Services:**

* JWT authentication (`simplejwt`)
* OpenAPI documentation (`drf-spectacular`)
* WeasyPrint (HTML → PDF)
* SMTP (invoice delivery)

---

## **5. Layered Backend Architecture**

```
Client (Mobile / Web)
        ↓
API Layer (ViewSets – intent only)
        ↓
Serializer Layer (validation & reconciliation)
        ↓
Service Layer (business authority)
        ↓
Domain Models (committed state)
        ↓
Database (ACID, row-locked on commit)
```

**Rules:**

* Each layer has a single responsibility
* Logic leakage between layers is prohibited
* Services are the only authority for state mutation

---

## **6. Domain-Aligned Django Applications**

* **accounts/** — identities, roles, JWT lifecycle
* **depots/**, **equipment/** — physical asset registry
* **distribution/** — depot ↔ truck logistics
* **transactions/**, **invoices/** — billing and invoicing
* **inventory/** — depot, truck, client stock
* **audit/** — append-only immutable ledger

> No domain bypasses inventory or audit controls.

---

## **7. Data Model Authority (ERD Summary)**

* Distribution: `DistributionHeader` → `DistributionItem` → atomic inventory updates
* Transactions: `TransactionHeader` → `TransactionItem` → Invoice
* Client inventory updated atomically
* Invoice lifecycle tracked explicitly

> Models represent **physical reality**, not UI convenience.

---

## **8. Serializer Layer: Validation & Offline Reconciliation**

Serializers **validate**, never mutate:

* Validate structure and enums
* Normalize offline payloads
* Accept `client_temp_id`

**Offline rules:**

* Server timestamps override client time
* Backend assigns authoritative IDs
* Last‑Write‑Wins applied; CRDT rejected

---

## **9. Service Layer: Business Authority**

**Responsibilities:**

* Inventory mutation
* Billing calculation
* Lifecycle transitions
* Document generation
* Notifications
* Audit creation

> **Views delegate, serializers validate, services decide.**

**Atomic Guarantees:**

* `@transaction.atomic`
* Row-level locking
* No partial commits or negative stock
* No concurrent double-claims

---

## **10. API Layer: Intent-Based State Transitions**

CRUD exists for exploration; **confirmation is explicit**:

```
POST /api/v1/transactions/{id}/confirm/
POST /api/v1/distributions/{id}/confirm/
```

Only confirm endpoints:

* Lock inventory
* Finalize billing
* Generate invoices
* Write audit records

---

## **11. End-to-End Operational Flow**

Frontend → Offline Queue → Backend Services → MySQL → Audit → Print / Email

**Properties:**

* Mandatory human confirmation
* Offline queueing
* Atomic backend commit
* Immediate physical + digital proof
* Immutable audit trail

---

## **12. Audit as Forensic Evidence**

* User, action, entity, payload snapshot, authoritative timestamp
* Append-only, immutable, read-only — even to admins

---

## **13. Hardware Boundary & Document Generation**

**Backend:** Generates facts, renders PDFs, produces ESC/POS payloads
**Frontend:** Handles printers and physical interaction

> Separation prevents hardware-driven data corruption.

---

## **14. Offline Sync & Conflict Strategy**

* Offline queue mandatory
* Server time authoritative
* Last-Write-Wins consistently applied

> Ensures reliable, auditable offline-first operation.

---

## **15. Implementation Roadmap (16 Weeks)**

1. Schema & contracts
2. Auth, inventory, audit
3. Distribution logic
4. Transactions & invoicing
5. Offline sync hardening
6. UAT & deployment

---

## **16. Architectural Takeaways**

* Physical assets require backend discipline
* State transitions must be explicit
* Offline capability is foundational

> **Confirmed actions define reality. Everything else is tentative.**

---

## **Appendix A – Sequence Diagrams (Textual)**

### **A1. Confirm Transaction / Distribution (Atomic Commit)**

1. Field App submits `/confirm/` intent with payload and `client_temp_id`
2. API Gateway authenticates JWT → routes to ViewSet
3. ViewSet delegates to Service Layer
4. Service Layer opens `@transaction.atomic` block
5. Rows locked (`SELECT … FOR UPDATE`)
6. Business rules validated
7. Inventory updated
8. Transaction/Distribution status → **CONFIRMED**
9. Audit record written inside atomic block
10. Commit succeeds → authoritative database state
11. API responds with server timestamp & IDs

**Failure:** Full rollback; safe retry using `client_temp_id`.

---

### **A2. Offline Sync & Reconciliation Flow**

1. Field App stores actions locally (`client_temp_id`)
2. Batch POST `/sync/` on connectivity
3. Backend authenticates, iterates payload
4. If `client_temp_id` reconciled → ignored; else → processed
5. Authoritative IDs & timestamps assigned
6. Last-Write-Wins applied
7. Response maps `client_temp_id → server_id`
8. Client purges queue

> Client timestamps never authoritative.

---

### **A3. Print & Email Invoice Flow**

1. Confirmed transaction triggers invoice generation
2. Service Layer renders HTML → PDF (WeasyPrint)
3. Invoice status: GENERATED → PRINTED / EMAILED
4. ESC/POS payload returned to frontend
5. Frontend handles printing
6. SMTP email sent asynchronously
7. Audit records print/email action

> Backend never directly controls hardware. Frontend never mutates state.

---

## **Appendix B – Data Integrity Guarantees**

* Cannot mutate inventory without confirmation
* Cannot change invoice/transaction via generic CRUD
* Client timestamps are non-authoritative
* Cannot partially commit stock movements
* Audit logs only for successful operations
* Duplicate reconciliation impossible

---

## **Appendix C – Architectural Decision Records (ADRs)**

**ADR-001:** LWW vs CRDT → Adopt LWW, manual exceptions
**ADR-002:** Explicit `/confirm/` endpoints → Align commit with physical action
**ADR-003:** Hardware-agnostic backend → Backend produces payloads; frontend handles devices

---

## **Appendix D – Plain-Language System Understanding**

* System exists to **digitize real LPG field operations safely**
* Primary users: drivers & field sales staff
* Two distinct workflows: Distribution vs Transaction
* Pressing **Confirm** = atomic, irrevocable action
* Cylinder locations derived only from confirmed actions

> **If it is not confirmed, it did not happen.**

---

## **Appendix E – API Design Alignment**

| Architectural Rule         | API Enforcement                                |
| -------------------------- | ---------------------------------------------- |
| Backend = source of truth  | Only `/confirm/` endpoints mutate inventory    |
| Offline-first              | POST accepts `client_temp_id`                  |
| Explicit state transitions | Only `/confirm/` actions                       |
| Atomicity                  | Service-layer `@transaction.atomic`            |
| Auditability               | Immutable audit records                        |
| Hardware-agnostic          | APIs return payloads, never connect to devices |

**Confirm endpoints:** idempotent, atomic, server timestamps only.
**Offline sync:** deterministic, batch-safe, no partial success.

> **If an API endpoint exists, it enforces — it cannot weaken — backend truth.**
