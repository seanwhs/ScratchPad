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

### **Step 12 – CRUD Verification Matrix**

| Entity        | Endpoint / Method                                  | Body Example (if applicable) | Test Script / Notes                        |
| ------------- | -------------------------------------------------- | ---------------------------- | ------------------------------------------ |
| Customers     | `/customers/{{CUSTOMER_ID}}/` PUT                  | Full customer update         | pm.test("Customer updated", ...)           |
| Customers     | `/customers/{{CUSTOMER_ID}}/` PATCH                | Partial update               | pm.test("Customer patched", ...)           |
| Equipment     | `/equipment/{{EQUIPMENT_ID}}/` PUT                 | Full update                  | pm.test("Equipment updated", ...)          |
| Equipment     | `/equipment/{{EQUIPMENT_ID}}/` PATCH               | Partial update               | pm.test("Equipment patched", ...)          |
| Inventory     | `/inventories/update-inventory/` POST              | Manual inventory update      | pm.test("Inventory manually updated", ...) |
| Distributions | `/distributions/create_distribution/` POST         | Create new distribution run  | pm.test("Distribution created", ...)       |
| Distributions | `/distributions/{{DISTRIBUTION_ID}}/confirm/` POST | Confirm distribution         | pm.test("Distribution confirmed", ...)     |
| Transactions  | `/transactions/create_transaction/` POST           | Create transaction           | pm.test("Transaction created", ...)        |
| Invoices      | `/invoices/{{INVOICE_ID}}/email/` POST             | Send invoice by email        | pm.test("Invoice emailed", ...)            |
| Audit Logs    | `/audit/` GET                                      | List all audit entries       | pm.test("Audit entries listed", ...)       |

> ⚡ **Notes:**
>
> * All actions are **audit logged**; test scripts verify both HTTP status and key field values.
> * PATCH requests can be used for partial updates without needing full object representation.


