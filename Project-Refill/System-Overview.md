# **HSH Sales System Overview (2026)**

This document provides a **comprehensive overview of the HSH Sales System**, a field-first LPG (Liquefied Petroleum Gas) sales and logistics solution for Singapore operations. It synthesizes system design, backend implementation, and frontend requirements into a technical reference for developers and architects.

---

## **1️⃣ System Design and Architecture**

The HSH Sales System addresses the challenges of **real-time inventory tracking and immediate billing** in the field. It emphasizes:

* Fast data entry
* Physical and digital proof of sale
* Precise cylinder tracking

### Core Philosophy and Workflows

| Feature              | Distribution Workflow             | Transaction Workflow                          |
| -------------------- | --------------------------------- | --------------------------------------------- |
| Purpose              | Logistics: Depot ↔ Truck movement | Sales: Delivery and billing to customers      |
| Customer Involvement | No                                | Yes                                           |
| Invoice Generation   | No                                | Yes (Usage, Delivery, Services)               |
| Cylinder Tracking    | Depot inventory levels            | Client site inventory levels                  |
| Output               | Optional receipt                  | Thermal printed invoice + automated PDF email |

### Architectural Principles

* **Mobile-First Design:** Optimized for tablet and mobile use with minimal typing; mandatory human confirmation steps.
* **Atomic Integrity:** Critical operations (e.g., confirming a delivery) are wrapped in atomic transactions to ensure **inventory updates, invoice generation, and audit logging all succeed or fail together**.
* **Offline Tolerance:** Supports operations in low-connectivity areas using a **last-write-wins** strategy with **client-generated temporary IDs**.
* **Traceability:** Every action is linked to **user + timestamp + unique number** (`TRX-…`, `DIST-…`, `INV-…`) and recorded in an **audit log**.

---

## **2️⃣ Backend Architecture and Implementation**

The backend is built on **Django 5.1** + **Django REST Framework (DRF)**, employing a **resource-oriented API**.

### Technical Stack

* **Framework:** Django 5.1.4 (LTS) + DRF 3.15+
* **Database:** MySQL 8 (production), SQLite (development)
* **Authentication:** JWT via `djangorestframework-simplejwt`
* **Documentation:** OpenAPI/Swagger via `drf-spectacular`
* **Utilities:** WeasyPrint (HTML → PDF invoices), Django SMTP (automated emailing)

### API Design Principles

1. **Versioning:** URI-based (`/api/v1/`) ensures backward compatibility
2. **Resource-Oriented:** Each entity (customers, transactions, inventory) accessible via standard REST conventions
3. **Idempotency:** Unique transaction/distribution numbers prevent duplicate processing
4. **Offline Sync Support:** Creation endpoints accept `client_temp_id`, server returns official ID + timestamp
5. **Filtering & Pagination:** Standardized list views with date ranges and entity filters, e.g., `?customer=3&date_from=2026-01-01`

### Core Data Models

* **Accounts/Users:** JWT-authenticated; roles include Sales and Admin
* **Depots & Equipment:** Storage locations and cylinder types (9kg, 12.7kg, 14kg, 50kg)
* **Inventory:** Tracks stock at Depots and Client sites
* **Transactions/Invoices:** Captures sales data, meter readings, and service fees; invoice lifecycle: **Generated → Printed → Emailed → Paid**

---

## **3️⃣ Frontend Architecture and Field Implementation**

The frontend is a **React 19 application** designed for **responsive, field-ready performance**.

### Technical Stack

* **Build Tool:** Vite 6.x
* **State Management:** Zustand 5.x (global state, offline queue)
* **Data Fetching:** TanStack Query 5.x (caching, offline sync)
* **Form Management:** React Hook Form + Zod (type-safe validation)
* **Styling:** TailwindCSS 4.x (mobile-first)

### Field-Specific Features

* **Dynamic Entry Tables:** Drivers can quickly add multiple items (Equipment + Quantity + Status)
* **Thermal Printing:** Web Bluetooth + ESC/POS protocols for mobile printer integration
* **Offline Queue:** Uses `localforage` (IndexedDB) to store transactions offline; auto-sync when online
* **Confirmation Dialogs:** Summarize totals before committing (e.g., Total Collection vs. Total Empty Return)

---

## **4️⃣ Comprehensive Glossary**

| Term               | Definition                                                                            |
| ------------------ | ------------------------------------------------------------------------------------- |
| Atomic Transaction | Ensures all steps (inventory update, log creation, invoice save) complete or rollback |
| Collection         | In Distribution workflow, taking full cylinders from depot                            |
| Empty Return       | Returning empty cylinders to depot or customer site                                   |
| client_temp_id     | Unique offline ID generated by mobile app for later reconciliation                    |
| Usage Billing      | Cost = Current Meter Reading - Last Meter Reading                                     |
| Service Items      | Non-cylinder billables (hose clips, regulators, repair fees)                          |
| ESC/POS            | Standard printer command language for thermal receipts                                |
| JWT                | JSON Web Token for secure frontend-backend authentication                             |
| WeasyPrint         | Backend library converting HTML/CSS templates into PDF invoices                       |
| Last-Write-Wins    | Offline conflict resolution strategy preserving the most recent update                |

---

## **5️⃣ Study Quiz**

1. Which backend technology converts invoice templates into downloadable PDF files?
   A) Django SMTP B) WeasyPrint C) TanStack Query D) Zod

2. What is the primary difference between a "Distribution" and a "Transaction"?
   A) Distribution is for admins; Transaction is for sales staff
   B) Distribution involves customers; Transaction does not
   C) Transaction creates an invoice and tracks client inventory; Distribution tracks depot movements
   D) Distribution requires a meter reading; Transaction does not

3. How does the system handle offline data entry?
   A) App prevents any entry until online
   B) Data stored in an offline queue via localforage; auto-sync when online
   C) Driver must call admin to manually enter
   D) App uses SMS to send data

4. What is the purpose of `@transaction.atomic` in Django?
   A) Speed up API responses
   B) Ensure inventory integrity; rollback if process fails
   C) Auto-generate Swagger docs
   D) Encrypt JWT tokens

5. Which frontend library handles global state and the offline queue?
   A) React Hook Form B) TailwindCSS C) Zustand D) Axios

6. Correct invoice status lifecycle?
   A) Pending → Approved → Paid
   B) Generated → Printed → Emailed → Paid
   C) Draft → Sent → Collected
   D) Collection → Delivery → Return

---

## **6️⃣ Answer Key**

1. **B** – WeasyPrint
2. **C** – Transaction creates an invoice and tracks client inventory; Distribution tracks depot movements
3. **B** – Offline queue via localforage, auto-sync when online
4. **B** – Atomic decorator ensures rollback if any step fails
5. **C** – Zustand
6. **B** – Generated → Printed → Emailed → Paid


