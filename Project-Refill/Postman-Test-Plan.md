# 🧪 HSH LPG Postman Test Plan – Full Production Flow (2026)

This Postman test plan is **production-ready**, including environment variables, authentication, full CRUD workflow, inventory, distributions, transactions, invoices, and audit checks.

---

## **1️⃣ Environment Setup**

### Environment Variables

| Key               | Initial Value               | Description                     |
| ----------------- | --------------------------- | ------------------------------- |
| `BASE_URL`        | `http://127.0.0.1:8000/api` | API base URL including `/api`   |
| `USERNAME`        | `admin`                     | Login username                  |
| `PASSWORD`        | `your_password`             | Login password                  |
| `ACCESS_TOKEN`    |                             | JWT access token                |
| `REFRESH_TOKEN`   |                             | JWT refresh token               |
| `DEPOT_ID`        |                             | Created Depot ID                |
| `EQUIPMENT_ID`    |                             | Created Equipment (Cylinder) ID |
| `CUSTOMER_ID`     |                             | Created Customer ID             |
| `DISTRIBUTION_ID` |                             | Created Distribution ID         |
| `TRANSACTION_ID`  |                             | Created Transaction ID          |
| `INVOICE_ID`      |                             | Created Invoice ID              |

---

## **2️⃣ Request Headers (Global)**

For all requests **after login**:

```
Authorization: Bearer {{ACCESS_TOKEN}}
Content-Type: application/json
```

> ⚠️ Only login request does **not** require the token.

---

## **3️⃣ Requests Flow**

### **Step 1: JWT Authentication**

**POST** `{{BASE_URL}}/token/`

**Body:**

```json
{
  "username": "{{USERNAME}}",
  "password": "{{PASSWORD}}"
}
```

**Tests Tab:**

```js
if (pm.response.code === 200) {
    const json = pm.response.json();
    pm.environment.set("ACCESS_TOKEN", json.access);
    pm.environment.set("REFRESH_TOKEN", json.refresh);
}
```

> ✅ This token is required for all subsequent requests.

---

### **Step 2: Create Depot**

**POST** `{{BASE_URL}}/depots/`

**Body:**

```json
{
  "code": "DEPOT-SG01",
  "name": "Singapore Main Depot",
  "address": "50 Tuas South Ave 1, Singapore 637315"
}
```

**Tests Tab:**

```js
if (pm.response.code === 201) {
    pm.environment.set("DEPOT_ID", pm.response.json().id);
}
```

---

### **Step 3: Create Equipment (Cylinder)**

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

**Tests Tab:**

```js
if (pm.response.code === 201) {
    pm.environment.set("EQUIPMENT_ID", pm.response.json().id);
}
```

---

### **Step 4: Create Customer**

**POST** `{{BASE_URL}}/customers/`

**Body – Retail Customer:**

```json
{
  "name": "Test Retail Customer",
  "email": "test.retail@example.sg",
  "address": "Blk 123 Jurong East Ave 3 #05-12",
  "payment_type": "CASH",
  "is_meter_installed": false
}
```

**Body – Metered Customer (Optional):**

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

**Tests Tab:**

```js
if (pm.response.code === 201) {
    pm.environment.set("CUSTOMER_ID", pm.response.json().id);
}
```

---

### **Step 5: Update Inventory**

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

**Tests Tab:**

```js
pm.test("Inventory updated successfully", function() {
    pm.response.to.have.status(200);
    pm.expect(pm.response.json().quantity).to.eql(8);
});
```

**List Inventory (Optional Verification):**

```
GET {{BASE_URL}}/inventories/?customer_id={{CUSTOMER_ID}}
```

---

### **Step 6: Create Distribution (Unconfirmed)**

**POST** `{{BASE_URL}}/distributions/create_distribution/`

**Body:**

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

**Tests Tab:**

```js
if (pm.response.code === 201) {
    pm.environment.set("DISTRIBUTION_ID", pm.response.json().id);
}
```

---

### **Step 7: Confirm Distribution**

**POST** `{{BASE_URL}}/distributions/{{DISTRIBUTION_ID}}/confirm/`

**Body:** `{}`

**Tests Tab:**

```js
pm.test("Distribution confirmed", function() {
    pm.response.to.have.status(200);
    pm.expect(pm.response.json().status).to.eql("CONFIRMED");
});
```

> ✅ Inventory is updated atomically. OUT subtracts, IN adds.

---

### **Step 8: Create Transaction + Invoice**

**POST** `{{BASE_URL}}/transactions/create_transaction/`

**Body Variant A – Meter Only:**

```json
{
  "customer": {{CUSTOMER_ID}},
  "current_meter": 1256.50
}
```

**Body Variant B – Meter + Items Sold:**

```json
{
  "customer": {{CUSTOMER_ID}},
  "current_meter": 1489.00,
  "items": [
    {
      "rate": 28.50,
      "quantity": 2
    }
  ]
}
```

**Tests Tab:**

```js
if (pm.response.code === 201) {
    const json = pm.response.json();
    if (json.transaction?.id) pm.environment.set("TRANSACTION_ID", json.transaction.id);
    if (json.invoice?.id) pm.environment.set("INVOICE_ID", json.invoice.id);
}
```

---

### **Step 9: Invoice Actions**

#### **Download PDF**

```
GET {{BASE_URL}}/invoices/{{INVOICE_ID}}/pdf/
```

* Response type: `application/pdf`

#### **Email Invoice**

```
POST {{BASE_URL}}/invoices/{{INVOICE_ID}}/email/
```

**Body:** `{}`

**Tests Tab:**

```js
pm.test("Invoice emailed successfully", function() {
    pm.response.to.have.status(200);
    pm.expect(pm.response.json().status).to.eql("EMAILED");
});
```

---

### **Step 10: Audit Logs**

```
GET {{BASE_URL}}/audit/
```

* Already ordered `-created_at`
* Validate:

  * Distribution confirmation
  * Transaction creation
  * Invoice emailing
  * Inventory updates

---

## **🔹 Execution Order (Strict)**

1. Login → get tokens (`ACCESS_TOKEN`)
2. Create Depot → save `DEPOT_ID`
3. Create Equipment → save `EQUIPMENT_ID`
4. Create Customer → save `CUSTOMER_ID`
5. Update Inventory
6. Create Distribution → save `DISTRIBUTION_ID`
7. Confirm Distribution
8. Create Transaction → saves `TRANSACTION_ID` & `INVOICE_ID`
9. Download / Email Invoice
10. Check Audit Logs

> ⚠️ Deviating from this order may result in FK errors, 404, or inventory mismatches.

---

## **🔹 Best Practices**

* Use **Environment Variables** for all dynamic IDs.
* Always confirm distributions before creating transactions.
* Keep **Authorization header** consistent across requests.
* Check inventory and audit logs after each critical operation.
* Follow hyphenated REST URLs: `update-inventory`, `create-distribution`, `create_transaction`.

Do you want me to create that ready-to-import collection?
