# **Singapore Field Operations MVP – Data Design Document (DDD)**

**Version:** 1.0
**Date:** 23 January 2026
**Author:** Sean Wong

---

## **1. Introduction**

### **1.1 Purpose**

This document defines the **data architecture and database design** for the Singapore Field Operations MVP. It governs:

* LPG inventory management
* Customer transactions
* Distribution tracking
* Invoicing
* Audit logging

The goal is a **compliant, scalable, and reliable digital logistics platform** for Singapore LPG operations.

### **1.2 Scope**

The system covers:

* Depot operations (inventory and equipment)
* Distribution logistics
* Customer transactions and invoicing
* Audit and compliance logging

---

## **2. Architectural Overview**

### **2.1 Technology Stack**

* **Backend:** Django 6 + DRF 3.15+
* **Database:** MySQL (production) / SQLite (prototype)
* **PDF Generation:** WeasyPrint
* **Authentication:** JWT (Simple JWT)
* **Logging:** Custom middleware for audit trails
* **Atomic Transactions:** Ensures consistency across inventory, transactions, and invoices

### **2.2 Design Principles**

1. **Data Integrity:** ACID-compliant operations.
2. **Compliance-Ready:** Audit logs and gapless invoice sequences.
3. **Scalable:** Asynchronous Django support for high concurrency.
4. **Secure:** JWT-secured endpoints and tamper-proof audit logs.

---

## **3. Core Entities**

| Entity           | PK / FK                                                                         | Attributes / Notes                                       |
| ---------------- | ------------------------------------------------------------------------------- | -------------------------------------------------------- |
| **Account**      | `id` (PK)                                                                       | email (unique), password, role (Admin/Driver), jwt_token |
| **Depot**        | `id` (PK), `admin_id` (FK → Account.id)                                         | name, coordinates (PointField)                           |
| **Equipment**    | `id` (PK, UUID), `depot_id` (FK → Depot.id)                                     | status (Active/Maintenance)                              |
| **Customer**     | `id` (PK)                                                                       | name, site_address, email                                |
| **Inventory**    | `id` (PK), `depot_id` (FK → Depot.id)                                           | quantity, cylinder_size                                  |
| **Distribution** | `id` (PK), `driver_id` (FK → Account.id)                                        | loadout (JSON), timestamp                                |
| **Transaction**  | `id` (PK), `customer_id` (FK → Customer.id), `inventory_id` (FK → Inventory.id) | datetime, amount                                         |
| **Invoice**      | `id` (PK), `transaction_id` (FK → Transaction.id, One-to-One)                   | invoice_no (generated), pdf_path, is_finalized           |
| **Sequence**     | `id` (PK)                                                                       | slug, current_value, format_string                       |
| **Audit**        | `id` (PK), `user_id` (FK → Account.id)                                          | request_meta (JSON), ip_address                          |

---

## **4. Relationships & Constraints**

| Origin Entity         | Target Entity | Type     | Notes                                       |
| --------------------- | ------------- | -------- | ------------------------------------------- |
| Account (Depot Admin) | Depot         | 1 → Many | `on_delete=PROTECT`                         |
| Depot                 | Inventory     | 1 → Many | Tracks LPG stock                            |
| Depot                 | Equipment     | 1 → Many | Tracks non-consumable assets                |
| Account (Driver)      | Distribution  | 1 → Many | Driver assigned to deliveries               |
| Customer              | Transaction   | 1 → Many | Customer can have multiple transactions     |
| Inventory             | Transaction   | 1 → Many | Tracks LPG consumption                      |
| Transaction           | Invoice       | 1 → 1    | One-to-one; triggers invoice generation     |
| Account               | Audit         | 1 → Many | Logs all user actions                       |
| Sequence              | Invoice       | 1 → Many | Ensures gapless, sequential invoice numbers |

**Integrity Rules:**

* `PROTECT` for critical relationships to prevent accidental deletion
* `SET_NULL` for optional foreign keys (e.g., historical references)
* `CASCADE` only for dependent deletions that do not violate compliance

---

## **5. Data Flow & Lifecycle**

1. **Depot Initialization:** Admin updates inventory and equipment.
2. **Distribution Assignment:** Driver receives load via Distribution record; inventory is decremented.
3. **Field Execution:** Transaction created at customer site:

   * Updates inventory
   * Generates invoice PDF (WeasyPrint)
   * Logs audit entry
4. **Atomic Transactions:** Ensures inventory, transaction, and invoice updates are consistent; prevents phantom inventory.

---

## **6. Security & Compliance**

* **JWT Authentication:** Only authorized users can access endpoints.
* **Audit Logging:** Middleware captures request metadata, IP, and user info.
* **Gapless Sequences:** Enforces compliance for Singapore invoice numbering.
* **Referential Integrity:** Prevents accidental deletion of financial and operational data.

---

## **7. ASCII ERD**

```
+---------------------+       1     *       +-----------------+
|       Account       |-------------------->|      Depot      |
|---------------------|                     |-----------------|
| PK id               |                     | PK id           |
| email (unique)      |                     | name            |
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
| loadout (JSON)      |                   | cylinder_size   |
| timestamp           |                   | FK depot_id --- |
+---------------------+                   +-----------------+
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
                                                  1
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
|---------------------|                     |-----------------|
| PK id               |                     | see above       |
| name                |                     +-----------------|
| site_address        |
| email               |
+---------------------+

+---------------------+       1     *       +-----------------+
|      Account        |-------------------->|      Audit      |
|---------------------|                     |-----------------|
| PK id               |                     | PK id           |
| see above           |                     | request_meta    |
+---------------------+                     | ip_address      |
                                            | FK user_id ---- |
                                            +-----------------+

+---------------------+       1     * 
|      Sequence       |-------------------->|     Invoice     |
|---------------------|                     | see above       |
| PK id               |                     +-----------------+
| slug                |
| current_value       |
| format_string       |
+---------------------+
```

**Legend:**

* `1 → *` = One-to-Many
* `1 → 1` = One-to-One
* `FK` = Foreign Key
* `PK` = Primary Key

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

### **8.1 Sequence Flow Explanation**

1. **Login / JWT Authentication:** Driver logs in; API validates JWT.
2. **Create Distribution Record:** API records driver assignment; inventory decremented.
3. **Post Transaction:** Driver submits delivery; inventory updated.
4. **Generate Invoice:** WeasyPrint generates PDF; Sequence entity assigns invoice number.
5. **Audit Logging:** Middleware logs request, user, IP, and response.
6. **Return Status:** API responds to driver with transaction and invoice details.

**Highlights:**

* **Atomic Transactions:** Steps 3–5 are wrapped to prevent partial updates.
* **Compliance-Ready:** Every transaction triggers invoice and audit log.
* **Separation of Concerns:** Inventory, invoicing, and audit modularized.

---

## **9. Production Readiness Notes**

* **Audit-Ready:** All transactions and distributions are logged.
* **Atomicity Ensured:** Prevents incomplete delivery or invoice generation.
* **Scalable:** Async Django supports multiple concurrent drivers.
* **Regulatory Compliance:** Gapless invoice numbers, immutable logs, and protected financial records.


