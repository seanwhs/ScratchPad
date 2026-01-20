# HSH Sales System (2026)

## Backend Architectural Blueprint & Implementation Guide

---

## 1. Architectural Philosophy: Backend as Physical Truth

In the LPG distribution domain, software does not merely *record* reality — it **defines accountability for physical assets**. Every cylinder moved, delivered, returned, or billed carries direct financial, operational, and regulatory consequences. The HSH Sales System is therefore built around a single, non‑negotiable mandate:

> **The backend is the authoritative source of truth for all physical and financial state.**

This mandate is not a stylistic preference or a software purity argument. It is a **risk‑containment strategy**.

Unlike purely digital systems, LPG operations manage reusable, high‑value assets that physically traverse depots, trucks, and customer sites. Any divergence between physical reality and digital state immediately manifests as revenue leakage, stock disputes, audit failures, and regulatory exposure. As a result, the backend architecture is intentionally conservative, explicit, and auditable by design.

### 1.1 Field‑First Architecture as a Risk Model

Field staff operate in environments where unreliable connectivity is the norm rather than the exception: industrial estates, basements, service corridors, roadside delivery points, and customer back‑of‑house areas. Designing for “always‑online” behavior introduces silent corruption vectors:

* lost inventory updates
* duplicated transactions
* unverifiable cylinder movements

The system therefore assumes **offline‑first operation as its baseline**. Network availability is treated strictly as an optimization, never as a prerequisite. Data must be capturable immediately at the point of physical action and reconciled deterministically under backend control once connectivity resumes.

### 1.2 Backend Discipline in Physical Operations

In LPG logistics, ambiguity is operationally and financially unacceptable. The backend must:

* enforce clear ownership of every state transition
* reject partial or ambiguous commits
* provide immutable, timestamped evidence for every action

This philosophy directly informs the system’s layering, transaction boundaries, concurrency controls, and audit mechanisms described in the sections that follow.

---

## 2. Technical Foundation & Stack Rationale

### 2.1 Core Technology Stack (2026)

The technology stack is selected for **predictability, transactional safety, and long‑term maintainability**, not novelty or framework churn.

**Runtime & Framework**

* Python 3.12
* Django 6 (LTS)
* Django REST Framework (DRF) 3.15+

**Persistence & Integrity**

* MySQL 8 (production) — ACID‑compliant with row‑level locking
* SQLite (local development)

**Infrastructure Services**

* JWT authentication (`djangorestframework-simplejwt`)
* OpenAPI documentation (`drf-spectacular`)
* WeasyPrint for HTML → PDF generation
* SMTP for invoice and receipt delivery

### 2.2 Asynchronous Capability as Load Protection

Django 6’s asynchronous support is leveraged as a **protective mechanism**, not as a core execution model. It is applied to the system’s edges to maintain responsiveness under real‑world field load:

* concurrent mobile device sync bursts
* high‑latency cellular networks
* I/O‑heavy workflows such as PDF rendering and email dispatch

Async execution is intentionally excluded from core inventory mutation and financial commits. These operations remain strictly synchronous, transactional, and database‑bound. Responsiveness must never be allowed to compromise correctness.

### 2.3 Foundational Architectural Principles

| Principle                | Technical Enforcement        | Strategic Outcome                             |
| ------------------------ | ---------------------------- | --------------------------------------------- |
| Resource‑Oriented Design | RESTful domain entities      | Predictable APIs, parallel development        |
| Explicit Versioning      | `/api/v1/`                   | Safe evolution without breaking field devices |
| JWT Security             | Short‑lived access + refresh | Secure, mobile‑appropriate authentication     |
| Idempotency              | TRX / DIST / INV identifiers | Duplicate prevention and audit alignment      |

This foundation ensures the backend behaves as a **deterministic state machine for real‑world logistics**, not a passive data store.

---

## 3. Layered Backend Architecture

The HSH Sales System enforces a **strict layered architecture**. Business logic is permitted to exist only in its designated layer. Violations are treated as architectural defects.

```
Client (Mobile / Web)
        ↓
API Layer (ViewSets)
        ↓
Serializer Layer (Validation & Normalization)
        ↓
Service Layer (Business Rules & Orchestration)
        ↓
Domain Models (Committed State)
        ↓
Database (ACID, Locked on Commit)
```

Each layer has a single responsibility and an explicit prohibition list, ensuring logic cannot leak upward or downward in ways that bypass controls.

---

## 4. Modular Django Application Structure

The backend is organized into **domain‑aligned Django applications** to minimize blast radius and enforce separation of concerns.

### 4.1 Core Domains

* **accounts/** — User identities, role separation, and JWT lifecycle
* **depots/** & **equipment/** — Authoritative physical asset registry (CYL 9kg, 12.7kg, 14kg, 50kg POL/L)
* **distribution/** — Internal logistics (Depot ↔ Truck), enforcing strict movement semantics
* **transactions/** & **invoices/** — Financial engine: usage billing, cylinder charges, services, and payments
* **inventory/** — Real‑time stock counters across depots, trucks, and customer sites
* **audit/** — Immutable, append‑only system activity log

This structure guarantees that logistics logic cannot bypass inventory enforcement and that financial logic cannot mutate physical state without traceability.

---

## 5. Configuration & Environment Control (settings.py)

Centralized configuration acts as the system’s **control plane**, ensuring security, consistency, and environmental isolation.

Key configuration domains include:

* **Authentication (SIMPLE_JWT)** — Short‑lived access tokens stored only in volatile frontend memory; secure refresh tokens support full‑day field shifts.
* **REST Framework Defaults** — Global JWT enforcement and mandatory pagination to protect mobile clients from payload bloat.
* **External Integrations** — WeasyPrint and SMTP configured centrally to prevent credential leakage into business logic.

This approach ensures environment‑specific behavior is explicit, auditable, and safe.

---

## 6. Serializer Layer: Validation & Offline Reconciliation

Serializers function as the system’s **data integrity gatekeepers**. They reconcile the imperfect, offline reality of field input with the authoritative expectations of the backend.

They are responsible for:

* structural and type validation
* enum and contract enforcement
* offline reconciliation support

They are explicitly forbidden from:

* mutating inventory
* calculating prices
* changing lifecycle state

### 6.1 Offline Reconciliation Strategy

When offline, the client generates a `client_temp_id`. Upon synchronization:

* the serializer accepts the temporary identifier
* the backend assigns the authoritative primary key
* the server timestamp replaces client‑supplied time

Conflict resolution follows a **Last‑Write‑Wins** strategy enforced by the server clock. CRDT‑based approaches were intentionally rejected due to cost, complexity, and reduced audit transparency under MVP constraints.

---

## 7. Service Layer: Business Authority

The Service Layer is the **operational brain of the system** and the only location where business rules are allowed to exist.

### 7.1 Responsibilities

* inventory mutation
* billing calculations
* lifecycle transitions
* orchestration of side effects (documents, notifications, audit)

Views delegate. Serializers validate. **Services decide.**

### 7.2 Atomic Integrity

All commit operations are wrapped in `@transaction.atomic` and use `SELECT ... FOR UPDATE` to lock affected inventory rows. This guarantees that:

* stock levels cannot go negative
* concurrent claims are serialized
* partial commits are impossible

---

## 8. API Layer: Intentional State Transitions

Standard CRUD endpoints exist strictly for drafting and review.

Irreversible actions are gated behind **intent‑based endpoints**:

```
POST /api/v1/transactions/{id}/confirm/
```

Only confirm endpoints may:

* lock inventory
* finalize billing
* generate invoices
* write audit records

This design prevents accidental taps, malicious manipulation, and unintended state corruption.

---

## 9. Observability & Audit Trail

Every request passes through dedicated logging middleware that captures:

* user identity
* action type
* entity affected
* request payload
* authoritative timestamp

Audit records are append‑only and read‑only, even to administrators. They are treated as **forensic artifacts**, not operational logs, ensuring integrity during dispute resolution or regulatory review.

---

## 10. Automated Invoicing & Hardware Boundary

Within a confirmed transaction:

1. HTML is rendered to PDF via WeasyPrint
2. PDF is dispatched via SMTP
3. ESC/POS payload is generated

The backend produces **facts and payloads**. The frontend manages **physical hardware interaction** (Web Bluetooth). This separation preserves portability, testability, and long‑term maintainability.

---

## 11. Offline Sync & Conflict Strategy

* Offline queueing is mandatory
* Server timestamps are authoritative
* Last‑Write‑Wins is applied consistently

This strategy optimizes for reliability, performance, and audit clarity within MVP budget and timeline constraints.

---

## 12. Implementation Roadmap (16 Weeks)

1. **Foundation** — Schema finalization and API contracts
2. **Core Services** — Authentication, inventory, audit
3. **Distribution Logic** — Atomic logistics flows
4. **Transaction & Billing** — Pricing, invoicing, printing
5. **Sync & Refinement** — Offline reconciliation
6. **Deployment** — UAT and cloud handover

---

## 13. Architectural Takeaways

1. Physical assets demand backend discipline
2. State transitions must be explicit and intentional
3. Offline capability is a core system property

The HSH Sales System backend demonstrates how disciplined layering, atomic enforcement, and intent‑driven APIs can safely digitize high‑risk, real‑world logistics under real field constraints.
