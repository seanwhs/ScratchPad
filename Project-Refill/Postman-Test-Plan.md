# 🧪 Postman Testing Guide – HSH LPG Sales & Logistics MVP

**Purpose**
This document provides a **complete, production-aligned Postman testing workflow** for the **HSH LPG Sales & Logistics backend**. It is written to reflect **real operational dependencies**, ensuring that APIs are tested in the correct order and with realistic data flows.

The guide covers:

* JWT authentication
* Equipment master data
* Customers
* Inventory
* Distributions
* Transactions
* Invoices
* Audit logs

All examples are **ready to copy into Postman**.

---

## 🔐 Authentication Requirement (Global)

All protected endpoints require the following headers:

| Key           | Value                     |
| ------------- | ------------------------- |
| Authorization | `Bearer {{ACCESS_TOKEN}}` |
| Content-Type  | `application/json`        |

---

## 🎯 1. Postman Environment Setup

Create a Postman environment (**Ctrl + Alt + E**) with the following variables:

| Variable        | Initial Value                                          | Purpose                           |
| --------------- | ------------------------------------------------------ | --------------------------------- |
| BASE_URL        | [http://127.0.0.1:8000/api](http://127.0.0.1:8000/api) | API root                          |
| ACCESS_TOKEN    | ""                                                     | JWT access token                  |
| REFRESH_TOKEN   | ""                                                     | JWT refresh token                 |
| EQUIPMENT_ID    | ""                                                     | Saved after equipment creation    |
| CUSTOMER_ID     | ""                                                     | Saved after customer creation     |
| DISTRIBUTION_ID | ""                                                     | Saved after distribution creation |
| TRANSACTION_ID  | ""                                                     | Saved after transaction creation  |
| INVOICE_ID      | ""                                                     | Saved after invoice creation      |

---

## 🔐 2. JWT Authentication Flow

### 2.1 Login – Obtain Tokens

```
POST {{BASE_URL}}/token/
```

```json
{
  "username": "admin",
  "password": "your_password"
}
```

**Expected Response – 200 OK**

```json
{
  "access": "<jwt_access_token>",
  "refresh": "<jwt_refresh_token>"
}
```

➡ Save tokens into environment variables:

* `ACCESS_TOKEN`
* `REFRESH_TOKEN`

---

### 2.2 Refresh Access Token

```
POST {{BASE_URL}}/token/refresh/
```

```json
{
  "refresh": "{{REFRESH_TOKEN}}"
}
```

---

## 🧰 3. Equipment API (FOUNDATIONAL)

> ⚠️ **Equipment is mandatory**. Inventory, distribution, and transaction APIs will fail if equipment does not exist.

### 3.1 List Equipment

```
GET {{BASE_URL}}/equipment/
```

---

### 3.2 Create Equipment – LPG Cylinder

```
POST {{BASE_URL}}/equipment/
```

```json
{
  "name": "LPG Cylinder 14kg",
  "sku": "LPG-14",
  "equipment_type": "CYLINDER",
  "weight_kg": 14
}
```

➡ Save returned `id` as `EQUIPMENT_ID`

---

### 3.3 Create Equipment – Gas Meter

```
POST {{BASE_URL}}/equipment/
```

```json
{
  "name": "Industrial Gas Meter",
  "sku": "METER-001",
  "equipment_type": "METER",
  "is_active": true
}

```

---

### 3.4 Retrieve Equipment

```
GET {{BASE_URL}}/equipment/{{EQUIPMENT_ID}}/
```

---

## 👤 4. Customer API

### 4.1 List Customers

```
GET {{BASE_URL}}/customers/
```

---

### 4.2 Create Customer

```
POST {{BASE_URL}}/customers/
```

```json
{
  "name": "Test Customer",
  "email": "testcustomer@example.com",
  "payment_type": "CASH"
}
```

➡ Save returned `id` as `CUSTOMER_ID`

---

### 4.3 Retrieve Customer

```
GET {{BASE_URL}}/customers/{{CUSTOMER_ID}}/
```

---

### 4.4 Update Customer

```
PATCH {{BASE_URL}}/customers/{{CUSTOMER_ID}}/
```

```json
{
  "payment_type": "CREDIT"
}
```

---

### 4.5 Delete Customer

```
DELETE {{BASE_URL}}/customers/{{CUSTOMER_ID}}/
```

---

## 📦 5. Inventory API

### 5.1 List Customer Inventory

```
GET {{BASE_URL}}/inventories/?customer_id={{CUSTOMER_ID}}
```

---

### 5.2 Update Inventory

```
POST {{BASE_URL}}/inventories/update_inventory/
```

```json
{
  "entity": "customer",
  "entity_id": {{CUSTOMER_ID}},
  "equipment_id": {{EQUIPMENT_ID}},
  "quantity": 10
}
```

---

## 🚚 6. Distribution API

### 6.1 Create Distribution

```
POST {{BASE_URL}}/distributions/
```

```json
{
  "user_id": 1,
  "items": [
    {
      "depot_id": 1,
      "equipment_id": {{EQUIPMENT_ID}},
      "quantity": 5,
      "movement_type": "Collection"
    }
  ],
  "client_temp_id": "tmp-001"
}
```

➡ Save returned `id` as `DISTRIBUTION_ID`

---

### 6.2 Confirm Distribution

```
POST {{BASE_URL}}/distributions/{{DISTRIBUTION_ID}}/confirm/
```

---

## 💰 7. Transaction API

### 7.1 Create Transaction

```
POST {{BASE_URL}}/transactions/create_transaction/
```

```json
{
  "user": 1,
  "customer": {{CUSTOMER_ID}},
  "current_meter": 1234,
  "items": [
    {
      "equipment_id": {{EQUIPMENT_ID}},
      "quantity": 10,
      "rate": 28.5,
      "type": "Delivery"
    }
  ],
  "client_temp_id": "tmp-001"
}
```

➡ Save `TRANSACTION_ID` and `INVOICE_ID`

---

### 7.2 Retrieve Transaction

```
GET {{BASE_URL}}/transactions/{{TRANSACTION_ID}}/
```

---

## 🧾 8. Invoice API

### 8.1 Retrieve Invoice

```
GET {{BASE_URL}}/invoices/{{INVOICE_ID}}/
```

---

### 8.2 Download Invoice PDF

```
GET {{BASE_URL}}/invoices/{{INVOICE_ID}}/pdf/
```

---

### 8.3 Email Invoice

```
POST {{BASE_URL}}/invoices/{{INVOICE_ID}}/email/
```

---

## 📝 9. Audit Log API

### 9.1 List Audit Logs

```
GET {{BASE_URL}}/audit/
```

---

### 9.2 Retrieve Audit Log

```
GET {{BASE_URL}}/audit/1/
```

---

## 📋 10. Endpoint Matrix

| Endpoint                          | Method | Auth | Purpose                      |
| --------------------------------- | ------ | ---- | ---------------------------- |
| /equipment/                       | GET    | ✅    | List equipment               |
| /equipment/                       | POST   | ✅    | Create equipment             |
| /customers/                       | GET    | ✅    | List customers               |
| /customers/                       | POST   | ✅    | Create customer              |
| /inventories/update_inventory/    | POST   | ✅    | Adjust inventory             |
| /distributions/                   | POST   | ✅    | Create distribution          |
| /distributions/<id>/confirm/      | POST   | ✅    | Confirm distribution         |
| /transactions/create_transaction/ | POST   | ✅    | Create transaction + invoice |
| /invoices/<id>/pdf/               | GET    | ✅    | Download invoice PDF         |
| /audit/                           | GET    | ✅    | View audit logs              |

---

## ⚡ 11. Postman Automation Tips

### Auto-inject JWT (Pre-request Script)

```javascript
pm.request.headers.upsert({
  key: 'Authorization',
  value: 'Bearer ' + pm.environment.get('ACCESS_TOKEN')
});
```

### Auto-save IDs (Tests Script)

```javascript
if (pm.response.code === 201) {
  const data = pm.response.json();
  if (data.id && pm.info.requestName.includes('Equipment')) pm.environment.set('EQUIPMENT_ID', data.id);
  if (data.id && pm.info.requestName.includes('Customer')) pm.environment.set('CUSTOMER_ID', data.id);
  if (data.transaction) pm.environment.set('TRANSACTION_ID', data.transaction.id);
  if (data.invoice) pm.environment.set('INVOICE_ID', data.invoice.id);
}
```

---

## 🧠 Correct Execution Order (CRITICAL)

```
1. Login
2. Equipment
3. Customers
4. Inventory
5. Distribution
6. Transaction
7. Invoice
8. Audit
```

---

✅ This guide now reflects **real LPG operational dependencies**, avoids invalid test states, and is suitable for **team onboarding, QA, and UAT**.
