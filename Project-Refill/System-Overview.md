# HSH Sales System (2026)

## Executive Summary

The **HSH Sales System** is a field-first LPG (Liquefied Petroleum Gas) sales and logistics platform designed for Singapore operations. It replaces manual processes with a **mobile-optimized, offline-tolerant, and fully traceable** digital workflow. The system is built around two operational pillars—**Distribution** (internal logistics) and **Transaction** (customer sales and billing)—and emphasizes atomic data integrity, immediate proof-of-sale, and precise cylinder tracking.

---

## 1. Design Philosophy

**Field-first by default.** The primary users are drivers and sales staff operating under intermittent connectivity.

**Core Objectives**

* **Speed & Safety:** Minimal typing, dynamic entry tables, mandatory confirmation screens.
* **Immediate Invoicing:** Thermal receipt printing and automated PDF emailing at the point of sale.
* **Total Traceability:** Every action linked to *User + Timestamp + Unique Number* (TRX, DIST, INV).
* **Data Integrity:** Digital records must always reflect physical cylinder movements.

---

## 2. Operational Workflows

### 2.1 Distribution vs Transaction

| Dimension            | Distribution             | Transaction                        |
| -------------------- | ------------------------ | ---------------------------------- |
| Purpose              | Logistics: Depot ↔ Truck | Sales: Customer Billing            |
| Customer Involvement | No                       | Yes                                |
| Invoice Generation   | No (optional receipt)    | Yes (PDF + thermal)                |
| Inventory Impact     | Depot Inventory          | Client Site Inventory              |
| Status Examples      | Collection, Empty Return | Deliver, Empty Return, Half Return |

### 2.2 Distribution (Internal Logistics)

* Moves cylinders between **Depot and Truck**.
* Inputs: Depot, Equipment Type, Quantity.
* Confirmation step summarizes totals before commit.
* Atomic save updates depot inventory and audit log.

### 2.3 Transaction (Customer Sales)

* Represents the **money flow**.
* Components:

  * **Usage Billing:** Meter-based (Latest − Last Reading).
  * **Cylinder Items:** Deliveries and returns at customer rates.
  * **Service Items:** Regulators, repairs, deposits, accessories.
* Invoice lifecycle: **Generated → Printed → Emailed → Paid**.

---

## 3. Technical Architecture

### 3.1 Stack (2026)

**Backend**

* Django 6 (LTS), Django REST Framework 3.15+
* MySQL 8 (production), SQLite (development)
* JWT authentication (`djangorestframework-simplejwt`)
* OpenAPI via `drf-spectacular`
* WeasyPrint (HTML → PDF), Django SMTP (email)

**Frontend**

* React 19 + Vite 6
* TailwindCSS 4
* Zustand (global state, offline queue)
* TanStack Query 5 (caching, sync)
* React Hook Form + Zod (type-safe validation)
* localforage (IndexedDB offline storage)

---

## 4. API Design Principles

### 4.1 Resource-Oriented Design

* Treats entities as **resources**, not procedures.
* Standard REST actions enable predictable integration and parallel development.

| Action          | Method    | Example                 |
| --------------- | --------- | ----------------------- |
| List / Retrieve | GET       | `/api/v1/customers/12/` |
| Create          | POST      | `/api/v1/transactions/` |
| Update          | PUT/PATCH | `/api/v1/invoices/34/`  |
| Delete          | DELETE    | Admin-only corrections  |

### 4.2 Intent-Based Exceptions

Used sparingly for complex side effects.

```
POST /api/v1/transactions/{id}/confirm/
```

Triggers atomic inventory updates, invoice generation, and audit logging.

### 4.3 Versioning & Security

* URI versioning: `/api/v1/`
* JWT: short-lived access token (memory), refresh token (secure storage).

---

## 5. Data Integrity & Reliability

### 5.1 Atomic Operations

Critical workflows are wrapped in `@transaction.atomic`.

* Inventory updates
* Invoice generation
* Audit logging

All succeed or roll back together.

### 5.2 Concurrency Control

* Row-level locking via `SELECT … FOR UPDATE`.
* Prevents race conditions on shared depot inventory.

### 5.3 Idempotency

* Human-readable unique numbers (TRX-…, DIST-…)
* Duplicate submissions are safely ignored.

---

## 6. Offline Support & Sync

### 6.1 Offline Creation

* Frontend generates `client_temp_id` when offline.
* Records stored locally (IndexedDB via localforage).

### 6.2 Reconciliation

* On sync, server returns official ID + timestamp.
* Client replaces temporary IDs automatically.

### 6.3 Conflict Resolution

* **Last-Write-Wins** using `last_modified` timestamp.
* Chosen for reliability and MVP delivery speed.

---

## 7. Hardware Integration

* **Thermal Printing:** 80mm mobile printers
* **Protocols:** Web Bluetooth + ESC/POS
* **Output:** Immediate physical proof-of-sale in the field

---

## 8. Core Resources & Endpoints

| Resource      | Key Endpoints        | Responsibility               |
| ------------- | -------------------- | ---------------------------- |
| Auth          | `/auth/login/`       | JWT issuance                 |
| Customers     | `/customers/`        | Client data & site inventory |
| Distributions | `/distributions/`    | Depot logistics              |
| Transactions  | `/transactions/`     | Sales & billing              |
| Inventory     | `/inventory/depots/` | Stock levels                 |
| Invoices      | `/invoices/`         | PDF & email                  |
| Audit         | `/audit/`            | Immutable action history     |

---

## 9. Equipment Types (Singapore LPG)

* CYL 9 kg
* CYL 12.7 kg
* CYL 14 kg
* CYL 50 kg (POL)
* CYL 50 kg (L)

---

## 10. Key Takeaways for Architects

1. **Nouns for data, verbs for intent** – resource-first APIs with explicit confirm actions.
2. **Atomicity is non-negotiable** when software represents physical assets.
3. **Design for bad connectivity** from day one using temp IDs, offline queues, and reconciliation.

The HSH Sales System demonstrates how strict architectural discipline enables fast, safe field operations while maintaining full business traceability.
