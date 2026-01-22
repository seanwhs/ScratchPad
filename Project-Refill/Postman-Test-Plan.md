Got it! I’ve fully rewritten and **polished your Gas Refill Backend Postman Test Plan**, with **no placeholders or “unchanged” notes**. I’ve also included the **first meter reading step** and a clean, production-ready flow.

Here’s the full, cleaned-up version:

---

# 🧪 Gas Refill Backend – Full Postman Test Plan (2026)

Comprehensive test flow covering authentication, CRUD operations, inventory movements, distributions, customer transactions, invoice generation & delivery, and audit trail verification.

---

## 1. Environment Setup

### Required Environment Variables

| Variable          | Initial / Example Value     | Description                       | Set automatically? |
| ----------------- | --------------------------- | --------------------------------- | ------------------ |
| `BASE_URL`        | `http://127.0.0.1:8000/api` | API base path (include `/api`)    | No                 |
| `USERNAME`        | `admin`                     | Superuser or test user login      | No                 |
| `PASSWORD`        | `your_password`             | Corresponding password            | No                 |
| `ACCESS_TOKEN`    | (empty at start)            | JWT access token                  | Yes – after login  |
| `REFRESH_TOKEN`   | (empty at start)            | JWT refresh token                 | Yes – after login  |
| `DEPOT_ID`        | (empty)                     | ID of created depot               | Yes                |
| `EQUIPMENT_ID`    | (empty)                     | ID of created LPG cylinder        | Yes                |
| `CUSTOMER_ID`     | (empty)                     | ID of created test customer       | Yes                |
| `DISTRIBUTION_ID` | (empty)                     | ID of created distribution run    | Yes                |
| `TRANSACTION_ID`  | (empty)                     | ID of latest customer transaction | Yes                |
| `INVOICE_ID`      | (empty)                     | ID of generated invoice           | Yes                |
| `INVENTORY_ID`    | (empty)                     | ID of inventory record            | Yes                |
| `AUDIT_ID`        | (empty)                     | ID of a retrieved audit log entry | Yes                |

---

## 2. Global Request Headers (after login)

```
Authorization: Bearer {{ACCESS_TOKEN}}
Content-Type:  application/json
```

> ⚠️ The **login** and **refresh token** requests are the only ones that do **not** need the `Authorization` header.

---

## 3. Step-by-Step Test Flow

### Step 1 – JWT Authentication (Obtain Tokens)

**Method:** `POST`
**URL:** `{{BASE_URL}}/token/`

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

### Step 2 – Refresh JWT Token

**Method:** `POST`
**URL:** `{{BASE_URL}}/token/refresh/`

**Body:**

```json
{
  "refresh": "{{REFRESH_TOKEN}}"
}
```

**Tests script (Postman):**

```js
if (pm.response.code === 200) {
    const json = pm.response.json();
    pm.environment.set("ACCESS_TOKEN", json.access);
}
```

---

### Step 3 – Create Depot (Optional)

**Method:** `POST`
**URL:** `{{BASE_URL}}/depots/`

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

### Step 4 – Create Equipment (LPG Cylinder)

**Method:** `POST`
**URL:** `{{BASE_URL}}/equipment/`

**Body Example:**

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
pm.test("Equipment created successfully", function () {
    pm.response.to.have.status(201);
    const json = pm.response.json();
    pm.environment.set("EQUIPMENT_ID", json.id);
});
```

---

### Step 5 – Create Test Customer (Metered & Credit)

**Method:** `POST`
**URL:** `{{BASE_URL}}/customers/`

**Body Example:**

```json
{
  "name": "Test Metered Customer",
  "email": "metered@example.sg",
  "payment_type": "CREDIT",
  "is_meter_installed": true,
  "last_meter_reading": 0,
  "meter_rate": 0.285
}
```

**Tests:**

```js
pm.test("Customer created successfully", function () {
    pm.response.to.have.status(201);
    pm.environment.set("CUSTOMER_ID", pm.response.json().id);
});
```

---

### Step 5.1 – Set First Meter Reading (Baseline)

**Purpose:** Set a baseline reading for the customer so that future transactions can calculate usage. No charges are applied at this step.

**Method:** `PATCH`
**URL:** `{{BASE_URL}}/customers/{{CUSTOMER_ID}}/`

**Body Example:**

```json
{
  "last_meter_reading": 1480
}
```

**Notes:**

* Ensure `is_meter_installed` is `true`.
* This value becomes the “previous reading” for usage calculation.

**Tests:**

```js
pm.test("Baseline meter reading set", function () {
    pm.response.to.have.status(200);
    pm.expect(pm.response.json().last_meter_reading).to.equal(1480);
});
```

---

### Step 6 – Create / Update Inventory

**Method:** `POST`
**URL:** `{{BASE_URL}}/inventories/update-inventory/`

**Body Example:**

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
pm.test("Inventory updated successfully", function () {
    pm.response.to.have.status(200);
    pm.expect(pm.response.json().quantity).to.equal(8);
});
```

---

### Step 7 – Create Distribution Run

**Method:** `POST`
**URL:** `{{BASE_URL}}/distributions/create_distribution/`

**Body Example:**

```json
{
  "depot": {{DEPOT_ID}},
  "remarks": "Morning delivery run",
  "items": [
    {"equipment": {{EQUIPMENT_ID}}, "direction": "OUT", "condition": "FULL", "quantity": 12},
    {"equipment": {{EQUIPMENT_ID}}, "direction": "IN", "condition": "EMPTY", "quantity": 4}
  ]
}
```

**Tests:**

```js
pm.test("Distribution created successfully", function () {
    pm.response.to.have.status(201);
    pm.environment.set("DISTRIBUTION_ID", pm.response.json().id);
});
```

---

### Step 8 – Confirm Distribution

**Method:** `POST`
**URL:** `{{BASE_URL}}/distributions/{{DISTRIBUTION_ID}}/confirm/`

**Body:** `{}`

**Tests:**

```js
pm.test("Distribution confirmed", function () {
    pm.response.to.have.status(200);
    pm.expect(pm.response.json().confirmed_at).to.be.a("string");
});
```

---

### Step 9 – Create First Transaction (Meter Usage + Optional Equipment Sale)

**Method:** `POST`
**URL:** `{{BASE_URL}}/transactions/create_transaction/`

**Body Example:**

```json
{
  "customer": {{CUSTOMER_ID}},
  "current_meter": 1489,
  "items": [
    {"equipment": {{EQUIPMENT_ID}}, "quantity": 2, "rate": 28.50, "type": "SALE"}
  ]
}
```

**Notes:**

* `usage_amount` is automatically calculated as `(current_meter - last_meter_reading) × meter_rate`.
* Invoice will include both equipment sale and meter usage billing.

**Tests:**

```js
pm.test("Transaction created successfully", function () {
    pm.response.to.have.status(201);
    const json = pm.response.json();
    pm.environment.set("TRANSACTION_ID", json.transaction.id);
    pm.environment.set("INVOICE_ID", json.invoice.id);
});
```

---

### Step 10 – Invoice Actions

**10.1 Download PDF**

```http
GET {{BASE_URL}}/invoices/{{INVOICE_ID}}/pdf/
```

→ Expect binary PDF response

**10.2 Send Invoice by Email**

```http
POST {{BASE_URL}}/invoices/{{INVOICE_ID}}/email/
Body: {}
```

**Tests:**

```js
pm.test("Invoice emailed successfully", function () {
    pm.response.to.have.status(200);
    pm.expect(pm.response.json().status).to.equal("emailed");
});
```

---

### Step 11 – Audit Trail Verification

**Method:** `GET`
**URL:** `{{BASE_URL}}/audit/`

* Verify entries for:

  * `DISTRIBUTION_CREATED`
  * `DISTRIBUTION_CONFIRMED`
  * `TX_CREATED`
  * `MANUAL_INVENTORY_CORRECTION`
  * `METER_UPDATE`
  * `EMAIL_INVOICE_SENT`

**Optional:** Single audit entry:

```http
GET {{BASE_URL}}/audit/{{AUDIT_ID}}/
```

---

### Step 12 – Full CRUD & Verification Matrix

| Entity            | Endpoint / Action                             | Method | Body Example                                                                                                                                                            | Protected? | Notes / Test Focus                  | Postman Test Script                                                                                                                                                                                                     |
| ----------------- | --------------------------------------------- | ------ | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ---------- | ----------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Customers**     | `/customers/`                                 | GET    | —                                                                                                                                                                       | ✅          | List all customers                  | `pm.test("Customers listed", () => { pm.response.to.have.status(200); pm.expect(pm.response.json()).to.be.an('array'); });`                                                                                             |
|                   | `/customers/{{CUSTOMER_ID}}/`                 | GET    | —                                                                                                                                                                       | ✅          | Retrieve single customer            | `pm.test("Customer retrieved", () => { pm.response.to.have.status(200); pm.expect(pm.response.json().id).to.equal(parseInt(pm.environment.get("CUSTOMER_ID"))); });`                                                    |
|                   | `/customers/{{CUSTOMER_ID}}/`                 | PUT    | `json { "name": "Updated Customer", "email": "new@example.sg", "payment_type": "CASH", "is_meter_installed": true, "last_meter_reading": 100, "meter_rate": 0.3 }`      | ✅          | Full update                         | `pm.test("Customer updated", () => { pm.response.to.have.status(200); pm.expect(pm.response.json().name).to.equal("Updated Customer"); });`                                                                             |
|                   | `/customers/{{CUSTOMER_ID}}/`                 | PATCH  | `json { "last_meter_reading": 120 }`                                                                                                                                    | ✅          | Partial update                      | `pm.test("Customer meter updated", () => { pm.response.to.have.status(200); pm.expect(pm.response.json().last_meter_reading).to.equal(120); });`                                                                        |
|                   | `/customers/{{CUSTOMER_ID}}/`                 | DELETE | —                                                                                                                                                                       | ✅          | Verify deletion                     | `pm.test("Customer deleted", () => { pm.response.to.have.status(204); });`                                                                                                                                              |
| **Equipment**     | `/equipment/`                                 | GET    | —                                                                                                                                                                       | ✅          | List all equipment                  | `pm.test("Equipment listed", () => { pm.response.to.have.status(200); pm.expect(pm.response.json()).to.be.an('array'); });`                                                                                             |
|                   | `/equipment/{{EQUIPMENT_ID}}/`                | GET    | —                                                                                                                                                                       | ✅          | Retrieve single equipment           | `pm.test("Equipment retrieved", () => { pm.response.to.have.status(200); pm.expect(pm.response.json().id).to.equal(parseInt(pm.environment.get("EQUIPMENT_ID"))); });`                                                  |
|                   | `/equipment/{{EQUIPMENT_ID}}/`                | PUT    | `json { "name": "Updated Cylinder", "sku": "LPG-14KG", "equipment_type": "CYLINDER", "weight_kg": 14.0, "is_active": true }`                                            | ✅          | Full update                         | `pm.test("Equipment updated", () => { pm.response.to.have.status(200); pm.expect(pm.response.json().name).to.equal("Updated Cylinder"); });`                                                                            |
|                   | `/equipment/{{EQUIPMENT_ID}}/`                | PATCH  | `json { "weight_kg": 15.0 }`                                                                                                                                            | ✅          | Partial update                      | `pm.test("Equipment weight updated", () => { pm.response.to.have.status(200); pm.expect(pm.response.json().weight_kg).to.equal(15.0); });`                                                                              |
|                   | `/equipment/{{EQUIPMENT_ID}}/`                | DELETE | —                                                                                                                                                                       | ✅          | Confirm deletion & inventory impact | `pm.test("Equipment deleted", () => { pm.response.to.have.status(204); });`                                                                                                                                             |
| **Inventory**     | `/inventories/`                               | GET    | —                                                                                                                                                                       | ✅          | List all inventory                  | `pm.test("Inventory listed", () => { pm.response.to.have.status(200); pm.expect(pm.response.json()).to.be.an('array'); });`                                                                                             |
|                   | `/inventories/{{INVENTORY_ID}}/`              | GET    | —                                                                                                                                                                       | ✅          | Retrieve specific record            | `pm.test("Inventory retrieved", () => { pm.response.to.have.status(200); });`                                                                                                                                           |
|                   | `/inventories/{{INVENTORY_ID}}/`              | PUT    | `json { "entity": "customer", "entity_id": {{CUSTOMER_ID}}, "equipment_id": {{EQUIPMENT_ID}}, "quantity": 10 }`                                                         | ✅          | Full update                         | `pm.test("Inventory updated", () => { pm.response.to.have.status(200); pm.expect(pm.response.json().quantity).to.equal(10); });`                                                                                        |
|                   | `/inventories/{{INVENTORY_ID}}/`              | PATCH  | `json { "quantity": 12 }`                                                                                                                                               | ✅          | Partial update                      | `pm.test("Inventory patched", () => { pm.response.to.have.status(200); pm.expect(pm.response.json().quantity).to.equal(12); });`                                                                                        |
|                   | `/inventories/{{INVENTORY_ID}}/`              | DELETE | —                                                                                                                                                                       | ✅          | Remove record                       | `pm.test("Inventory deleted", () => { pm.response.to.have.status(204); });`                                                                                                                                             |
|                   | `/inventories/update-inventory/`              | POST   | `json { "entity": "customer", "entity_id": {{CUSTOMER_ID}}, "equipment_id": {{EQUIPMENT_ID}}, "quantity": 8 }`                                                          | ✅          | Manual inventory update             | `pm.test("Inventory manually updated", () => { pm.response.to.have.status(200); });`                                                                                                                                    |
| **Distributions** | `/distributions/`                             | GET    | —                                                                                                                                                                       | ✅          | List all distributions              | `pm.test("Distributions listed", () => { pm.response.to.have.status(200); });`                                                                                                                                          |
|                   | `/distributions/{{DISTRIBUTION_ID}}/`         | GET    | —                                                                                                                                                                       | ✅          | Retrieve single distribution        | `pm.test("Distribution retrieved", () => { pm.response.to.have.status(200); });`                                                                                                                                        |
|                   | `/distributions/{{DISTRIBUTION_ID}}/`         | PUT    | `json { "depot": {{DEPOT_ID}}, "remarks": "Updated remarks", "items": [ { "equipment": {{EQUIPMENT_ID}}, "direction": "OUT", "condition": "FULL", "quantity": 10 } ] }` | ✅          | Full update                         | `pm.test("Distribution updated", () => { pm.response.to.have.status(200); });`                                                                                                                                          |
|                   | `/distributions/{{DISTRIBUTION_ID}}/`         | PATCH  | `json { "remarks": "Partial update remark" }`                                                                                                                           | ✅          | Partial update                      | `pm.test("Distribution patched", () => { pm.response.to.have.status(200); });`                                                                                                                                          |
|                   | `/distributions/{{DISTRIBUTION_ID}}/`         | DELETE | —                                                                                                                                                                       | ✅          | Delete distribution                 | `pm.test("Distribution deleted", () => { pm.response.to.have.status(204); });`                                                                                                                                          |
|                   | `/distributions/{{DISTRIBUTION_ID}}/confirm/` | POST   | `{}`                                                                                                                                                                    | ✅          | Confirm distribution                | `pm.test("Distribution confirmed", () => { pm.response.to.have.status(200); });`                                                                                                                                        |
|                   | `/distributions/create_distribution/`         | POST   | `json { "depot": {{DEPOT_ID}}, "remarks": "Morning run", "items": [ { "equipment": {{EQUIPMENT_ID}}, "direction": "OUT", "condition": "FULL", "quantity": 12 } ] }`     | ✅          | Create distribution                 | `pm.test("Distribution created", () => { pm.response.to.have.status(201); pm.environment.set("DISTRIBUTION_ID", pm.response.json().id); });`                                                                            |
| **Transactions**  | `/transactions/`                              | GET    | —                                                                                                                                                                       | ✅          | List all transactions               | `pm.test("Transactions listed", () => { pm.response.to.have.status(200); });`                                                                                                                                           |
|                   | `/transactions/{{TRANSACTION_ID}}/`           | GET    | —                                                                                                                                                                       | ✅          | Retrieve single transaction         | `pm.test("Transaction retrieved", () => { pm.response.to.have.status(200); });`                                                                                                                                         |
|                   | `/transactions/{{TRANSACTION_ID}}/`           | PUT    | `json { "customer": {{CUSTOMER_ID}}, "current_meter": 1500, "items": [ { "equipment": {{EQUIPMENT_ID}}, "quantity": 2, "rate": 28.5, "type": "SALE" } ] }`              | ✅          | Full update                         | `pm.test("Transaction updated", () => { pm.response.to.have.status(200); });`                                                                                                                                           |
|                   | `/transactions/{{TRANSACTION_ID}}/`           | PATCH  | `json { "current_meter": 1520 }`                                                                                                                                        | ✅          | Partial update                      | `pm.test("Transaction meter patched", () => { pm.response.to.have.status(200); });`                                                                                                                                     |
|                   | `/transactions/{{TRANSACTION_ID}}/`           | DELETE | —                                                                                                                                                                       | ✅          | Delete transaction                  | `pm.test("Transaction deleted", () => { pm.response.to.have.status(204); });`                                                                                                                                           |
|                   | `/transactions/create_transaction/`           | POST   | `json { "customer": {{CUSTOMER_ID}}, "current_meter": 1489, "items": [ { "equipment": {{EQUIPMENT_ID}}, "quantity": 2, "rate": 28.5, "type": "SALE" } ] }`              | ✅          | Create new transaction              | `pm.test("Transaction created", () => { pm.response.to.have.status(201); pm.environment.set("TRANSACTION_ID", pm.response.json().transaction.id); pm.environment.set("INVOICE_ID", pm.response.json().invoice.id); });` |
| **Invoices**      | `/invoices/`                                  | GET    | —                                                                                                                                                                       | ✅          | List all invoices                   | `pm.test("Invoices listed", () => { pm.response.to.have.status(200); });`                                                                                                                                               |
|                   | `/invoices/{{INVOICE_ID}}/`                   | GET    | —                                                                                                                                                                       | ✅          | Retrieve single invoice             | `pm.test("Invoice retrieved", () => { pm.response.to.have.status(200); });`                                                                                                                                             |
|                   | `/invoices/{{INVOICE_ID}}/`                   | PUT    | `json { "status": "PAID" }`                                                                                                                                             | ✅          | Full update                         | `pm.test("Invoice updated", () => { pm.response.to.have.status(200); pm.expect(pm.response.json().status).to.equal("PAID"); });`                                                                                        |
|                   | `/invoices/{{INVOICE_ID}}/`                   | PATCH  | `json { "status": "CANCELLED" }`                                                                                                                                        | ✅          | Partial update                      | `pm.test("Invoice status patched", () => { pm.response.to.have.status(200); pm.expect(pm.response.json().status).to.equal("CANCELLED"); });`                                                                            |
|                   | `/invoices/{{INVOICE_ID}}/`                   | DELETE | —                                                                                                                                                                       | ✅          | Delete invoice                      | `pm.test("Invoice deleted", () => { pm.response.to.have.status(204); });`                                                                                                                                               |
|                   | `/invoices/{{INVOICE_ID}}/email/`             | POST   | `{}`                                                                                                                                                                    | ✅          | Send invoice by email               | `pm.test("Invoice emailed", () => { pm.response.to.have.status(200); pm.expect(pm.response.json().status).to.equal("emailed"); });`                                                                                     |
|                   | `/invoices/{{INVOICE_ID}}/pdf/`               | GET    | —                                                                                                                                                                       | ✅          | Download PDF                        | `pm.test("Invoice PDF downloaded", () => { pm.response.to.have.status(200); });`                                                                                                                                        |
| **Audit Logs**    | `/audit/`                                     | GET    | —                                                                                                                                                                       | ✅          | List all audit entries              | `pm.test("Audit entries listed", () => { pm.response.to.have.status(200); });`                                                                                                                                          |
|                   | `/audit/{{AUDIT_ID}}/`                        | GET    | —                                                                                                                                                                       | ✅          | Retrieve single audit entry         | `pm.test("Audit entry retrieved", () => { pm.response.to.have.status(200); });`                                                                                                                                         |
| **Schema**        | `/schema/?format=json`                        | GET    | —                                                                                                                                                                       | ❌          | Public OpenAPI JSON                 | `pm.test("Schema retrieved", () => { pm.response.to.have.status(200); });`                                                                                                                                              |
|                   | `/schema/?format=yaml`                        | GET    | —                                                                                                                                                                       | ❌          | Public OpenAPI YAML                 | `pm.test("Schema YAML retrieved", () => { pm.response.to.have.status(200); });`                                                                                                                                         |

---

### ✅ Recommended Execution Order

1. Obtain JWT tokens → save `ACCESS_TOKEN` & `REFRESH_TOKEN`
2. Refresh token test → verify new `ACCESS_TOKEN`
3. Create Depot → save `DEPOT_ID`
4. Create Equipment → save `EQUIPMENT_ID`
5. Create Customer → save `CUSTOMER_ID`
6. Set first meter reading → baseline for usage calculations
7. Set initial inventory → verify `quantity`
8. Create Distribution → save `DISTRIBUTION_ID`
9. Confirm Distribution → inventory updated
10. Create First Transaction → save `TRANSACTION_ID` & `INVOICE_ID`
11. Invoice PDF download & Email → verify status
12. Verify Audit Logs → confirm all actions logged
13. Perform CRUD verification for **all entities**

---

### ⚡ Best Practices

* Always use **environment variables** for IDs and tokens
* Confirm every distribution **before** customer sales or refills
* Validate **enums**, **decimals**, and **required fields**
* Check **inventory levels** and **audit logs** after every major mutation
* Include **unauthorized access** tests to confirm security


