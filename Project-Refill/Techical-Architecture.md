# **HSH Sales System – Full Architecture & Workflow**

A **field-first LPG sales and logistics system** for Singapore operations.
Focus: **fast, safe data entry**, **instant invoicing**, **physical + digital proof**, **full traceability of cylinders**.

---

## **1️⃣ Core Philosophy & Users**

**Users:**

* Delivery drivers & field sales staff
* Tablets + mobile thermal printers

**Goals (priority order):**

1. Record cylinder movements (Depot ↔ Truck) safely & quickly
2. Deliver to customers, read meters, perform services
3. Generate correct invoice immediately
4. Print invoice (thermal printer)
5. Auto-email PDF invoice
6. Track cylinder location (Depot / Client Site)
7. Track invoice status: Generated → Printed → Emailed → Paid

**Design Principles:**

* Mobile-first, minimal typing, dynamic dropdowns
* Mandatory human confirmation
* Physical + email proof
* Full traceability: user + timestamp + unique numbers

---

## **2️⃣ Two Main Workflows**

| Workflow     | Purpose                                 | Customer involved? | Creates Invoice? | Prints?          | Emails? | Tracks Cylinders? |
| ------------ | --------------------------------------- | ------------------ | ---------------- | ---------------- | ------- | ----------------- |
| Distribution | Depot ↔ Truck movement of cylinders     | No                 | No               | Optional Receipt | No      | Depot inventory   |
| Transaction  | Customer sale / meter / service billing | Yes                | Yes              | Invoice          | Yes     | Client inventory  |

---

## **3️⃣ MySQL ERD – Distribution & Inventory (Depot ↔ Truck)**

```
+------------------------+
| DistributionHeader      |
|------------------------|
| id (PK)                |
| distribution_number     |
| user_id (FK -> Users)  |
| timestamp              |
| total_collection       |
| total_return           |
+------------------------+
          1
          |
          N
+------------------------+
| DistributionItem        |
|------------------------|
| id (PK)                |
| header_id (FK -> DistributionHeader) |
| depot_id (FK -> Depots)|
| equipment_id (FK -> Equipment)      |
| quantity               |
| movement_type ENUM('Collection','Empty Return') |
+------------------------+
          |
          |
          v
+------------------------+
| Inventory              |
|------------------------|
| id (PK)                |
| depot_id (FK -> Depots)|
| equipment_id (FK -> Equipment)|
| full_qty INT           |
| empty_qty INT          |
+------------------------+
```

**Notes:**

* **DistributionHeader → DistributionItem**: transactional records
* **Inventory**: real-time depot stock
* Movements update inventory atomically (`SELECT ... FOR UPDATE`)

---

## **4️⃣ MySQL ERD – Transaction, Invoice & Client Inventory**

```
+-------------------------+
| TransactionHeader       |
|-------------------------|
| id (PK)                 |
| transaction_number       |
| user_id (FK -> Users)   |
| customer_id (FK -> Customers) |
| timestamp               |
| total_amount            |
+-------------------------+
          1
          |
          N
+-------------------------+
| TransactionItem         |
|-------------------------|
| id (PK)                 |
| transaction_id (FK -> TransactionHeader) |
| equipment_id (FK -> Equipment) |
| quantity                |
| unit_price              |
| item_type ENUM('Usage','Delivery','Service') |
+-------------------------+
          |
          1
          |
          v
+-------------------------+
| Invoice                 |
|-------------------------|
| id (PK)                 |
| transaction_id (FK -> TransactionHeader) |
| pdf_path                |
| status_generated BOOL   |
| status_printed BOOL     |
| status_emailed BOOL     |
| status_paid BOOL        |
+-------------------------+
          |
          |
          v
+-------------------------+
| ClientInventory         |
|-------------------------|
| id (PK)                 |
| customer_id (FK -> Customers) |
| equipment_id (FK -> Equipment) |
| quantity                |
+-------------------------+
```

**Notes:**

* Tracks **customer inventory** after delivery
* Invoice statuses track **Generated → Printed → Emailed → Paid**
* Atomic updates ensure no stock mismatches

---

## **5️⃣ Full Workflow (Frontend → Backend → MySQL → Audit → Print/Email)**

```
┌───────────────────────────────────────┐
│ Frontend (React SPA)                  │
│ Mobile-first UI + Offline Queue       │
├─────────────┬─────────────────────────┤
│ DistributionForm / TransactionForm    │
│ DynamicItemRow / Table                │
│ ConfirmationDialog.jsx                │
└─────────────┬─────────────────────────┘
              │
       [User Confirms]
              │
              ▼
     ┌───────────────────────────┐
     │ Offline Queue (localStorage) │
     │ Saves payload if offline    │
     │ Auto-sync when online       │
     └─────────────┬─────────────┘
                   │
                   ▼
         REST API Call → Django Backend
                   │
┌──────────────────┴─────────────────────┐
│ Backend (Django + DRF)                 │
│ Services:                              │
│  - create_distribution()               │
│  - create_transaction()                │
│  - generate_invoice_pdf()              │
│  - send_email_invoice()                │
│  - update Inventory / ClientInventory │
│  - create AuditLog                     │
└───────────────┬───────────────────────┘
                │
        ┌───────▼────────┐
        │ MySQL Database │
        │----------------│
        │ DistributionHeader + Items   │
        │ TransactionHeader + Items    │
        │ Inventory / ClientInventory  │
        │ Invoice                      │
        │ Customers / Equipment        │
        │ Users                        │
        └─────────┬─────────┘
                  │
          Optional Thermal Print
                  │
          PDF / Email sent
                  │
                  ▼
           Frontend Confirmation / Receipt
```

---

## **6️⃣ Key Features & Safety Principles**

1. **Mandatory Confirmation:** No transaction is committed without human approval
2. **Atomic Transactions:** `SELECT ... FOR UPDATE` ensures inventory integrity
3. **Offline Queue:** Local storage sync for poor connectivity
4. **Traceability:** Every action → user + timestamp + unique ID + audit log
5. **Immediate Proof:** Thermal print + auto PDF/email
6. **Inventory Accuracy:** Depot & ClientInventory always reflect reality

---

## **7️⃣ Combined ERD + Workflow (Hybrid Horizontal Diagram)**

```
┌─────────────────────────────────────────────────────────────┐
│ Frontend (React SPA)                                        │
│ DistributionForm / TransactionForm / ConfirmationDialog     │
└─────────────┬───────────────────────────────────────────────┘
              │
         [User Confirms]
              │
              ▼
       Offline Queue (localStorage) → Auto-sync
              │
              ▼
       REST API Call → Backend (Django/DRF)
              │
┌─────────────┴────────────────────────────┐
│ Backend Services:                         │
│ - create_distribution()                   │
│ - create_transaction()                    │
│ - generate_invoice_pdf()                  │
│ - send_email_invoice()                    │
│ - update Inventory / ClientInventory     │
│ - create AuditLog                         │
└─────────────┬────────────────────────────┘
              │
              ▼
          MySQL Database
  ┌─────────────┬─────────────┐
  │ DistributionHeader + Items │
  │ Inventory                  │
  │ TransactionHeader + Items  │
  │ ClientInventory            │
  │ Invoice                    │
  │ Users / Customers / Equipment │
  └─────────────┬─────────────┘
                │
        Optional Thermal Print / PDF Email
                │
                ▼
          Frontend Confirmation / Receipt
```

---

✅ **Highlights for Developers & Operations Teams:**

* ERDs show **Distribution vs Transaction modules**
* Offline queue ensures **reliability in field operations**
* Atomic updates guarantee **inventory integrity**
* PDF / thermal printing + email ensures **physical + digital proof**
* Audit logs and unique IDs maintain **full traceability**

