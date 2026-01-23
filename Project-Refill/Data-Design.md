# **Data Design Document (DDD)**

**Version:** 1.0
**Date:** 23 January 2026
**Author:** Sean Wong

---

## **1. Introduction**

### **1.1 Purpose**

This document defines the **data architecture, database design, and operational flows** for the Singapore Field Operations MVP. It covers:

* LPG **inventory management**
* **Customer transactions** and billing
* **Distribution logistics**
* **Invoicing and PDF generation**
* **Audit and compliance logging**

The goal is a **compliant, reliable, and scalable digital logistics platform** enabling precise tracking of **physical stock**, financial transactions, and regulatory compliance.

### **1.2 Scope**

<img width="1536" height="1024" alt="image" src="https://github.com/user-attachments/assets/d2539ab1-81ef-4313-bd89-3ed406649e6d" />

The system encompasses:

* Depot operations (inventory and equipment management)
* Distribution tracking (Depot → Customer / Depot)
* Customer site transactions and invoicing
* Audit and compliance logging
* Role-based access control via **Accounts**

---

## **2. Architectural Overview**

### **2.1 Technology Stack**

| Layer          | Technology / Library                           |
| -------------- | ---------------------------------------------- |
| Backend        | Django 6 + Django REST Framework (DRF 3.16+)   |
| Database       | MySQL (Production), SQLite (Prototype)         |
| PDF Generation | WeasyPrint                                     |
| Authentication | JWT (Simple JWT)                               |
| Logging        | Middleware capturing requests & audit events   |
| Atomicity      | Database transactions ensuring ACID compliance |

### **2.2 Design Principles**

1. **Data Integrity:** ACID-compliant operations for inventory, transactions, and invoicing.
2. **Compliance-Ready:** Gapless invoice numbering; audit trail for all actions.
3. **Scalable:** Asynchronous Django supports multiple drivers and high concurrency.
4. **Secure:** JWT authentication, role-based access, and tamper-proof audit logs.
5. **Separation of Concerns:** Modularized inventory, distribution, transactions, invoicing, and audit services.

---

## **3. Core Entities**

| Entity           | PK / FK                                                       | Attributes / Notes                                                                                                                                                       |
| ---------------- | ------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| **Account**      | `id` (PK)                                                     | Represents a **system user** (Admin / Driver). Stores email (unique), password, role, assigned depot (optional), JWT token. **Accounts drive all operational actions**.  |
| **Depot**        | `id` (PK), `admin_id` (FK → Account.id)                       | Physical storage location. Admin account manages depot.                                                                                                                  |
| **Equipment**    | `id` (PK, UUID), `depot_id` (FK → Depot.id)                   | Non-consumable assets (meters, cylinders, regulators). Tracks status (Active/Maintenance). Linked to inventory for tracking availability and usage.                      |
| **Customer**     | `id` (PK)                                                     | Receives LPG and equipment. Stores name, site address, email, payment type, meter info.                                                                                  |
| **Inventory**    | `id` (PK), `depot_id` / `customer_id` (FK)                    | Tracks stock quantities per `(entity, equipment)` pair. Updated via distribution, transactions, or manual adjustments.                                                   |
| **Distribution** | `id` (PK), `driver_id` (FK → Account.id)                      | Represents physical movement of stock from depot → customer/depot. Contains loadout (JSON) and confirmation timestamp. Updates inventory on both source and destination. |
| **Transaction**  | `id` (PK), `customer_id` (FK → Customer.id)                   | Represents a **sale or consumption event at a customer site**. Updates inventory at customer location, triggers invoice generation.                                      |
| **Invoice**      | `id` (PK), `transaction_id` (FK → Transaction.id, One-to-One) | Auto-generated PDF invoice, invoice number, and status. Ensures **financial auditability**.                                                                              |
| **Sequence**     | `id` (PK)                                                     | Ensures gapless invoice numbering for compliance.                                                                                                                        |
| **Audit**        | `id` (PK), `user_id` (FK → Account.id)                        | Logs all user actions, request metadata, IP, and timestamp.                                                                                                              |

---

### **3.1 Account – Role and Function**

The **Account entity** represents **system users**, including:

* **Admin / Depot Manager** – Manages depot, equipment, inventory, and approves distributions.
* **Driver / Field Staff** – Executes distributions, posts customer transactions, confirms deliveries.

**Key Responsibilities of Account:**

* Authenticates via JWT for secure API access.
* Creates and confirms **Distributions** → affects depot and customer inventory.
* Posts **Transactions** → triggers **Invoice** creation and updates inventory.
* All actions logged in **Audit** entity.

> Accounts are the **actors** driving all physical and financial movements in the system.

---

### **3.2 Equipment – Role in Operations**

* **Non-consumable asset tracking:** Meters, cylinders, regulators, tools.
* **Linked to Inventory:** Enables accurate stock and availability tracking.
* **Used in Distributions and Transactions:** Equipment is assigned, moved, or consumed.
* **Status tracking:** Active, Maintenance, or Retired.

---

### **3.3 Transaction – Role in Operations**

* Represents **a sale or LPG usage event** at a customer site.
* **Consumes Inventory:** Reduces quantity at the customer location.
* **Triggers Invoice Generation:** Each transaction creates one invoice.
* Does **not** physically move stock — the physical movement is handled by Distribution.

---

### **3.4 Invoice – Role in Operations**

* Linked **one-to-one** with a Transaction.
* Contains PDF path, invoice number, status (draft/finalized/emailed).
* Ensures **regulatory compliance** with gapless sequences.
* Can be automatically emailed to customers.

---

## **4. Relationships & Constraints**

| Origin Entity         | Target Entity | Type     | Notes                                   |
| --------------------- | ------------- | -------- | --------------------------------------- |
| Account (Depot Admin) | Depot         | 1 → Many | Protect critical links                  |
| Depot                 | Inventory     | 1 → Many | Tracks depot stock                      |
| Depot                 | Equipment     | 1 → Many | Non-consumable assets                   |
| Account (Driver)      | Distribution  | 1 → Many | Driver executes distributions           |
| Customer              | Transaction   | 1 → Many | Customer can have multiple transactions |
| Inventory             | Transaction   | 1 → Many | Tracks inventory consumption            |
| Transaction           | Invoice       | 1 → 1    | Automatic invoice creation              |
| Account               | Audit         | 1 → Many | Logs all user actions                   |
| Sequence              | Invoice       | 1 → Many | Ensures gapless numbering               |

**Integrity Rules:**

* `PROTECT` for critical references
* `SET_NULL` for optional references
* `CASCADE` for dependent records that do not violate compliance

---

## **5. Data Flow & Lifecycle**

1. **Account Login:** JWT authentication for driver/admin.
2. **Depot Initialization:** Admin updates depot inventory and equipment master data.
3. **Distribution Assignment:** Driver account receives load; source depot inventory decremented.
4. **Customer Site Transaction:**

   * Inventory updated (customer inventory increased / consumed)
   * Invoice generated (WeasyPrint PDF, sequence assigned)
   * Audit log created linking action to Account
5. **Atomic Transactions:** Steps 3–4 are transactional to prevent inconsistencies.

---

## **6. Security & Compliance**

* **JWT Authentication:** Role-based access (Admin vs Driver).
* **Audit Logging:** Middleware logs request metadata, Account, IP, and timestamp.
* **Gapless Invoice Sequences:** Compliance-ready numbering.
* **Referential Integrity:** Protects financial and operational data.

---

## **7. ASCII ERD – Accounts, Transactions, Invoice, and Equipment**

```
+---------------------+       1     *       +-----------------+
|       Account       |-------------------->|      Depot      |
|---------------------|                     |-----------------|
| PK id               |                     | PK id           |
| email               |                     | name            |
| password            |                     | coordinates     |
| role (Admin/Driver) |                     | FK admin_id ----|
| jwt_token           |                     +-----------------+
+---------------------+                       |
       | 1                                     | 1
       |                                       |
       *                                       *
+---------------------+                   +-----------------+
|     Distribution    |                   |    Inventory    |
|---------------------|                   |-----------------|
| PK id               |                   | PK id           |
| FK driver_id ------ |                   | quantity        |
| loadout (JSON)      |                   | FK depot_id --- |
| confirmed_at        |                   | FK customer_id -|
+---------------------+                   | FK equipment_id |
                                           +-----------------+
                                                |
                                                | 1
                                                |
                                                * 
                                          +-----------------+
                                          |  Transaction    |
                                          |-----------------|
                                          | PK id           |
                                          | datetime        |
                                          | amount          |
                                          | FK customer_id -|
                                          | FK inventory_id |
                                          +-----------------+
                                                  |
                                                  | 1
                                                  |
                                          +-----------------+
                                          |     Invoice     |
                                          |-----------------|
                                          | PK id           |
                                          | invoice_no      |
                                          | pdf_path        |
                                          | is_finalized    |
                                          | FK transaction_id|
                                          +-----------------+

+---------------------+       1     *       +-----------------+
|      Customer       |-------------------->|  Transaction    |
+---------------------+                     +-----------------+

+---------------------+       1     *       +-----------------+
|      Account        |-------------------->|      Audit      |
+---------------------+                     +-----------------+

+---------------------+       1     * 
|      Sequence       |-------------------->|     Invoice     |
+---------------------+                     +-----------------+

+---------------------+       1     *       +-----------------+
|     Equipment       |-------------------->|    Inventory    |
+---------------------+                     +-----------------+
```

---

## **8. Sequence Diagram – LPG Delivery & Transaction Flow**

```
Driver/User           API Server           Inventory Service        Invoice Service        Audit Service
     |                    |                        |                     |                     |
     |  1. Login (JWT)    |                        |                     |                     |
     |------------------->|                        |                     |                     |
     |                    |  2. Authenticate JWT   |                     |                     |
     |                    |----------------------->|                     |                     |
     |                    |<-----------------------|                     |                     |
     |<-------------------|                        |                     |                     |
     |                    |                        |                     |                     |
     |  3. Create Distribution Record             |                     |                     |
     |------------------->|                        |                     |                     |
     |                    |  4. Assign driver &    |                     |                     |
     |                    |     decrement Inventory|                     |                     |
     |                    |----------------------->|                     |                     |
     |                    |<-----------------------|                     |                     |
     |                    |                        |                     |                     |
     |  5. Post Transaction at Customer Site       |                     |                     |
     |------------------->|                        |                     |                     |
     |                    |  6. Update Inventory   |                     |                     |
     |                    |----------------------->|                     |                     |
     |                    |<-----------------------|                     |                     |
     |                    |                        |                     |                     |
     |                    |  7. Generate Invoice   |                     |                     |
     |                    |----------------------->|  (WeasyPrint)      |                     |
     |                    |                        |-------------------->|                     |
     |                    |                        |<-------------------|                     |
     |                    |<-----------------------|                     |                     |
     |                    |                        |                     |                     |
     |                    |  8. Log Audit Event    |                     |                     |
     |                    |----------------------->|                     |                     |
     |                    |                        |                     |<-------------------|
     |                    |<-----------------------|                     |                     |
     |  9. Return Transaction & Invoice Status    |                     |                     |
     |<-------------------|                        |                     |                     |
```

**Highlights:**

* **Accounts** drive all operations (login, distributions, transactions).
* **Equipment** tracked via Inventory for both depots and customers.
* **Transactions** record sales / usage, consume inventory, trigger invoices.
* **Invoices** ensure financial auditability and compliance.
* **Audit logs** capture all actions by Accounts.
* **Atomic Transactions** ensure consistency across inventory, transaction, and invoice updates.

---

## **9. Production Readiness Highlights**

* Audit-ready, atomic, and secure
* Scalable for multiple drivers
* Regulatory compliance: gapless invoice numbers, immutable logs
* Clear accountability: all actions tied to Accounts
* Modularized flows: Inventory, Distribution, Transaction, Invoice, Audit

---

## **10. Distribution, Inventory, Customer, Depot, Account, Transaction, Invoice, and Equipment Relationship**

| Concept      | Role                       | Links to Inventory          | Triggered By                                             |
| ------------ | -------------------------- | --------------------------- | -------------------------------------------------------- |
| Depot        | Source of stock            | Holds stock                 | Distribution creation / adjustment                       |
| Customer     | Receives stock             | Holds stock                 | Distribution arrival / Transaction                       |
| Inventory    | Tracks quantities          | For depot or customer       | Adjusted by Distribution, Transaction, or manual update  |
| Distribution | Physical transfer          | Updates inventory           | Created/confirmed by driver (Account)                    |
| Transaction  | Usage / sale event         | Consumes customer inventory | Customer site event → triggers invoice                   |
| Invoice      | Billing / financial record | Represents financial record | Created per transaction, linked to sequence              |
| Account      | System user / actor        | Controls & executes actions | Logs audit, creates distributions, posts transactions    |
| Equipment    | Non-consumable asset       | Linked to inventory         | Assigned to depot, moved via distribution, consumed/used |

**Flow Conceptual Diagram:**

```
Depot (source) ------Distribution------> Customer (destination)
        |                                      |
     Inventory                                Inventory
        |                                      |
       Reduce                                  Increase
        |
Transaction at customer site (Account triggers)
        |
     Invoice generated & emailed
        |
   Audit log captured (linked to Account)
        |
   Equipment tracked & status updated if used
```

---

