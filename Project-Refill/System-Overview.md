# HSH Sales System (2026)

## Backend Architectural Blueprint & Implementation Guide

---

## 1. Architectural Philosophy: Backend as Physical Truth

In the LPG distribution domain, software does not merely *record* reality — it **defines accountability for physical assets**. Every cylinder moved, delivered, returned, or billed has direct financial and regulatory consequences. For this reason, the HSH Sales System is designed with a single uncompromising mandate:

> **The backend is the ultimate source of truth for all physical and financial reality.**

### 1.1 Field‑First Is Not a UX Choice — It Is a Risk Model

Delivery drivers and sales staff operate in depots, back alleys, industrial estates, and customer sites where connectivity is intermittent or unreliable. Designing for “always online” would introduce silent failure modes:

* missed inventory updates
* duplicate billing
* untraceable cylinder leakage

The system therefore assumes **offline-first behavior by default** and treats network availability as an optimization, not a dependency. Digital records must be capturable immediately in the field and reconciled deterministically once connectivity resumes.

### 1.2 Why Backend Discipline Matters in LPG Operations

Unlike purely digital products, LPG systems must reconcile:

* high‑value reusable physical assets
* regulatory audit expectations
* real‑time billing obligations
* human-operated workflows under pressure

Any ambiguity between physical movement and digital state is unacceptable. The backend architecture is therefore intentionally conservative, explicit, and auditable.

---

## 2. Technical Foundation & Stack Rationale

### 2.1 Core Stack (2026)

The technology stack is selected for **industrial-grade reliability**, long-term maintainability, and predictable behavior under load.

**Runtime & Framework**

* Python 3.12
* Django 6 (LTS)
* Django REST Framework (DRF) 3.15+

**Persistence & Integrity**

* MySQL 8 (production)
* SQLite (local development)

**Infrastructure Services**

* JWT Authentication (`djangorestframework-simplejwt`)
* OpenAPI / Swagger (`drf-spectacular`)
* WeasyPrint (HTML → PDF)
* SMTP for invoice delivery

### 2.2 Asynchronous Capability as a Strategic Choice

Django 6’s async support is leveraged to safely handle:

* concurrent sync bursts from multiple field devices
* high-latency mobile networks
* long-running I/O tasks (PDF generation, email dispatch)

Async is used **selectively** — never to weaken transactional guarantees — but to ensure the system remains responsive under field load.

### 2.3 Foundational Architectural Principles

| Principle                | Technical Enforcement                                     | Strategic Outcome                                       |
| ------------------------ | --------------------------------------------------------- | ------------------------------------------------------- |
| Resource-Oriented Design | RESTful entities (Customers, Transactions, Distributions) | Predictable APIs, parallel frontend/backend development |
| Explicit Versioning      | `/api/v1/` URI strategy                                   | Safe evolution without breaking deployed field devices  |
| JWT Authentication       | Short-lived access tokens + refresh                       | Secure mobile usage without session fragility           |
| Idempotency              | Human-readable unique numbers (TRX / DIST / INV)          | Prevents duplicates, aligns with paper receipts         |

This foundation ensures the backend is not a thin database wrapper, but a **resilient execution engine** for LPG operations.

---

## 3. Layered Backend Architecture

The HSH Sales System follows a **strict layered architecture** to prevent logic leakage, duplication, and unauthorized state transitions.

### 3.1 Layer Overview

```
Client (Mobile / Web)
        ↓
API Layer (ViewSets & Routers)
        ↓
Serializer Layer (Validation & Shaping)
        ↓
Service Layer (Business Rules)
        ↓
Domain Models (State)
        ↓
Database (ACID, Locked on Commit)
```

Each layer has a **non-negotiable responsibility boundary**.

---

## 4. Structural Overview: Modular Django Applications

To ensure long-term maintainability and blast-radius containment, the backend enforces a **multi-app Django structure**, where each app maps to a real operational domain.

### 4.1 Core Applications

* **accounts/**
  User identities, roles (Sales vs Admin), and JWT lifecycle management.

* **depots/** & **equipment/**
  Authoritative registry of physical assets, including Singapore LPG variants:
  CYL 9 kg, 12.7 kg, 14 kg, 50 kg (POL), 50 kg (L).

* **distribution/**
  Internal logistics engine governing Depot ↔ Truck movements. Enforces strict movement types:
  *Collection (Full Out)* and *Empty Return (Empty In)*.

* **transactions/** & **invoices/**
  The financial engine. Manages usage billing, cylinder charges, service items, payments, and invoice lifecycle.

* **inventory/**
  Real-time stock counters for depots, trucks, and customer sites.

* **audit/**
  Immutable append-only action log for full traceability.

This structure ensures that logistics logic cannot silently corrupt billing, and billing logic cannot bypass inventory enforcement.

---

## 5. Middleware & Observability Layer

A dedicated middleware layer (e.g. `request_logging.py`) captures:

* request metadata
* authenticated user
* endpoint invoked
* response status
* correlation identifiers

This is essential for:

* resolving billing disputes
* diagnosing sync anomalies
* satisfying audit and compliance reviews

Observability is treated as a **first-class architectural concern**, not an afterthought.

---

## 6. The Model Layer: Authoritative State

Domain models represent **committed reality only**.

### 6.1 Header / Line Item Pattern

Key entities such as `DistributionHeader` and `TransactionHeader`:

* generate human-readable unique numbers at commit time
* own their lifecycle status
* aggregate immutable line items

### 6.2 Payment Semantics

`TransactionHeader.payment_received` drives downstream behavior:

* `True` → Receipt generated
* `False` → Invoice marked *Unpaid* in audit trail

This prevents ambiguous financial states.

---

## 7. The Service Layer: Business Rule Enforcement

The Service Layer (e.g. `transactions/services.py`) is the **brain of the system**.

### 7.1 Why Services Exist

Business rules **must never** live in:

* serializers
* views
* frontend logic

All critical logic — especially billing and inventory mutation — is centralized in services to ensure:

* testability
* reusability
* impossibility of bypass

### 7.2 Example: Meter Billing Logic

```
Usage = Current Reading − Last Reading
```

This rule exists **once**, in one service, and is invoked only during confirmation.

---

## 8. API Layer: Controlled Exposure

### 8.1 Standard CRUD

CRUD endpoints exist for drafting and viewing data, but **never finalize state**.

### 8.2 Intent-Based Endpoints

Critical irreversible actions are gated behind explicit intent endpoints:

```
POST /api/v1/transactions/{id}/confirm/
```

Only this endpoint may:

* lock inventory rows
* finalize billing
* generate invoices
* write audit records

This prevents accidental or malicious state manipulation.

---

## 9. Atomicity & Concurrency Control

### 9.1 Atomic Commit Strategy

All confirm endpoints are wrapped in `@transaction.atomic`.

Within the atomic block:

* inventory counters update
* invoice generation executes
* audit records persist

If **any step fails**, the entire operation rolls back.

### 9.2 Row-Level Locking

```
SELECT ... FOR UPDATE
```

Applied **only** during confirmation to:

* prevent race conditions
* preserve throughput for browsing operations

---

## 10. Offline Sync & Reconciliation Layer

### 10.1 Client-Side Temporary Identity

When offline:

* frontend generates `client_temp_id`
* records queued locally

### 10.2 Server Reconciliation

Upon reconnection:

* backend assigns authoritative ID
* server timestamp replaces client time

### 10.3 Conflict Strategy

* **Last-Write-Wins**, enforced by server clock

A CRDT-based approach was explicitly rejected as non-viable for MVP cost and complexity constraints.

---

## 11. Automated Invoicing & Hardware Boundary

### 11.1 Invoicing Pipeline (Atomic)

1. HTML rendered to PDF via WeasyPrint
2. PDF emailed via SMTP
3. ESC/POS payload generated

### 11.2 Hardware Separation

* Backend produces print payloads
* Frontend manages Web Bluetooth & printer connection

This keeps the backend **hardware-agnostic and portable**.

---

## 12. Implementation Roadmap (16 Weeks)

1. **Foundation (Weeks 1–1.5)** – Schema & API contracts
2. **Core Services (Weeks 2–5.5)** – Auth, inventory, audit
3. **Distribution Logic (Weeks 4–8.5)** – Atomic logistics flows
4. **Transaction & Billing (Weeks 7–11.5)** – Billing & printing
5. **Sync & Refinement (Weeks 10–14)** – Offline reconciliation
6. **Deployment (Weeks 14–16)** – UAT & cloud handover

---

## 13. Strategic Takeaways

1. **Physical assets demand backend discipline** — shortcuts create financial risk.
2. **State transitions must be intentional** — confirm endpoints are sacred.
3. **Offline is a core feature** — not a future enhancement.

The HSH Sales System backend demonstrates how layered architecture, strict atomicity, and explicit intent modeling can safely digitize high-risk, real-world operations under real field constraints.
