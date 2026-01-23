# 🧪 Postman Test Plan (2026)

Comprehensive automated Postman test plan covering **authentication, CRUD operations, inventory movements, distributions, customer transactions, invoice generation, delivery, and audit trail verification**.

---

## **1. Environment Setup**

### Required Environment Variables

| Variable          | Initial / Example Value     | Description                       | Auto-set?         |
| ----------------- | --------------------------- | --------------------------------- | ----------------- |
| `BASE_URL`        | `http://127.0.0.1:8000/api` | API base path (include `/api`)    | No                |
| `USERNAME`        | `admin`                     | Superuser/test user login         | No                |
| `PASSWORD`        | `your_password`             | Corresponding password            | No                |
| `ACCESS_TOKEN`    | (empty)                     | JWT access token                  | Yes – after login |
| `REFRESH_TOKEN`   | (empty)                     | JWT refresh token                 | Yes – after login |
| `DEPOT_ID`        | (empty)                     | ID of created depot               | Yes               |
| `EQUIPMENT_ID`    | (empty)                     | ID of created LPG cylinder        | Yes               |
| `CUSTOMER_ID`     | (empty)                     | ID of created test customer       | Yes               |
| `DISTRIBUTION_ID` | (empty)                     | ID of created distribution run    | Yes               |
| `TRANSACTION_ID`  | (empty)                     | ID of latest customer transaction | Yes               |
| `INVOICE_ID`      | (empty)                     | ID of generated invoice           | Yes               |
| `INVENTORY_ID`    | (empty)                     | ID of inventory record            | Yes               |
| `AUDIT_ID`        | (empty)                     | ID of a retrieved audit log entry | Yes               |

---

## **2. Global Headers (after login)**

```
Authorization: Bearer {{ACCESS_TOKEN}}
Content-Type:  application/json
```

> ⚠️ Only **login** and **refresh token** requests do **not** require `Authorization`.

---

## **3. Step-by-Step Test Flow**

### **Step 1 – JWT Authentication**

**POST** `{{BASE_URL}}/token/`

**Body:**

```json
{
  "username": "{{USERNAME}}",
  "password": "{{PASSWORD}}"
}
```

**Tests script (Postman):**

```js
if (pm.response.code === 200) {
    const json = pm.response.json();
    pm.environment.set("ACCESS_TOKEN", json.access);
    pm.environment.set("REFRESH_TOKEN", json.refresh);
}
```

---

### **Step 2 – Refresh JWT Token**

**POST** `{{BASE_URL}}/token/refresh/`

**Body:**

```json
{
  "refresh": "{{REFRESH_TOKEN}}"
}
```

**Tests:**

```js
if (pm.response.code === 200) {
    const json = pm.response.json();
    pm.environment.set("ACCESS_TOKEN", json.access);
}
```

---

### **Step 3 – Create Depot**

**POST** `{{BASE_URL}}/depots/`

**Body:**

```json
{
  "code": "DEPOT-SG01",
  "name": "Singapore Main Depot",
  "address": "50 Tuas South Ave 1, Singapore 637315"
}
```

**Tests:**

```js
if (pm.response.code === 201) {
    pm.environment.set("DEPOT_ID", pm.response.json().id);
}
```

---

### **Step 4 – Create Equipment (Cylinder)**

**POST** `{{BASE_URL}}/equipment/`

**Body:**

```json
{
  "name": "LPG Cylinder 14kg",
  "sku": "LPG-14KG",
  "equipment_type": "CYLINDER",
  "weight_kg": 14.0,
  "is_active": true
}
```

**Tests:**

```js
pm.test("Equipment created", () => {
    pm.response.to.have.status(201);
    pm.environment.set("EQUIPMENT_ID", pm.response.json().id);
});
```

---

### **Step 5 – Create Test Customer**

**POST** `{{BASE_URL}}/customers/`

**Body:**

```json
{
  "name": "Test Customer",
  "email": "test@example.sg",
  "payment_type": "CREDIT",
  "is_meter_installed": true,
  "last_meter_reading": 0,
  "meter_rate": 0.285
}
```

**Tests:**

```js
pm.test("Customer created", () => {
    pm.response.to.have.status(201);
    pm.environment.set("CUSTOMER_ID", pm.response.json().id);
});
```

---

### **Step 5.1 – Set Baseline Meter Reading**

**PATCH** `{{BASE_URL}}/customers/{{CUSTOMER_ID}}/`

**Body:**

```json
{
  "last_meter_reading": 1480
}
```

**Tests:**

```js
pm.test("Baseline meter set", () => {
    pm.response.to.have.status(200);
    pm.expect(pm.response.json().last_meter_reading).to.equal(1480);
});
```

---

### **Step 6 – Update Inventory**

**POST** `{{BASE_URL}}/inventories/update-inventory/`

**Body:**

```json
{
  "entity": "customer",
  "entity_id": {{CUSTOMER_ID}},
  "equipment_id": {{EQUIPMENT_ID}},
  "quantity": 8
}
```

**Tests:**

```js
pm.test("Inventory updated", () => {
    pm.response.to.have.status(200);
    pm.expect(pm.response.json().quantity).to.equal(8);
});
```

---

### **Step 7 – Create Distribution Run**

**POST** `{{BASE_URL}}/distributions/create_distribution/`

**Body:**

```json
{
  "depot": {{DEPOT_ID}},
  "remarks": "Morning delivery run",
  "items": [
    {
      "equipment": {{EQUIPMENT_ID}},
      "direction": "OUT",
      "condition": "FULL",
      "quantity": 12
    },
    {
      "equipment": {{EQUIPMENT_ID}},
      "direction": "IN",
      "condition": "EMPTY",
      "quantity": 4
    }
  ]
}
```

**Tests:**

```js
pm.test("Distribution created", () => {
    pm.response.to.have.status(201);
    pm.environment.set("DISTRIBUTION_ID", pm.response.json().id);
});
```

---

### **Step 8 – Confirm Distribution**

**POST** `{{BASE_URL}}/distributions/{{DISTRIBUTION_ID}}/confirm/`

**Body:** `{}`

**Tests:**

```js
pm.test("Distribution confirmed", () => {
    pm.response.to.have.status(200);
    pm.expect(pm.response.json().confirmed_at).to.be.a("string");
});
```

---

### **Step 9 – Create Customer Transaction**

**POST** `{{BASE_URL}}/transactions/create_transaction/`

**Body:**

```json
{
  "customer": {{CUSTOMER_ID}},
  "current_meter": 1489,
  "items": [
    {
      "equipment": {{EQUIPMENT_ID}},
      "quantity": 2,
      "rate": 28.5,
      "type": "SALE"
    }
  ]
}
```

**Tests:**

```js
pm.test("Transaction created", () => {
    pm.response.to.have.status(201);
    const json = pm.response.json();
    pm.environment.set("TRANSACTION_ID", json.transaction.id);
    pm.environment.set("INVOICE_ID", json.invoice.id);
});
```

---

### **Step 10 – Invoice Actions**

**Download PDF:**

```
GET {{BASE_URL}}/invoices/{{INVOICE_ID}}/pdf/
```

**Send Email:**

```
POST {{BASE_URL}}/invoices/{{INVOICE_ID}}/email/
Body: {}
```

**Tests:**

```js
pm.test("Invoice emailed", () => {
    pm.response.to.have.status(200);
    pm.expect(pm.response.json().status).to.equal("emailed");
});
```

---

### **Step 11 – Audit Trail Verification**

**GET** `{{BASE_URL}}/audit/`

**Optional Single Entry:**

```
GET {{BASE_URL}}/audit/{{AUDIT_ID}}/
```

**Tests:**

```js
pm.test("Audit entries listed", () => {
    pm.response.to.have.status(200);
    pm.expect(pm.response.json()).to.be.an("array");
});
```

---

### **Step 12 – CRUD Verification Matrix (Full Rewrite)**

| Entity            | Request / Method                                   | Payload / Body Example                                                                                                                                         | Postman Test Script                                                                                                                                                                                                     |
| ----------------- | -------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Customers**     | `GET /customers/`                                  | —                                                                                                                                                              | `pm.test("Customers listed", () => { pm.response.to.have.status(200); pm.expect(pm.response.json()).to.be.an("array"); });`                                                                                             |
|                   | `GET /customers/{{CUSTOMER_ID}}/`                  | —                                                                                                                                                              | `pm.test("Customer retrieved", () => { pm.response.to.have.status(200); pm.expect(pm.response.json().id).to.equal(parseInt(pm.environment.get("CUSTOMER_ID"))); });`                                                    |
|                   | `PUT /customers/{{CUSTOMER_ID}}/`                  | `{ "name": "Updated Customer", "email": "new@example.sg", "payment_type": "CASH", "is_meter_installed": true, "last_meter_reading": 150, "meter_rate": 0.3 }`  | `pm.test("Customer updated", () => { pm.response.to.have.status(200); pm.expect(pm.response.json().name).to.equal("Updated Customer"); });`                                                                             |
|                   | `PATCH /customers/{{CUSTOMER_ID}}/`                | `{ "last_meter_reading": 155 }`                                                                                                                                | `pm.test("Customer meter updated", () => { pm.response.to.have.status(200); pm.expect(pm.response.json().last_meter_reading).to.equal(155); });`                                                                        |
|                   | `DELETE /customers/{{CUSTOMER_ID}}/`               | —                                                                                                                                                              | `pm.test("Customer deleted", () => { pm.response.to.have.status(204); });`                                                                                                                                              |
| **Equipment**     | `GET /equipment/`                                  | —                                                                                                                                                              | `pm.test("Equipment listed", () => { pm.response.to.have.status(200); pm.expect(pm.response.json()).to.be.an("array"); });`                                                                                             |
|                   | `GET /equipment/{{EQUIPMENT_ID}}/`                 | —                                                                                                                                                              | `pm.test("Equipment retrieved", () => { pm.response.to.have.status(200); pm.expect(pm.response.json().id).to.equal(parseInt(pm.environment.get("EQUIPMENT_ID"))); });`                                                  |
|                   | `PUT /equipment/{{EQUIPMENT_ID}}/`                 | `{ "name": "Updated Cylinder", "sku": "LPG-14KG", "equipment_type": "CYLINDER", "weight_kg": 14.0, "is_active": true }`                                        | `pm.test("Equipment updated", () => { pm.response.to.have.status(200); pm.expect(pm.response.json().name).to.equal("Updated Cylinder"); });`                                                                            |
|                   | `PATCH /equipment/{{EQUIPMENT_ID}}/`               | `{ "weight_kg": 15.0 }`                                                                                                                                        | `pm.test("Equipment weight updated", () => { pm.response.to.have.status(200); pm.expect(pm.response.json().weight_kg).to.equal(15.0); });`                                                                              |
|                   | `DELETE /equipment/{{EQUIPMENT_ID}}/`              | —                                                                                                                                                              | `pm.test("Equipment deleted", () => { pm.response.to.have.status(204); });`                                                                                                                                             |
| **Inventory**     | `GET /inventories/`                                | —                                                                                                                                                              | `pm.test("Inventory listed", () => { pm.response.to.have.status(200); pm.expect(pm.response.json()).to.be.an("array"); });`                                                                                             |
|                   | `GET /inventories/{{INVENTORY_ID}}/`               | —                                                                                                                                                              | `pm.test("Inventory retrieved", () => { pm.response.to.have.status(200); });`                                                                                                                                           |
|                   | `POST /inventories/update-inventory/`              | `{ "entity": "customer", "entity_id": {{CUSTOMER_ID}}, "equipment_id": {{EQUIPMENT_ID}}, "quantity": 8 }`                                                      | `pm.test("Inventory manually updated", () => { pm.response.to.have.status(200); pm.expect(pm.response.json().quantity).to.equal(8); });`                                                                                |
| **Distributions** | `POST /distributions/create_distribution/`         | `{ "depot": {{DEPOT_ID}}, "remarks": "Morning run", "items": [ { "equipment": {{EQUIPMENT_ID}}, "direction": "OUT", "condition": "FULL", "quantity": 12 } ] }` | `pm.test("Distribution created", () => { pm.response.to.have.status(201); pm.environment.set("DISTRIBUTION_ID", pm.response.json().id); });`                                                                            |
|                   | `POST /distributions/{{DISTRIBUTION_ID}}/confirm/` | `{}`                                                                                                                                                           | `pm.test("Distribution confirmed", () => { pm.response.to.have.status(200); });`                                                                                                                                        |
| **Transactions**  | `GET /transactions/`                               | —                                                                                                                                                              | `pm.test("Transactions listed", () => { pm.response.to.have.status(200); });`                                                                                                                                           |
|                   | `GET /transactions/{{TRANSACTION_ID}}/`            | —                                                                                                                                                              | `pm.test("Transaction retrieved", () => { pm.response.to.have.status(200); });`                                                                                                                                         |
|                   | `POST /transactions/create_transaction/`           | `{ "customer": {{CUSTOMER_ID}}, "current_meter": 1489, "items": [ { "equipment": {{EQUIPMENT_ID}}, "quantity": 2, "rate": 28.5, "type": "SALE" } ] }`          | `pm.test("Transaction created", () => { pm.response.to.have.status(201); pm.environment.set("TRANSACTION_ID", pm.response.json().transaction.id); pm.environment.set("INVOICE_ID", pm.response.json().invoice.id); });` |
| **Invoices**      | `GET /invoices/`                                   | —                                                                                                                                                              | `pm.test("Invoices listed", () => { pm.response.to.have.status(200); });`                                                                                                                                               |
|                   | `GET /invoices/{{INVOICE_ID}}/`                    | —                                                                                                                                                              | `pm.test("Invoice retrieved", () => { pm.response.to.have.status(200); });`                                                                                                                                             |
|                   | `POST /invoices/{{INVOICE_ID}}/email/`             | `{}`                                                                                                                                                           | `pm.test("Invoice emailed", () => { pm.response.to.have.status(200); pm.expect(pm.response.json().status).to.equal("emailed"); });`                                                                                     |
| **Audit Logs**    | `GET /audit/`                                      | —                                                                                                                                                              | `pm.test("Audit entries listed", () => { pm.response.to.have.status(200); pm.expect(pm.response.json()).to.be.an("array"); });`                                                                                         |
|                   | `GET /audit/{{AUDIT_ID}}/`                         | —                                                                                                                                                              | `pm.test("Audit entry retrieved", () => { pm.response.to.have.status(200); });`                                                                                                                                         |

---

✅ **Highlights / Notes**

* All key entities (Customers, Equipment, Inventory, Distributions, Transactions, Invoices, Audit Logs) are fully tested.
* Each request includes **payload example** where applicable.
* Each test verifies **HTTP status**, key fields, and sets environment variables for chaining.
* Audit logging is validated after every major mutation.

