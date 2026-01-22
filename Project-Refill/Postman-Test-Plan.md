# 🧪 HSH LPG – Postman Test Plan  
Full Production-like Flow (2026)

Test sequence covering authentication, CRUD operations, inventory movements, distributions, customer transactions, invoice generation & delivery, and audit trail verification.

---

## 1. Environment Setup

### Required Environment Variables

| Variable          | Initial / Example Value              | Description                                | Set automatically? |
|-------------------|--------------------------------------|--------------------------------------------|--------------------|
| `BASE_URL`        | `http://127.0.0.1:8000/api`          | API base path (include `/api`)             | No                 |
| `USERNAME`        | `admin`                              | Superuser or test user login               | No                 |
| `PASSWORD`        | `your_password`                      | Corresponding password                     | No                 |
| `ACCESS_TOKEN`    | (empty at start)                     | JWT access token                           | Yes – after login  |
| `REFRESH_TOKEN`   | (empty at start)                     | JWT refresh token                          | Yes – after login  |
| `DEPOT_ID`        | (empty)                              | ID of created depot                        | Yes                |
| `EQUIPMENT_ID`    | (empty)                              | ID of created LPG cylinder                 | Yes                |
| `CUSTOMER_ID`     | (empty)                              | ID of created test customer                | Yes                |
| `DISTRIBUTION_ID` | (empty)                              | ID of created distribution run             | Yes                |
| `TRANSACTION_ID`  | (empty)                              | ID of latest customer transaction          | Yes                |
| `INVOICE_ID`      | (empty)                              | ID of generated invoice                    | Yes                |

---

## 2. Global Request Headers (after login)

```
Authorization: Bearer {{ACCESS_TOKEN}}
Content-Type:  application/json
```

> ⚠️ The **login** request is the **only** request that does **not** need the `Authorization` header.

---

## 3. Step-by-Step Test Flow

### Step 1 – JWT Authentication (Obtain Tokens)

**Method:** `POST`  
**URL:** `{{BASE_URL}}/token/`

**Body (raw JSON):**

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
    console.log("JWT tokens saved successfully");
}
```

---

### Step 2 – Create Depot

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

### Step 3 – Create Equipment (14 kg Cylinder)

**Method:** `POST`  
**URL:** `{{BASE_URL}}/equipment/`

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
if (pm.response.code === 201) {
    pm.environment.set("EQUIPMENT_ID", pm.response.json().id);
}
```

---

### Step 4 – Create Test Customer

**Method:** `POST`  
**URL:** `{{BASE_URL}}/customers/`

**Variant A – Cash / Non-metered customer**

```json
{
  "name": "Test Retail Customer",
  "email": "test.retail@example.sg",
  "address": "Blk 123 Jurong East Ave 3 #05-12",
  "payment_type": "CASH",
  "is_meter_installed": false
}
```

**Variant B – Credit / Metered customer** (recommended for full flow testing)

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

**Tests (both variants):**

```js
if (pm.response.code === 201) {
    pm.environment.set("CUSTOMER_ID", pm.response.json().id);
}
```

---

### Step 5 – Set Initial Customer Inventory

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

**Optional verification request:**

```
GET {{BASE_URL}}/inventories/?customer_id={{CUSTOMER_ID}}
```

---

### Step 6 – Create Distribution (Delivery / Collection Run)

**Method:** `POST`  
**URL:** `{{BASE_URL}}/distributions/create_distribution/`

**Body example:**

```json
{
  "depot": {{DEPOT_ID}},
  "remarks": "Morning delivery run to Jurong",
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
if (pm.response.code === 201) {
    pm.environment.set("DISTRIBUTION_ID", pm.response.json().id);
}
```

---

### Step 7 – Confirm Distribution (triggers inventory update)

**Method:** `POST`  
**URL:** `{{BASE_URL}}/distributions/{{DISTRIBUTION_ID}}/confirm/`

**Body:** empty object → `{}`

**Tests:**

```js
pm.test("Distribution confirmed successfully", function () {
    pm.response.to.have.status(200);
    // Optional: check that confirmed_at is now set
    pm.expect(pm.response.json().confirmed_at).to.be.a("string");
});
```

> Inventory is updated **atomically**:  
> `OUT` → subtracts from depot stock  
> `IN` → adds to depot stock

---

### Step 8 – Create Customer Transaction + Invoice

**Method:** `POST`  
**URL:** `{{BASE_URL}}/transactions/create_transaction/`

**Variant A – Meter reading only**

```json
{
  "customer": {{CUSTOMER_ID}},
  "current_meter": 1256.50
}
```

**Variant B – Meter reading + cylinder sale**

```json
{
  "customer": {{CUSTOMER_ID}},
  "current_meter": 1489.00,
  "items": [
    {
      "equipment": {{EQUIPMENT_ID}},
      "quantity": 2,
      "rate": 28.50,
      "type": "SALE"
    }
  ]
}
```

**Tests (both variants):**

```js
if (pm.response.code === 201) {
    const json = pm.response.json();
    if (json.transaction?.id) pm.environment.set("TRANSACTION_ID", json.transaction.id);
    if (json.invoice?.id)    pm.environment.set("INVOICE_ID", json.invoice.id);
}
```

---

### Step 9 – Invoice Actions

#### 9.1 Download PDF

**Method:** `GET`  
**URL:** `{{BASE_URL}}/invoices/{{INVOICE_ID}}/pdf/`

→ Expect binary PDF response

#### 9.2 Send Invoice by Email

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

### Step 10 – Verify Audit Trail

**Method:** `GET`  
**URL:** `{{BASE_URL}}/audit/`

- Results are ordered by `-timestamp` (newest first)
- Look for entries like:
  - `DISTRIBUTION_CREATED` / `DISTRIBUTION_CONFIRMED`
  - `TX_CREATED`
  - `MANUAL_INVENTORY_CORRECTION`
  - `METER_UPDATE`
  - `Email Invoice`

---

## Recommended Strict Execution Order

1. Obtain JWT tokens  
2. Create Depot → save `DEPOT_ID`  
3. Create Equipment → save `EQUIPMENT_ID`  
4. Create Customer → save `CUSTOMER_ID`  
5. Set customer inventory  
6. Create Distribution → save `DISTRIBUTION_ID`  
7. Confirm Distribution  
8. Create Transaction (+ auto invoice) → save IDs  
9. Download / Email invoice  
10. Review audit logs

> ⚠️ Deviating from this sequence will likely cause foreign key errors, 404s or inventory inconsistencies.

---

## Best Practices & Tips

- Always use **environment variables** for IDs and tokens  
- **Confirm** every distribution before selling/refilling from it  
- Keep the `Authorization` header consistent after login  
- After each major mutation: check inventory levels + audit log  
- Use hyphenated action endpoints:  
  `update-inventory` • `create_distribution` • `create_transaction`

