# **Data Design Document (DDD)**

**Version:** 1.1
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

The goal is a **compliant, reliable, and scalable digital logistics platform**, enabling precise tracking of **physical stock**, financial transactions, and regulatory compliance.

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

1. **Data Integrity:** ACID-compliant operations for inventory, transactions, and invoicing
2. **Compliance-Ready:** Gapless invoice numbering; audit trail for all actions
3. **Scalable:** Asynchronous Django supports multiple drivers and high concurrency
4. **Secure:** JWT authentication, role-based access, and tamper-proof audit logs
5. **Separation of Concerns:** Modularized inventory, distribution, transactions, invoicing, and audit services

---

## **3. Core Entities**

| Entity           | PK / FK                                                       | Attributes / Notes                                                                                                                                                       |
| ---------------- | ------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| **Account**      | `id` (PK)                                                     | Represents a **system user** (Admin / Driver). Stores email (unique), password, role, assigned depot (optional), JWT token. **Accounts drive all operational actions**.  |
| **Depot**        | `id` (PK), `admin_id` (FK → Account.id)                       | Physical storage location. Admin account manages depot.                                                                                                                  |
| **Equipment**    | `id` (PK, UUID), `depot_id` (FK → Depot.id)                   | Non-consumable assets (meters, cylinders, regulators). Tracks status (Active/Maintenance). Linked to inventory for stock and usage tracking.                             |
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

* **Admin / Depot Manager** – Manages depot, equipment, inventory, and approves distributions
* **Driver / Field Staff** – Executes distributions, posts customer transactions, confirms deliveries

**Key Responsibilities of Account:**

* Authenticates via JWT for secure API access
* Creates and confirms **Distributions** → affects depot and customer inventory
* Posts **Transactions** → triggers **Invoice** creation and updates inventory
* All actions logged in **Audit** entity

> Accounts are the **actors** driving all physical and financial movements in the system.

---

### **3.2 Equipment – Role in Operations**

* Tracks **non-consumable assets** such as meters, cylinders, and regulators
* **Linked to Inventory** for accurate stock and usage tracking
* **Used in Distributions and Transactions:** equipment is assigned, moved, or consumed
* **Status tracking:** Active, Maintenance, or Retired

---

### **3.3 Transaction – Role in Operations**

* Represents **a sale or LPG usage event** at a customer site
* **Consumes Inventory:** reduces quantity at customer location
* **Triggers Invoice Generation:** each transaction creates one invoice
* Does **not** physically move stock — the physical movement is handled by Distribution

---

### **3.4 Invoice – Role in Operations**

* Linked **one-to-one** with a Transaction
* Contains PDF path, invoice number, status (draft/finalized/emailed)
* Ensures **regulatory compliance** with gapless sequences
* Can be automatically emailed to customers

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

1. **Account Login:** JWT authentication for driver/admin
2. **Depot Initialization:** Admin updates depot inventory and equipment master data
3. **Distribution Assignment:** Driver receives load; source depot inventory decremented
4. **Customer Site Transaction:**

   * Inventory updated (customer inventory increased/consumed)
   * Invoice generated (WeasyPrint PDF, sequence assigned)
   * Audit log created linking action to Account
5. **Atomic Transactions:** Steps 3–4 are transactional to prevent inconsistencies

---

## **6. Security & Compliance**

* **JWT Authentication:** Role-based access (Admin vs Driver)
* **Audit Logging:** Middleware logs request metadata, Account, IP, timestamp
* **Gapless Invoice Sequences:** Compliance-ready numbering
* **Referential Integrity:** Protects financial and operational data

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

---

## **9. Production Readiness Highlights**

* Audit-ready, atomic, and secure
* Scalable for multiple drivers
* Regulatory compliance: gapless invoice numbers, immutable logs
* Clear accountability: all actions tied to Accounts
* Modularized flows: Inventory, Distribution, Transaction, Invoice, Audit

---

## **10. Distribution Payload Explanation**

Distributions represent **physical movements of stock** from a depot to a customer or another depot. The payload structure ensures:

* Clear tracking of **equipment types, quantities, and conditions**
* Accurate **inventory updates** on both source and destination
* Logging for **audit and compliance**

**Example JSON Payload:**

```json
{
    "depot": {{DEPOT_ID}},
    "remarks": "Updated remarks",
    "items": [
        {
            "equipment": {{EQUIPMENT_ID}},
            "direction": "OUT",
            "condition": "FULL",
            "quantity": 10
        }
    ]
}
```

**Field Explanation:**

| Field       | Type    | Description                                                           |
| ----------- | ------- | --------------------------------------------------------------------- |
| `depot`     | integer | ID of source depot performing the distribution                        |
| `remarks`   | string  | Optional notes for driver or audit logs                               |
| `items`     | array   | List of stock movements                                               |
| `equipment` | integer | Equipment ID (FK → Equipment.id)                                      |
| `direction` | string  | `"OUT"` for dispatch from depot, `"IN"` for receipt at depot/customer |
| `condition` | string  | `"FULL"`, `"EMPTY"`, `"MAINTENANCE"` – status of the stock/equipment  |
| `quantity`  | integer | Number of units being moved                                           |

**Operational Notes:**

* **Inventory Update:**

  * `OUT` → decreases source depot stock
  * `IN` → increases destination (customer or depot) stock
* **Audit Logging:** Captures full JSON payload linked to driver account
* **Atomicity:** Distribution applied as a transaction → no partial stock updates

**Integration with Core Flows:**

1. Driver posts distribution → API validates items and depot
2. System updates depot (`OUT`) and customer (`IN`) inventory atomically
3. Audit log created for traceability
4. Distribution confirmation timestamp recorded

---

## **11. Conceptual Flow – Distribution Example**

```
Depot (source) ------Distribution (OUT)------> Customer (destination)
        |                                          |
     Inventory                                   Inventory
   decrease stock                                increase stock
        |
  Equipment assigned & tracked
        |
   Audit log created
        |
   Remarks captured for driver & compliance
```

Ensures **physical stock movement, inventory changes, and audit records are all synchronized**, keeping operations traceable, compliant, and atomic.

---

## **12. Distribution Payload Flow – Visual Diagram**

**Flow of a Distribution JSON payload through the system:**

```
         +----------------+
         | Driver / Account|
         +--------+-------+
                  |
                  | POST /distributions
                  | JSON Payload:
                  | {
                  |   "depot": DEPOT_ID,
                  |   "remarks": "...",
                  |   "items": [...]
                  | }
                  v
         +----------------+
         | API Server      |
         | (DRF Viewset)   |
         +--------+-------+
                  |
        Validate payload, depot, and equipment
                  |
                  v
         +----------------+
         | Distribution    |
         | Service / Model |
         +--------+-------+
                  |
                  | 1. Update depot inventory (OUT)
                  | 2. Update customer inventory (IN)
                  | 3. Save distribution record
                  |
                  v
     +------------+------------+
     | Inventory Service       |
     | (Depot / Customer Stock)|
     +------------+------------+
                  |
                  | Atomic transaction ensures consistency
                  v
         +----------------+
         | Audit Service  |
         +--------+-------+
                  |
                  | Log driver, depot, items, remarks, timestamp
                  v
         +----------------+
         | Audit Record   |
         +----------------+
                  |
                  | Optional: triggers downstream processes
                  v
         +----------------+
         | Invoice Service|
         +----------------+
                  |
                  | If items are customer sales, generate Invoice
                  | Update Sequence for gapless numbering
                  v
         +----------------+
         | Invoice Record |
         +----------------+
```

**Highlights of this flow:**

1. Driver submits **Distribution JSON** representing stock movement
2. **API Server** validates depot, equipment, and quantities
3. **Distribution Service** updates inventory atomically (`OUT` / `IN`)
4. **Audit Service** logs all actions for compliance and traceability
5. **Invoice Service** triggered for customer sales → invoice generated
6. **Atomic Transactions** ensure inventory, audit, and invoices remain consistent

**Integration with Existing DDD Concepts:**

*Distribution JSON → Distribution Model → Inventory → Audit → Invoice*

---

