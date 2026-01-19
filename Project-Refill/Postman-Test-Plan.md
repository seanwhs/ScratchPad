# 🧪 **Postman Testing Guide – HSH LPG Sales & Logistics MVP**

**Purpose:** Complete testing workflow for the **HSH LPG backend**, including **JWT auth, customers, inventory, distributions, transactions, and invoices**. Ready-to-copy Postman setup.

---

## 🎯 **1. Postman Environment Setup**

Create **Environment Variables** (Ctrl+Alt+E):

| **Variable**     | **Initial Value**           | **Purpose**                             |
| ---------------- | --------------------------- | --------------------------------------- |
| `BASE_URL`       | `http://127.0.0.1:8000/api` | API root                                |
| `ACCESS_TOKEN`   | `""`                        | JWT token (set after login)             |
| `REFRESH_TOKEN`  | `""`                        | Refresh token (set after login)         |
| `CUSTOMER_ID`    | `""`                        | Saved dynamically for transaction tests |
| `TRANSACTION_ID` | `""`                        | Saved dynamically for invoice tests     |
| `INVOICE_ID`     | `""`                        | Saved dynamically for invoice actions   |

---

## 🔐 **2. JWT Authentication Flow**

### **Step 1: Login → Get Tokens**

```
POST {{BASE_URL}}/token/
Content-Type: application/json

{
  "username": "admin",
  "password": "your_password"
}
```

**✅ Response (200 OK):**

```json
{
  "access": "eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9...",
  "refresh": "eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9..."
}
```

**→ Set `{{ACCESS_TOKEN}}` and `{{REFRESH_TOKEN}}`**

---

### **Step 2: Refresh Token**

```
POST {{BASE_URL}}/token/refresh/
Content-Type: application/json

{
  "refresh": "{{REFRESH_TOKEN}}"
}
```

**✅ Response:**

```json
{
  "access": "new_access_token_here"
}
```

---

## 🌐 **3. Customer API Tests**

**Headers for all protected requests:**

```
Authorization: Bearer {{ACCESS_TOKEN}}
Content-Type: application/json
```

### **List Customers (Protected)**

```
GET {{BASE_URL}}/customers/
```

**✅ 200 OK** → Array of customers

### **Create Customer**

```
POST {{BASE_URL}}/customers/
{
  "name": "Test Customer",
  "email": "testcustomer@example.com",
  "payment_type": "CASH"
}
```

**✅ 201 Created** → Response contains `id`, `name`, `email`
**→ Set `{{CUSTOMER_ID}}`** for future tests

### **Retrieve Customer**

```
GET {{BASE_URL}}/customers/{{CUSTOMER_ID}}/
```

**✅ 200 OK** → Verify fields

### **Update Customer**

```
PATCH {{BASE_URL}}/customers/{{CUSTOMER_ID}}/
{
  "payment_type": "CREDIT"
}
```

**✅ 200 OK** → Field updated

### **Delete Customer**

```
DELETE {{BASE_URL}}/customers/{{CUSTOMER_ID}}/
```

**✅ 204 No Content**

---

## 📦 **4. Inventory API Tests**

### **List Customer Inventory**

```
GET {{BASE_URL}}/inventories/?customer_id={{CUSTOMER_ID}}
```

**✅ 200 OK** → Array of inventory items

### **Update Inventory**

```
POST {{BASE_URL}}/inventories/update/
{
  "entity": "customer",
  "entity_id": {{CUSTOMER_ID}},
  "equipment_id": 1,
  "quantity": 10
}
```

**✅ 200 OK** → Verify quantity updated

---

## 🚚 **5. Distribution API Tests**

### **Create Distribution**

```
POST {{BASE_URL}}/distributions/
{
  "user_id": 1,
  "items": [
    {"depot_id": 1, "equipment_id": 2, "quantity": 5, "movement_type": "Collection"}
  ],
  "client_temp_id": "tmp-001"
}
```

**✅ 201 Created** → Response contains `distribution_number`

### **Confirm Distribution**

```
POST {{BASE_URL}}/distributions/{{DISTRIBUTION_ID}}/confirm/
```

**✅ 200 OK** → Inventory updated, status `confirmed`

---

## 💰 **6. Transaction API Tests**

### **Create Transaction**

```
POST {{BASE_URL}}/transactions/create_transaction/
{
  "user": 1,
  "customer": {{CUSTOMER_ID}},
  "current_meter": 1234,
  "items": [
    {"equipment_id": 2, "quantity": 10, "rate": 28.5, "type": "Delivery"}
  ],
  "client_temp_id": "tmp-001"
}
```

**✅ 201 Created** → Response contains `transaction_number` & `invoice`
**→ Set `{{TRANSACTION_ID}}` & `{{INVOICE_ID}}`**

### **Retrieve Transaction**

```
GET {{BASE_URL}}/transactions/{{TRANSACTION_ID}}/
```

**✅ 200 OK**

---

## 🧾 **7. Invoice API Tests**

### **Retrieve Invoice**

```
GET {{BASE_URL}}/invoices/{{INVOICE_ID}}/
```

**✅ 200 OK** → Includes transaction details

### **Download PDF**

```
GET {{BASE_URL}}/invoices/{{INVOICE_ID}}/pdf/
```

**✅ 200 OK**, Content-Type: application/pdf

### **Email Invoice**

```
POST {{BASE_URL}}/invoices/{{INVOICE_ID}}/email/
```

**✅ 200 OK** → `status: emailed`

---

## 📝 **8. Audit Log Tests**

### **List Audit Logs**

```
GET {{BASE_URL}}/audit/
```

**✅ 200 OK** → Each log includes `user_id`, `action`, `entity_type`

### **Retrieve Single Audit Log**

```
GET {{BASE_URL}}/audit/1/
```

**✅ 200 OK** → Details match expected operation

---

## 📋 **9. Endpoint Matrix**

| Endpoint                            | Method | Auth | Action                       |
| ----------------------------------- | ------ | ---- | ---------------------------- |
| `/customers/`                       | GET    | ✅    | List                         |
| `/customers/`                       | POST   | ✅    | Create                       |
| `/customers/<id>/`                  | GET    | ✅    | Retrieve                     |
| `/customers/<id>/`                  | PATCH  | ✅    | Update                       |
| `/customers/<id>/`                  | DELETE | ✅    | Delete                       |
| `/inventories/`                     | GET    | ✅    | List                         |
| `/inventories/update/`              | POST   | ✅    | Adjust Quantity              |
| `/distributions/`                   | POST   | ✅    | Create                       |
| `/distributions/<id>/confirm/`      | POST   | ✅    | Confirm                      |
| `/transactions/create_transaction/` | POST   | ✅    | Create Transaction + Invoice |
| `/transactions/<id>/`               | GET    | ✅    | Retrieve Transaction         |
| `/invoices/<id>/`                   | GET    | ✅    | Retrieve Invoice             |
| `/invoices/<id>/pdf/`               | GET    | ✅    | Download PDF                 |
| `/invoices/<id>/email/`             | POST   | ✅    | Send Email                   |
| `/audit/`                           | GET    | ✅    | List Audit Logs              |
| `/audit/<id>/`                      | GET    | ✅    | Retrieve Audit Log           |

---

## ⚡ **10. Pro Tips**

1. **Pre-request Script** (auto-set JWT):

```javascript
pm.request.headers.add({
    key: 'Authorization',
    value: 'Bearer ' + pm.environment.get('ACCESS_TOKEN')
});
```

2. **Tests Script** (store IDs dynamically):

```javascript
if (pm.response.code === 201) {
    const jsonData = pm.response.json();
    if(jsonData.transaction) pm.environment.set("TRANSACTION_ID", jsonData.transaction.id);
    if(jsonData.invoice) pm.environment.set("INVOICE_ID", jsonData.invoice.id);
    if(jsonData.id) pm.environment.set("CUSTOMER_ID", jsonData.id);
}
```

3. **Chained Requests:** Use `{{CUSTOMER_ID}}`, `{{TRANSACTION_ID}}`, and `{{INVOICE_ID}}` across requests to simulate real workflows.

4. **Negative Testing:** Attempt invalid IDs, missing fields, negative quantities, or expired JWT to ensure proper error handling.


