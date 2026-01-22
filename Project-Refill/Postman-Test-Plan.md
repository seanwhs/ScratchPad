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

**Tests script:**

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

**Tests script:**

```js
if (pm.response.code === 200) {
    const json = pm.response.json();
    pm.environment.set("ACCESS_TOKEN", json.access);
}
```

---

### Step 3 – Create Depot (Optional if applicable)

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

### Step 4 – Create Equipment

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
if (pm.response.code === 201) {
    pm.environment.set("EQUIPMENT_ID", pm.response.json().id);
}
```

---

### Step 5 – Create Test Customer

**Method:** `POST`
**URL:** `{{BASE_URL}}/customers/`

**Metered / Credit Customer (recommended for full flow):**

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
if (pm.response.code === 201) {
    pm.environment.set("CUSTOMER_ID", pm.response.json().id);
}
```

---

### Step 6 – Create / Update Inventory

**Method:** `POST`
**URL:** `{{BASE_URL}}/inventories/update-inventory/`

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
pm.test("Inventory updated successfully", function () {
    pm.response.to.have.status(200);
    pm.expect(pm.response.json().quantity).to.equal(8);
});
```

---

### Step 7 – Create Distribution Run

**Method:** `POST`
**URL:** `{{BASE_URL}}/distributions/create_distribution/`

**Body:**

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
if (pm.response.code === 201) {
    pm.environment.set("DISTRIBUTION_ID", pm.response.json().id);
}
```

---

### Step 8 – Confirm Distribution

**Method:** `POST`
**URL:** `{{BASE_URL}}/distributions/{{DISTRIBUTION_ID}}/confirm/`

**Body:** `{}`

**Tests:**

```js
pm.test("Distribution confirmed successfully", function () {
    pm.response.to.have.status(200);
    pm.expect(pm.response.json().confirmed_at).to.be.a("string");
});
```

---

### Step 9 – Transactions (Customer Meter + Cylinder Sale)

**Method:** `POST`
**URL:** `{{BASE_URL}}/transactions/create_transaction/`

**Body Example (Meter + Sale):**

```json
{
  "customer": {{CUSTOMER_ID}},
  "current_meter": 1489.0,
  "items": [
    {"equipment": {{EQUIPMENT_ID}}, "quantity": 2, "rate": 28.50, "type": "SALE"}
  ]
}
```

**Tests:**

```js
if (pm.response.code === 201) {
    const json = pm.response.json();
    pm.environment.set("TRANSACTION_ID", json.transaction.id);
    pm.environment.set("INVOICE_ID", json.invoice.id);
}
```

---

### Step 10 – Invoice Actions

**10.1 Download PDF**

**Method:** `GET`
**URL:** `{{BASE_URL}}/invoices/{{INVOICE_ID}}/pdf/`

→ Expect binary PDF response

**10.2 Send Invoice by Email**

**Method:** `POST`
**URL:** `{{BASE_URL}}/invoices/{{INVOICE_ID}}/email/`

**Body:** `{}`

**Tests:**

```js
pm.test("Invoice marked as emailed", function () {
    pm.response.to.have.status(200);
    pm.expect(pm.response.json().status).to.equal("emailed");
});
```

---

### Step 11 – Audit Trail Verification

**Method:** `GET`
**URL:** `{{BASE_URL}}/audit/`

* Verify recent entries for:

  * `DISTRIBUTION_CREATED`, `DISTRIBUTION_CONFIRMED`
  * `TX_CREATED`
  * `MANUAL_INVENTORY_CORRECTION`
  * `METER_UPDATE`
  * `EMAIL_INVOICE_SENT`

**Optional:** Retrieve single audit entry:

```
GET {{BASE_URL}}/audit/{{AUDIT_ID}}/
```

---

### Step 12 – Full CRUD & Verification Matrix (with Body & Auth Info)

| Entity            | Endpoint / Action                             | Method | Body Example                                                                                                                                                            | Protected? | Notes / Test Focus                                          |
| ----------------- | --------------------------------------------- | ------ | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ---------- | ----------------------------------------------------------- |
| **Customers**     | `/customers/`                                 | GET    | —                                                                                                                                                                       | ✅          | List all customers                                          |
|                   | `/customers/{{CUSTOMER_ID}}/`                 | GET    | —                                                                                                                                                                       | ✅          | Retrieve single customer                                    |
|                   | `/customers/{{CUSTOMER_ID}}/`                 | PUT    | `json { "name": "Updated Customer", "email": "new@example.sg", "payment_type": "CASH", "is_meter_installed": true, "last_meter_reading": 100, "meter_rate": 0.3 }`      | ✅          | Full update; validate enums, decimals                       |
|                   | `/customers/{{CUSTOMER_ID}}/`                 | PATCH  | `json { "last_meter_reading": 120 }`                                                                                                                                    | ✅          | Partial update                                              |
|                   | `/customers/{{CUSTOMER_ID}}/`                 | DELETE | —                                                                                                                                                                       | ✅          | Verify deletion & 404 on subsequent GET                     |
| **Equipment**     | `/equipment/`                                 | GET    | —                                                                                                                                                                       | ✅          | List all equipment                                          |
|                   | `/equipment/{{EQUIPMENT_ID}}/`                | GET    | —                                                                                                                                                                       | ✅          | Retrieve single equipment                                   |
|                   | `/equipment/{{EQUIPMENT_ID}}/`                | PUT    | `json { "name": "Updated Cylinder", "sku": "LPG-14KG", "equipment_type": "CYLINDER", "weight_kg": 14.0, "is_active": true }`                                            | ✅          | Full update; validate required fields                       |
|                   | `/equipment/{{EQUIPMENT_ID}}/`                | PATCH  | `json { "weight_kg": 15.0 }`                                                                                                                                            | ✅          | Partial update; validate decimal                            |
|                   | `/equipment/{{EQUIPMENT_ID}}/`                | DELETE | —                                                                                                                                                                       | ✅          | Confirm deletion & impact on inventory                      |
| **Inventory**     | `/inventories/`                               | GET    | —                                                                                                                                                                       | ✅          | List all inventory records                                  |
|                   | `/inventories/{{INVENTORY_ID}}/`              | GET    | —                                                                                                                                                                       | ✅          | Retrieve a specific inventory record                        |
|                   | `/inventories/{{INVENTORY_ID}}/`              | PUT    | `json { "entity": "customer", "entity_id": {{CUSTOMER_ID}}, "equipment_id": {{EQUIPMENT_ID}}, "quantity": 10 }`                                                         | ✅          | Full update; verify quantity & direction                    |
|                   | `/inventories/{{INVENTORY_ID}}/`              | PATCH  | `json { "quantity": 12 }`                                                                                                                                               | ✅          | Partial update                                              |
|                   | `/inventories/{{INVENTORY_ID}}/`              | DELETE | —                                                                                                                                                                       | ✅          | Remove record; verify inventory consistency                 |
|                   | `/inventories/update-inventory/`              | POST   | `json { "entity": "customer", "entity_id": {{CUSTOMER_ID}}, "equipment_id": {{EQUIPMENT_ID}}, "quantity": 8 }`                                                          | ✅          | Manual inventory update; test invalid/negative quantities   |
| **Distributions** | `/distributions/`                             | GET    | —                                                                                                                                                                       | ✅          | List all distributions                                      |
|                   | `/distributions/{{DISTRIBUTION_ID}}/`         | GET    | —                                                                                                                                                                       | ✅          | Retrieve single distribution                                |
|                   | `/distributions/{{DISTRIBUTION_ID}}/`         | PUT    | `json { "depot": {{DEPOT_ID}}, "remarks": "Updated remarks", "items": [ { "equipment": {{EQUIPMENT_ID}}, "direction": "OUT", "condition": "FULL", "quantity": 10 } ] }` | ✅          | Full update; validate all fields                            |
|                   | `/distributions/{{DISTRIBUTION_ID}}/`         | PATCH  | `json { "remarks": "Partial update remark" }`                                                                                                                           | ✅          | Partial update; confirm inventory adjustments               |
|                   | `/distributions/{{DISTRIBUTION_ID}}/`         | DELETE | —                                                                                                                                                                       | ✅          | Delete distribution; verify cascading effects               |
|                   | `/distributions/{{DISTRIBUTION_ID}}/confirm/` | POST   | `{}`                                                                                                                                                                    | ✅          | Confirm distribution; verify stock levels                   |
|                   | `/distributions/create_distribution/`         | POST   | `json { "depot": {{DEPOT_ID}}, "remarks": "Morning run", "items": [ { "equipment": {{EQUIPMENT_ID}}, "direction": "OUT", "condition": "FULL", "quantity": 12 } ] }`     | ✅          | Create distribution; validate items & quantities            |
| **Transactions**  | `/transactions/`                              | GET    | —                                                                                                                                                                       | ✅          | List all transactions                                       |
|                   | `/transactions/{{TRANSACTION_ID}}/`           | GET    | —                                                                                                                                                                       | ✅          | Retrieve single transaction                                 |
|                   | `/transactions/{{TRANSACTION_ID}}/`           | PUT    | `json { "customer": {{CUSTOMER_ID}}, "current_meter": 1500, "items": [ { "equipment": {{EQUIPMENT_ID}}, "quantity": 2, "rate": 28.5, "type": "SALE" } ] }`              | ✅          | Full update; validate totals & items                        |
|                   | `/transactions/{{TRANSACTION_ID}}/`           | PATCH  | `json { "current_meter": 1520 }`                                                                                                                                        | ✅          | Partial update; check inventory & meter readings            |
|                   | `/transactions/{{TRANSACTION_ID}}/`           | DELETE | —                                                                                                                                                                       | ✅          | Delete transaction; verify invoice & inventory rollback     |
|                   | `/transactions/create_transaction/`           | POST   | `json { "customer": {{CUSTOMER_ID}}, "current_meter": 1489, "items": [ { "equipment": {{EQUIPMENT_ID}}, "quantity": 2, "rate": 28.5, "type": "SALE" } ] }`              | ✅          | New transaction; verify totals, inventory, invoice creation |
| **Invoices**      | `/invoices/`                                  | GET    | —                                                                                                                                                                       | ✅          | List all invoices                                           |
|                   | `/invoices/{{INVOICE_ID}}/`                   | GET    | —                                                                                                                                                                       | ✅          | Retrieve single invoice                                     |
|                   | `/invoices/{{INVOICE_ID}}/`                   | PUT    | `json { "status": "PAID" }`                                                                                                                                             | ✅          | Full update; validate status & totals                       |
|                   | `/invoices/{{INVOICE_ID}}/`                   | PATCH  | `json { "status": "CANCELLED" }`                                                                                                                                        | ✅          | Partial update                                              |
|                   | `/invoices/{{INVOICE_ID}}/`                   | DELETE | —                                                                                                                                                                       | ✅          | Delete invoice; confirm impact on transaction               |
|                   | `/invoices/{{INVOICE_ID}}/email/`             | POST   | `{}`                                                                                                                                                                    | ✅          | Send invoice by email; verify status                        |
|                   | `/invoices/{{INVOICE_ID}}/pdf/`               | GET    | —                                                                                                                                                                       | ✅          | Download PDF; binary response                               |
| **Audit Logs**    | `/audit/`                                     | GET    | —                                                                                                                                                                       | ✅          | List all audit entries; verify recent actions               |
|                   | `/audit/{{AUDIT_ID}}/`                        | GET    | —                                                                                                                                                                       | ✅          | Retrieve single audit entry                                 |
| **Schema**        | `/schema/?format=json`                        | GET    | —                                                                                                                                                                       | ❌          | Public; retrieve OpenAPI JSON schema                        |
|                   | `/schema/?format=yaml`                        | GET    | —                                                                                                                                                                       | ❌          | Public; retrieve OpenAPI YAML schema                        |

---

### ✅ Recommended Execution Order

1. Obtain JWT tokens → save `ACCESS_TOKEN` & `REFRESH_TOKEN`
2. Refresh token test → verify new `ACCESS_TOKEN`
3. Create Depot → save `DEPOT_ID`
4. Create Equipment → save `EQUIPMENT_ID`
5. Create Customer → save `CUSTOMER_ID`
6. Set initial inventory → verify `quantity`
7. Create Distribution → save `DISTRIBUTION_ID`
8. Confirm Distribution → inventory updated
9. Create Transaction → save `TRANSACTION_ID` & `INVOICE_ID`
10. Invoice PDF download & Email → verify status
11. Verify Audit Logs → confirm all actions logged
12. Perform CRUD verification for **all entities**

---

### ⚡ Best Practices

* Always use **environment variables** for IDs and tokens
* Confirm every distribution **before** customer sales or refills
* Validate **enums**, **decimals**, and **required fields**
* Check **inventory levels** and **audit logs** after every major mutation
* Include **unauthorized access** tests to confirm security


