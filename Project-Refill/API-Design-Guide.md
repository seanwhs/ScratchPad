# **HSH Sales System – API Design Guide**

**Audience:** Backend developers, Frontend engineers, QA
**Stack:** Django 5.1 + DRF + JWT + drf-spectacular (OpenAPI/Swagger)
**Goals:**

* Provide a **consistent, secure, maintainable API**
* Support **mobile-first UX**, offline sync, and transactional integrity
* Track **distributions, transactions, and client inventory**

---

## **1️⃣ API Design Principles**

1. **Resource-Oriented Design**

   * Every entity is a resource: `/customers/`, `/transactions/`, `/distributions/`, `/inventory/`
   * Follow REST conventions:

     * `GET` → list/retrieve
     * `POST` → create
     * `PUT/PATCH` → update
     * `DELETE` → delete

2. **Consistent Naming**

   * Use **plural nouns** for endpoints (`/transactions/` not `/transaction/`)
   * Use **action verbs only** when necessary (`/transactions/{id}/confirm/`)

3. **Atomic & Idempotent Operations**

   * Critical operations (distributions, transactions) wrapped in `@transaction.atomic`
   * Idempotency via unique number (`transaction_number` / `distribution_number`)

4. **Versioning**

   * URI versioning: `/api/v1/...`
   * Ensures backward-compatible future changes

5. **JWT Authentication**

   * Short-lived access token + refresh token
   * Frontend stores access token in memory; refresh token in secure storage

6. **Error Handling**

   * Standardized JSON format:

     ```json
     {
       "status": "error",
       "code": 400,
       "message": "Invalid quantity for equipment CYL 12.7kg",
       "details": { "quantity": ["Must be greater than zero"] }
     }
     ```
   * Use proper HTTP status codes: 200, 201, 400, 401, 403, 500

7. **Pagination**

   * Lists: distributions, transactions, customers
   * Use DRF’s `PageNumberPagination` or `LimitOffsetPagination`

8. **Filtering & Searching**

   * Filter by date, customer, depot, equipment type
   * Example: `/transactions/?customer=3&date_from=2026-01-01&date_to=2026-01-19`

9. **Offline & Sync Considerations**

   * All creation endpoints accept **client-generated temp IDs**
   * Server returns official ID + timestamp for reconciliation
   * Include `last_modified` for conflict resolution

---

## **2️⃣ Resource Endpoints**

### **2.1 Authentication**

| Endpoint                | Method | Payload Example                                 | Response Example                                                                     | Notes                                      |
| ----------------------- | ------ | ----------------------------------------------- | ------------------------------------------------------------------------------------ | ------------------------------------------ |
| `/api/v1/auth/login/`   | POST   | `{ "username": "driver01", "password": "xxx" }` | `{ "access": "...", "refresh": "...", "user": { "id": 1, "username": "driver01" } }` | Returns JWT tokens                         |
| `/api/v1/auth/refresh/` | POST   | `{ "refresh": "..." }`                          | `{ "access": "..." }`                                                                | Refresh access token                       |
| `/api/v1/auth/logout/`  | POST   | `{}`                                            | `{ "status": "logged_out" }`                                                         | Optional: server-side refresh token revoke |

---

### **2.2 Customers**

| Endpoint                     | Method | Description               |
| ---------------------------- | ------ | ------------------------- |
| `/customers/`                | GET    | List all customers        |
| `/customers/{id}/`           | GET    | Retrieve customer detail  |
| `/customers/{id}/inventory/` | GET    | Get client site inventory |

*Customer Object Example:*

```json
{
  "id": 3,
  "name": "Hock Soon Heng Factory",
  "email": "customer@example.com",
  "last_meter_reading": 1200,
  "meter_rate": 0.45,
  "is_meter_installed": true
}
```

---

### **2.3 Distributions**

| Endpoint                       | Method | Description                             |
| ------------------------------ | ------ | --------------------------------------- |
| `/distributions/`              | GET    | List distributions                      |
| `/distributions/`              | POST   | Create a new distribution               |
| `/distributions/{id}/confirm/` | POST   | Confirm distribution & update inventory |

*POST Payload Example:*

```json
{
  "user_id": 1,
  "items": [
    { "depot_id": 1, "equipment_id": 2, "quantity": 10, "movement_type": "Collection" },
    { "depot_id": 1, "equipment_id": 3, "quantity": 5, "movement_type": "Empty Return" }
  ],
  "client_temp_id": "tmp-001"
}
```

*Response Example:*

```json
{
  "distribution_number": "DIST-20260119-153045",
  "status": "pending",
  "total_collection": 10,
  "total_return": 5,
  "timestamp": "2026-01-19T15:30:45Z"
}
```

**Enums:**

* `movement_type`: `"Collection"`, `"Empty Return"`
* `status`: `"pending"`, `"confirmed"`

---

### **2.4 Transactions**

| Endpoint                      | Method | Description                                              |
| ----------------------------- | ------ | -------------------------------------------------------- |
| `/transactions/`              | GET    | List transactions                                        |
| `/transactions/`              | POST   | Create new transaction                                   |
| `/transactions/{id}/confirm/` | POST   | Confirm transaction → generate invoice, update inventory |

*POST Payload Example:*

```json
{
  "user_id": 1,
  "customer_id": 3,
  "current_meter": 1250,
  "items": [
    { "equipment_id": 2, "quantity": 10, "rate": 28.5, "type": "Delivery" },
    { "equipment_id": null, "quantity": 1, "rate": 50, "type": "Service", "description": "Installation" }
  ],
  "client_temp_id": "tmp-001"
}
```

*Response Example:*

```json
{
  "transaction_number": "TRX-20260119-153045",
  "invoice_number": "INV-20260119-153045",
  "total_amount": 875.00,
  "status": "confirmed",
  "pdf_url": "/media/invoices/TRX-20260119-153045.pdf",
  "emailed_at": "2026-01-19T15:45:00Z"
}
```

**Enums:**

* `item_type`: `"Usage"`, `"Delivery"`, `"Service"`
* `status`: `"pending"`, `"confirmed"`

---

### **2.5 Inventory**

**Depot Inventory**

| Endpoint                  | Method | Description                 |
| ------------------------- | ------ | --------------------------- |
| `/inventory/depots/{id}/` | GET    | Get current depot inventory |

**Client Inventory**

| Endpoint                     | Method | Description                  |
| ---------------------------- | ------ | ---------------------------- |
| `/inventory/customers/{id}/` | GET    | Get current client inventory |

*Response Example:*

```json
[
  { "equipment_id": 2, "equipment_name": "CYL 12.7kg", "quantity": 10 },
  { "equipment_id": 3, "equipment_name": "CYL 14kg", "quantity": 5 }
]
```

---

### **2.6 Audit Logs**

| Endpoint       | Method | Description           |
| -------------- | ------ | --------------------- |
| `/audit/`      | GET    | List audit logs       |
| `/audit/{id}/` | GET    | Retrieve specific log |

*Audit Object Example:*

```json
{
  "user_id": 1,
  "action": "TRANSACTION_CONFIRMED",
  "entity_type": "Transaction",
  "entity_id": 42,
  "payload": { "total": "875.00" },
  "timestamp": "2026-01-19T15:45:00Z"
}
```

---

### **2.7 Invoice Endpoints**

| Endpoint                | Method | Description           |
| ----------------------- | ------ | --------------------- |
| `/invoices/`            | GET    | List invoices         |
| `/invoices/{id}/pdf/`   | GET    | Download PDF          |
| `/invoices/{id}/email/` | POST   | Re-send invoice email |

**Enums:**

* `status`: `"generated"`, `"printed"`, `"emailed"`, `"paid"`

---

## **3️⃣ Offline & Sync Support**

* All POST endpoints accept **client_temp_id**
* Server returns official ID + timestamp → reconcile on next sync
* Use `last_modified` timestamps for conflict detection
* Delta fetch example:

```
GET /transactions/?updated_since=2026-01-19T00:00:00Z
```

* Returns only updated records since timestamp

---

## **4️⃣ Example Enums & Constants**

| Field               | Values                                            |
| ------------------- | ------------------------------------------------- |
| movement_type       | `"Collection"`, `"Empty Return"`                  |
| item_type           | `"Usage"`, `"Delivery"`, `"Service"`              |
| invoice_status      | `"generated"`, `"printed"`, `"emailed"`, `"paid"` |
| distribution_status | `"pending"`, `"confirmed"`                        |
| transaction_status  | `"pending"`, `"confirmed"`                        |

---

## **5️⃣ Notes for Frontend / Mobile**

1. Use `confirm` endpoints to finalize distribution/transaction → triggers inventory updates & invoice generation
2. Always store server `id` + timestamp for offline reconciliation
3. Handle errors gracefully, especially negative quantities
4. All responses include `unique_number` (`TRX-...`, `DIST-...`, `INV-...`)
5. Use enums consistently for dropdowns

---

## **6️⃣ DRF Viewset Skeleton Example**

```python
from rest_framework import viewsets
from rest_framework.decorators import action
from rest_framework.response import Response
from rest_framework.permissions import IsAuthenticated
from transactions.models import Transaction
from transactions.serializers import TransactionSerializer
from transactions.services import create_customer_transaction_and_invoice

class TransactionViewSet(viewsets.ModelViewSet):
    queryset = Transaction.objects.all()
    serializer_class = TransactionSerializer
    permission_classes = [IsAuthenticated]

    @action(detail=True, methods=['post'])
    def confirm(self, request, pk=None):
        tx = self.get_object()
        create_customer_transaction_and_invoice(request.user, tx)
        return Response({"status": "confirmed"})
```

---

✅ **Summary**

* **RESTful, versioned, resource-oriented API**
* **Atomic operations** → maintain inventory & transaction integrity
* **Offline-ready** → client_temp_id, last-write-wins, delta fetch
* **Traceable & auditable** → unique IDs, timestamps, audit logs
* **Swagger/OpenAPI** → frontend & QA clarity
* **Extensible for Phase 2** → PDF invoices, QuickBooks sync, advanced reporting

---

# **HSH Sales System – API Endpoint Map (MVP)**

```
                          ┌───────────────┐
                          │   Auth        │
                          │───────────────│
                          │ /auth/login/  │
                          │ /auth/refresh/│
                          │ /auth/logout/ │
                          └─────┬─────────┘
                                │
                                ▼
                     ┌───────────────────────┐
                     │      Users / Roles    │
                     └─────────┬────────────┘
                               │
       ┌───────────────────────┼────────────────────────┐
       ▼                       ▼                        ▼
┌─────────────┐          ┌───────────────┐        ┌───────────────┐
│ Customers   │          │ Distributions │        │ Transactions  │
│─────────────│          │───────────────│        │───────────────│
│ /customers/ │          │ /distributions/│       │ /transactions/│
│ /customers/ │          │ /distributions/│       │ /transactions/│
│ {id}/       │          │ {id}/confirm/ │       │ {id}/confirm/ │
│ {id}/inv/   │          │               │       │               │
└─────┬───────┘          └─────┬─────────┘       └─────┬─────────┘
      │                        │                        │
      ▼                        ▼                        ▼
┌───────────────┐        ┌───────────────┐        ┌───────────────┐
│ ClientInventory│       │ DepotInventory│        │ Invoice       │
│ /inventory/    │       │ /inventory/depots/{id}/│ /invoices/   │
│ /customers/{id}/│      │               │        │ {id}/pdf/    │
└───────────────┘        └───────────────┘        │ {id}/email/  │
                                                     └─────┬───────┘
                                                           │
                                                           ▼
                                                      ┌─────────────┐
                                                      │ Audit Logs  │
                                                      │ /audit/     │
                                                      │ /audit/{id}/│
                                                      └─────────────┘
```

---

### **Key Notes on the Map**

1. **Auth layer** → protects all resources with JWT
2. **Customers** → read-only in MVP, can fetch inventory per client
3. **Distributions** → create & confirm, atomically updates depot inventory + audit logs
4. **Transactions** → create & confirm, generates invoice, updates client inventory, triggers audit logs
5. **Invoice endpoints** → download PDF, re-send email
6. **Inventory endpoints** → depot & client views for real-time stock checks
7. **Audit logs** → tracks all critical actions (distribution/transaction confirmations)
8. **Offline flow** → POST endpoints accept `client_temp_id`, confirm endpoints trigger official ID assignment

---

### **Visual Flow Example – Offline + Sync**

```
Frontend Device
  ┌──────────────┐
  │ User Action  │
  └─────┬────────┘
        │ (POST with client_temp_id)
        ▼
  ┌──────────────┐
  │ Offline Queue│
  └─────┬────────┘
        │ (sync when online)
        ▼
  ┌──────────────┐
  │ API Server   │
  │ Django + DRF │
  └─────┬────────┘
        │
        ▼
  ┌──────────────┐
  │ MySQL DB     │
  └─────┬────────┘
        │
        ▼
  ┌──────────────┐
  │ Audit / PDF  │
  └──────────────┘
        │
        ▼
Frontend Confirmation / Receipt
```

✅ **Highlights:**

* Every critical endpoint → inventory update + audit log + optional PDF/email
* Offline devices → client_temp_id ensures no duplication / id mismatch
* All resources are **traceable** using unique IDs (`TRX-…`, `DIST-…`, `INV-…`)

---

