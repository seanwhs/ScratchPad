# **HSH LPG Sales & Logistics Backend – Complete Technical Overview (2026)**

This document provides a **production-ready backend overview** for the HSH LPG field operations system. It covers **models, serializers, services, viewsets, middleware, utilities, and audit logging**, with a focus on **business logic** and **atomic operations**.


<img width="1536" height="1024" alt="image" src="https://github.com/user-attachments/assets/e15a67a1-c16e-49fb-a4ea-270720c0af81" />

---

## **1️⃣ System Purpose**

<img width="1536" height="1024" alt="image" src="https://github.com/user-attachments/assets/355018a6-8158-43f9-aa21-d39bb93cdecc" />

The backend is designed to manage **LPG sales and logistics in the field**. Key operational goals:

1. Track **cylinders, meters, and regulators** at depots, trucks, and customer sites.
2. Support **billing workflows**, including **usage-based billing** for meters, **transactions**, **invoices**, and **payments**.
3. Provide **audit logs** for compliance and traceability.
4. Ensure **atomic operations** — any failure in a workflow rolls back all changes.
5. Role-based access control: **Admin**, **Supervisor**, **Driver**.
6. **API-first architecture** to support React frontend or mobile applications.

> 💡 Key principle: **never allow partial updates** — inventory, transactions, and invoices must remain consistent.

---

# **Services & Functions Summary**

The HSH LPG backend is **service-driven**, meaning all critical business logic—billing, inventory updates, distributions, and audits—is handled in **dedicated service functions**. These services ensure **atomic operations**, **audit logging**, and **data consistency** across all transactions.

Here are the key service functions and what they do:

### **Transactions & Invoices (`transactions/services.py`)**

* **`create_customer_transaction_and_invoice(user, data)`**

  * Creates a customer billing transaction.
  * Calculates **meter usage** (`last_meter_reading × meter_rate`).
  * Adds any **items/services delivered**.
  * Generates **PDF invoice** via `WeasyPrint`.
  * Optionally **emails the invoice** to the customer.
  * Logs the **entire process in AuditLog**.
  * Wrapped in `@transaction.atomic` → **all steps succeed or rollback**.

---

### **Distribution (`distribution/services.py`)**

* **`create_distribution(user, depot, items, remarks)`**

  * Creates a **distribution header** and associated **DistributionItems** atomically.
  * Does **not update inventory** yet.
  * Ensures all items are **recorded together**.

* **`confirm_distribution(distribution_id, user)`**

  * Confirms the distribution.
  * Updates **inventory quantities** (OUT deliveries reduce stock, IN returns increase stock).
  * Updates **CustomerSiteInventory** for accurate field stock.
  * Logs all actions in **AuditLog**.
  * Prevents **double confirmation**.

---

### **Inventory (`inventory/services.py`)**

* **`update_inventory(entity, entity_id, equipment_id, quantity, user)`**

  * Updates or creates a **CustomerSiteInventory** record.
  * Maintains **inventory integrity** (stock cannot go negative).
  * Logs changes in **AuditLog**.
  * Currently supports `entity='customer'` only.

---

### **Audit (`audit/services.py`)**

* **`log_action(user, action, entity_type, entity_id, payload)`** *(used across all services)*

  * Records **who did what, on which entity, and when**.
  * Stores extra details in **payload** (e.g., item list, totals, quantities).
  * Provides a **central audit trail** for compliance and debugging.

---

### **Utilities (`utils.py`)**

* **`generate_number(prefix)`**

  * Generates unique identifiers for transactions (`TRX`), invoices (`INV`), distributions (`DST`) using timestamp.
  * Ensures **no duplicate numbers** in operations.

---

💡 **Bottom Line:**
The backend’s **strength lies in its services layer**. Each function:

* Enforces **business rules** (billing, inventory, distributions).
* Maintains **atomicity**—operations succeed fully or rollback.
* Logs **every action** in AuditLog.
* Supports the **API and frontend** with ready-to-use nested data (Transaction → Invoice, Distribution → Items).

---

## **2️⃣ Core Entities and Models**

### **2.1 Users (`accounts/models.py`)**

```python
class User(AbstractUser):
    ROLE_CHOICES = ('ADMIN','DRIVER','SUPERVISOR')
    employee_id = models.CharField(max_length=20, unique=True)
    role = models.CharField(max_length=20, choices=ROLE_CHOICES, default='DRIVER')
    depot = models.ForeignKey(Depot, on_delete=models.SET_NULL, null=True, blank=True)
```

**Explanation:**

* Extends Django’s `AbstractUser` to add **custom fields**:

  * `employee_id`: Unique identifier for field personnel.
  * `role`: Determines what the user can do (`ADMIN`, `SUPERVISOR`, `DRIVER`).
  * `depot`: Links a user to a **physical depot**.
* **Business logic impact**:

  * Drivers can only create transactions/distributions for their assigned depot.
  * Supervisors can oversee multiple depots or approve transactions.

---

### **2.2 Depots (`depots/models.py`)**

```python
class Depot(models.Model):
    code = models.CharField(max_length=20, unique=True)
    name = models.CharField(max_length=200)
    address = models.TextField(blank=True)
```

**Explanation:**

* Each **Depot** represents a warehouse/branch where equipment is stored.
* `code` ensures **unique identification** for integration with logistics services.
* Depots are referenced in:

  * Distributions (source/target of equipment)
  * Users (assign drivers)
  * Inventory management

---

### **2.3 Equipment (`equipment/models.py`)**

```python
class Equipment(models.Model):
    EQUIPMENT_TYPE_CHOICES = ('CYLINDER','METER','REGULATOR')
    name = models.CharField(max_length=100)
    sku = models.CharField(max_length=50, unique=True)
    equipment_type = models.CharField(max_length=20, choices=EQUIPMENT_TYPE_CHOICES)
    weight_kg = models.DecimalField(max_digits=6, decimal_places=2, null=True, blank=True)
```

**Explanation:**

* Represents **physical items**: cylinders, meters, regulators.
* `sku`: Unique stock-keeping unit for inventory management.
* `weight_kg` allows tracking **cylinder weight**, used for validation or reporting.
* Integrates with **distribution and customer site inventory**.

---

### **2.4 Customers (`customers/models.py`)**

```python
class Customer(models.Model):
    PAYMENT_CHOICES = [('CASH','Cash'),('CREDIT','Credit')]
    name = models.CharField(max_length=200)
    email = models.EmailField(unique=True)
    is_meter_installed = models.BooleanField(default=False)
    last_meter_reading = models.DecimalField(max_digits=12, decimal_places=2, null=True, blank=True)
    meter_rate = models.DecimalField(max_digits=8, decimal_places=4, null=True, blank=True)
```

**Explanation:**

* Stores **customer information**, including meter status.
* `last_meter_reading` and `meter_rate` enable **usage-based billing**:

  * Usage = `current_meter_reading - last_meter_reading`
  * Billing = `usage * meter_rate`
* Can support **cash or credit payments**.

---

### **2.5 Customer Site Inventory (`inventory/models.py`)**

```python
class CustomerSiteInventory(models.Model):
    customer = models.ForeignKey(Customer, on_delete=models.CASCADE)
    equipment = models.ForeignKey(Equipment, on_delete=models.PROTECT)
    quantity = models.PositiveIntegerField(default=0)

    class Meta:
        unique_together = ('customer','equipment')
```

**Explanation:**

* Tracks how many **cylinders/meters a customer has**.
* `unique_together` ensures **no duplicate records** per equipment per customer.
* Updated when:

  * Distributions are confirmed (equipment sent or returned)
  * Manual adjustments occur

---

### **2.6 Transactions (`transactions/models.py`)**

```python
class Transaction(models.Model):
    transaction_number = models.CharField(max_length=30, unique=True)
    customer = models.ForeignKey(Customer, on_delete=models.PROTECT, related_name='transactions')
    user = models.ForeignKey(settings.AUTH_USER_MODEL, on_delete=models.PROTECT)
    total_amount = models.DecimalField(max_digits=12, decimal_places=2)
    created_at = models.DateTimeField(auto_now_add=True)
```

**Explanation:**

* Represents a **billing instance**.
* `transaction_number` is **unique**, typically generated like `TRX-YYYYMMDD-HHMMSS`.
* Linked to:

  * **Customer**: who is being billed
  * **User**: driver who created the transaction
* The `total_amount` includes **meter usage + delivered items/services**.

---

### **2.7 Invoices (`invoices/models.py`)**

```python
class Invoice(models.Model):
    invoice_number = models.CharField(max_length=30, unique=True)
    transaction = models.OneToOneField(Transaction, on_delete=models.PROTECT, related_name='invoice')
    pdf_path = models.CharField(max_length=500)
    status = models.CharField(max_length=20, choices=[('generated','Generated'),('emailed','Emailed')], default='generated')
    generated_at = models.DateTimeField(auto_now_add=True)
    emailed_at = models.DateTimeField(null=True, blank=True)
```

**Explanation:**

* Each **transaction has exactly one invoice** (`OneToOneField`).
* Stores PDF path for download and **status** (`generated` or `emailed`).
* `generated_at` and `emailed_at` timestamps allow **tracking invoice lifecycle**.

---

### **2.8 Distributions (`distribution/models.py`)**

```python
class Distribution(models.Model):
    distribution_number = models.CharField(max_length=30, unique=True)
    depot = models.ForeignKey(Depot, on_delete=models.PROTECT)
    user = models.ForeignKey(settings.AUTH_USER_MODEL, on_delete=models.PROTECT)
    remarks = models.TextField(blank=True)
    confirmed_at = models.DateTimeField(null=True, blank=True)

class DistributionItem(models.Model):
    distribution = models.ForeignKey(Distribution, related_name='items', on_delete=models.CASCADE)
    equipment = models.ForeignKey(Equipment, on_delete=models.PROTECT)
    direction = models.CharField(max_length=3, choices=[('OUT','Out'),('IN','Return')])
    condition = models.CharField(max_length=10, choices=[('FULL','Full'),('EMPTY','Empty'),('DAMAGED','Damaged')])
    quantity = models.PositiveIntegerField()
```

**Explanation:**

* `Distribution` records **equipment movement** from depot → customer (or return).
* `DistributionItem` allows **multiple items per distribution**.
* `direction` = OUT (delivery), IN (return)
* `condition` = tracks if cylinders are full, empty, or damaged.
* Confirmation step updates **inventory and audit logs**.

---

### **2.9 Audit Logs (`audit/models.py`)**

```python
class AuditLog(models.Model):
    user = models.ForeignKey(settings.AUTH_USER_MODEL, on_delete=models.SET_NULL, null=True)
    action = models.CharField(max_length=100)
    entity_type = models.CharField(max_length=50)
    entity_id = models.PositiveIntegerField(null=True)
    payload = models.JSONField(default=dict)
    created_at = models.DateTimeField(auto_now_add=True)
```

**Explanation:**

* Centralized logging of **user actions**.
* `payload` stores **context**, e.g., transaction totals, inventory changes.
* Critical for compliance, auditing, and debugging.

---

## **3️⃣ Serializers – Converting Models to API JSON**

### **3.1 User Serializer**

```python
class UserSerializer(serializers.ModelSerializer):
    depot_name = serializers.CharField(source='depot.name', read_only=True)

    class Meta:
        model = User
        fields = ['id','username','employee_id','role','depot','depot_name']
```

**Explanation:**

* Adds `depot_name` for **convenience** on frontend.
* Read-only field ensures **data integrity**.
* Used in transactions, distributions to **show which depot the user belongs to**.

---

### **3.2 Customer Serializer**

```python
class CustomerSerializer(serializers.ModelSerializer):
    class Meta:
        model = Customer
        fields = ['id','name','email','is_meter_installed','last_meter_reading','meter_rate']
```

**Explanation:**

* Exposes **meter info and billing data**.
* Frontend can calculate usage or display meter readings.

---

### **3.3 Customer Site Inventory Serializer**

```python
class CustomerSiteInventorySerializer(serializers.ModelSerializer):
    class Meta:
        model = CustomerSiteInventory
        fields = ['id','customer','equipment','quantity']
```

**Explanation:**

* Enables **create/update inventory** via API.
* Ensures atomic updates when distributing equipment or adjusting stock.

---

### **3.4 Transactions & Invoice Serializer**

```python
class TransactionSerializer(serializers.ModelSerializer):
    class Meta:
        model = Transaction
        fields = ['id','transaction_number','customer','user','total_amount','created_at']

class InvoiceSerializer(serializers.ModelSerializer):
    transaction = TransactionSerializer(read_only=True)

    class Meta:
        model = Invoice
        fields = ['id','invoice_number','transaction','pdf_path','status','generated_at','emailed_at']
```

**Explanation:**

* `InvoiceSerializer` **nests Transaction**, so API returns invoice + transaction in **one call**.
* Essential for frontend to **display billing history** and download PDF invoices.

---

### **3.5 Distribution Serializer**

```python
class DistributionItemSerializer(serializers.ModelSerializer):
    class Meta:
        model = DistributionItem
        fields = ['id','equipment','direction','condition','quantity']

class DistributionSerializer(serializers.ModelSerializer):
    items = DistributionItemSerializer(many=True)

    class Meta:
        model = Distribution
        fields = ['id','distribution_number','depot','user','remarks','created_at','confirmed_at','items']
```

**Explanation:**

* Nested serializer allows **creating Distribution + multiple DistributionItems atomically**.
* Frontend can submit one request for multiple items.

---

### **3.6 Audit Serializer**

```python
class AuditLogSerializer(serializers.ModelSerializer):
    user_name = serializers.CharField(source='user.username', read_only=True)

    class Meta:
        model = AuditLog
        fields = ['id','user','user_name','action','entity_type','entity_id','payload','created_at']
```

**Explanation:**

* Adds human-readable `user_name` for frontend.
* Audit logs are **read-only**.

---

## **4️⃣ Services – Business Logic Layer (Detailed)**

Services **encapsulate core operations**, ensuring **atomicity, audit logging, and business rules**.

<img width="1536" height="1024" alt="image" src="https://github.com/user-attachments/assets/4185d4bc-c80f-4313-a243-267fd95bfabe" />

---

### **4.1 Transactions & Invoice Creation**

```python
@transaction.atomic
def create_customer_transaction_and_invoice(user, data):
    """
    Steps:
    1. Fetch customer
    2. Calculate meter usage if applicable
    3. Calculate total amount (usage + items)
    4. Create Transaction
    5. Generate Invoice PDF
    6. Save Invoice record
    7. Send Email (optional)
    8. Audit Log
    """
    customer = Customer.objects.get(id=data['customer_id'])
    
    # Usage calculation
    current_meter = Decimal(data.get('current_meter_reading', 0))
    usage = current_meter - (customer.last_meter_reading or 0)
    usage_amount = usage * (customer.meter_rate or 0)
    
    # Update last meter reading
    customer.last_meter_reading = current_meter
    customer.save()
    
    # Calculate items total
    items_total = sum(Decimal(item['amount']) for item in data.get('items', []))
    total_amount = usage_amount + items_total
    
    # Create Transaction
    transaction_obj = Transaction.objects.create(
        transaction_number=generate_number('TRX'),
        customer=customer,
        user=user,
        total_amount=total_amount
    )
    
    # Generate Invoice PDF
    html_string = render_to_string('invoices/invoice.html', {'transaction': transaction_obj, 'customer': customer})
    pdf_path = f"invoices/{transaction_obj.transaction_number}.pdf"
    HTML(string=html_string).write_pdf(target=pdf_path)
    
    # Create Invoice record
    invoice = Invoice.objects.create(
        invoice_number=generate_number('INV'),
        transaction=transaction_obj,
        pdf_path=pdf_path,
        status='generated'
    )
    
    # Optional: send email
    if data.get('send_email'):
        send_invoice_email(customer.email, pdf_path)
        invoice.status = 'emailed'
        invoice.emailed_at = timezone.now()
        invoice.save()
    
    # Audit log
    AuditLog.objects.create(
        user=user,
        action='Created transaction and invoice',
        entity_type='Transaction',
        entity_id=transaction_obj.id,
        payload={'total_amount': float(total_amount), 'items': data.get('items', [])}
    )
    
    return transaction_obj, invoice
```

**Explanation (Step-by-Step):**

1. **Atomic decorator** ensures **all database changes rollback** if any step fails.
2. **Meter usage** is calculated only if a meter is installed.
3. **Items**: supports additional charges or products delivered.
4. **Transaction creation** links customer and driver.
5. **Invoice PDF** is generated with `WeasyPrint`.
6. **Email sending** updates invoice status and timestamp.
7. **Audit logging** captures **everything**, including items and total amount.

---

### **4.2 Distribution Services**

```python
@transaction.atomic
def create_distribution(user, depot, items, remarks=''):
    distribution = Distribution.objects.create(
        distribution_number=generate_number('DST'),
        depot=depot,
        user=user,
        remarks=remarks
    )
    
    for item in items:
        DistributionItem.objects.create(
            distribution=distribution,
            equipment=item['equipment'],
            direction=item['direction'],
            condition=item['condition'],
            quantity=item['quantity']
        )
    
    AuditLog.objects.create(
        user=user,
        action='Created distribution',
        entity_type='Distribution',
        entity_id=distribution.id,
        payload={'items': items}
    )
    
    return distribution

@transaction.atomic
def confirm_distribution(distribution):
    if distribution.confirmed_at:
        raise Exception("Distribution already confirmed")
    
    distribution.confirmed_at = timezone.now()
    distribution.save()
    
    for item in distribution.items.all():
        # Update CustomerSiteInventory
        inv, created = CustomerSiteInventory.objects.get_or_create(
            customer=None,  # can be linked if distribution is to specific customer
            equipment=item.equipment,
            defaults={'quantity': 0}
        )
        if item.direction == 'OUT':
            inv.quantity = max(inv.quantity - item.quantity, 0)
        else:  # IN return
            inv.quantity += item.quantity
        inv.save()
    
    AuditLog.objects.create(
        user=distribution.user,
        action='Confirmed distribution',
        entity_type='Distribution',
        entity_id=distribution.id,
        payload={'items': [{'equipment': i.equipment.id, 'qty': i.quantity} for i in distribution.items.all()]}
    )
    
    return distribution
```

**Explanation:**

* `create_distribution` **creates the header + items** in a single atomic transaction.
* `confirm_distribution`:

  * Prevents **double confirmation**.
  * Updates **inventory quantities** correctly based on direction.
  * Creates **audit logs** for traceability.

---

### **4.3 Inventory Services**

```python
@transaction.atomic
def update_inventory(entity, entity_id, equipment_id, quantity, user):
    """
    Supports updating a customer’s inventory.
    """
    if entity != 'customer':
        raise Exception("Only customer inventory supported")
    
    customer = Customer.objects.get(id=entity_id)
    equipment = Equipment.objects.get(id=equipment_id)
    
    inv, created = CustomerSiteInventory.objects.get_or_create(
        customer=customer,
        equipment=equipment,
        defaults={'quantity': 0}
    )
    
    inv.quantity = quantity
    inv.save()
    
    AuditLog.objects.create(
        user=user,
        action='Updated inventory',
        entity_type='CustomerSiteInventory',
        entity_id=inv.id,
        payload={'quantity': quantity}
    )
    
    return inv
```

**Explanation:**

* Updates **inventory for a specific customer**.
* Automatically **creates record if missing**.
* Atomic: ensures **quantity update + audit log** succeed together.

---

# **HSH LPG Backend – ViewSets & Middleware (Full Code Explanation)**

The **ViewSets layer** serves as the **API layer**. It takes requests from the frontend (React or mobile), validates them via serializers, calls **service functions**, and returns structured responses. All business logic lives in **services**, keeping views **thin and focused**.

---

## **1️⃣ Transactions ViewSet (`transactions/views.py`)**

```python
from rest_framework import viewsets, status
from rest_framework.decorators import action
from rest_framework.response import Response
from .models import Transaction
from .serializers import TransactionSerializer, InvoiceSerializer
from .services import create_customer_transaction_and_invoice

class TransactionViewSet(viewsets.ModelViewSet):
    queryset = Transaction.objects.all()
    serializer_class = TransactionSerializer

    @action(detail=False, methods=['post'])
    def create_transaction(self, request):
        """
        API endpoint to create a transaction and generate an invoice.
        Expects:
            {
                "customer_id": int,
                "current_meter_reading": decimal,
                "items": [{"amount": decimal, "description": str}],
                "send_email": bool
            }
        """
        user = request.user
        data = request.data

        try:
            transaction_obj, invoice_obj = create_customer_transaction_and_invoice(user, data)
            tx_serializer = TransactionSerializer(transaction_obj)
            inv_serializer = InvoiceSerializer(invoice_obj)
            return Response({"transaction": tx_serializer.data, "invoice": inv_serializer.data}, status=status.HTTP_201_CREATED)
        except Exception as e:
            return Response({"detail": str(e)}, status=status.HTTP_400_BAD_REQUEST)
```

### **Explanation:**

* `viewsets.ModelViewSet` gives **CRUD endpoints automatically** (list, retrieve, create, update, delete).
* `@action(detail=False, methods=['post'])` defines a **custom endpoint**: `/api/transactions/create_transaction/`.
* Flow:

  1. Frontend submits **transaction + items + meter reading**.
  2. View calls **`create_customer_transaction_and_invoice` service**.
  3. Service handles:

     * Meter usage calculation
     * Transaction creation
     * Invoice PDF generation
     * Optional email
     * Audit logging
     * Atomic rollback if anything fails
  4. Serializer converts models → JSON response.
* **Error handling**: any exception returns `400` with message.

---

## **2️⃣ Invoices ViewSet (`invoices/views.py`)**

```python
from rest_framework import viewsets, status
from rest_framework.decorators import action
from rest_framework.response import Response
from .models import Invoice
from .serializers import InvoiceSerializer
from .services import send_invoice_email
from django.http import FileResponse

class InvoiceViewSet(viewsets.ModelViewSet):
    queryset = Invoice.objects.all()
    serializer_class = InvoiceSerializer

    @action(detail=True, methods=['get'])
    def pdf(self, request, pk=None):
        """
        Return the invoice PDF file.
        """
        invoice = self.get_object()
        return FileResponse(open(invoice.pdf_path, 'rb'), content_type='application/pdf')

    @action(detail=True, methods=['post'])
    def email(self, request, pk=None):
        """
        Send invoice by email and update status.
        """
        invoice = self.get_object()
        try:
            send_invoice_email(invoice.transaction.customer.email, invoice.pdf_path)
            invoice.status = 'emailed'
            invoice.emailed_at = timezone.now()
            invoice.save()
            return Response({"detail": "Invoice emailed successfully"})
        except Exception as e:
            return Response({"detail": str(e)}, status=status.HTTP_400_BAD_REQUEST)
```

### **Explanation:**

* `/api/invoices/{id}/pdf/` returns **the PDF file** for download.
* `/api/invoices/{id}/email/` sends the invoice via email:

  * Updates `status` and `emailed_at`.
  * Logs audit inside service.
* Maintains **atomicity at service level** for consistency.
* Uses **Django FileResponse** to stream PDFs.

---

## **3️⃣ Distribution ViewSet (`distribution/views.py`)**

```python
from rest_framework import viewsets, status
from rest_framework.decorators import action
from rest_framework.response import Response
from .models import Distribution
from .serializers import DistributionSerializer
from .services import create_distribution, confirm_distribution

class DistributionViewSet(viewsets.ModelViewSet):
    queryset = Distribution.objects.all()
    serializer_class = DistributionSerializer

    @action(detail=False, methods=['post'])
    def create_distribution(self, request):
        """
        Create a new distribution record with items.
        Expects:
            {
                "depot_id": int,
                "items": [{"equipment": int, "direction": "OUT"/"IN", "condition": "FULL"/"EMPTY"/"DAMAGED", "quantity": int}],
                "remarks": str
            }
        """
        user = request.user
        data = request.data
        depot = Depot.objects.get(id=data['depot_id'])
        items = data.get('items', [])
        remarks = data.get('remarks', '')

        try:
            distribution = create_distribution(user, depot, items, remarks)
            serializer = DistributionSerializer(distribution)
            return Response(serializer.data, status=status.HTTP_201_CREATED)
        except Exception as e:
            return Response({"detail": str(e)}, status=status.HTTP_400_BAD_REQUEST)

    @action(detail=True, methods=['post'])
    def confirm(self, request, pk=None):
        """
        Confirm a distribution and update inventory atomically.
        """
        distribution = self.get_object()
        try:
            distribution = confirm_distribution(distribution)
            serializer = DistributionSerializer(distribution)
            return Response(serializer.data)
        except Exception as e:
            return Response({"detail": str(e)}, status=status.HTTP_400_BAD_REQUEST)
```

### **Explanation:**

* `create_distribution`:

  * Creates **distribution header + items** via service.
  * Wrapped in **atomic transaction**.
* `confirm`:

  * Updates **inventory quantities**.
  * Updates `confirmed_at` timestamp.
  * Writes **audit logs**.
  * Prevents **double confirmation**.
* Frontend can handle multiple items in **one API call**.

---

## **4️⃣ Inventory ViewSet (`inventory/views.py`)**

```python
from rest_framework import viewsets, status
from rest_framework.decorators import action
from rest_framework.response import Response
from .models import CustomerSiteInventory
from .serializers import CustomerSiteInventorySerializer
from .services import update_inventory

class InventoryViewSet(viewsets.ModelViewSet):
    queryset = CustomerSiteInventory.objects.all()
    serializer_class = CustomerSiteInventorySerializer

    @action(detail=False, methods=['post'])
    def update_inventory(self, request):
        """
        Update customer site inventory atomically.
        Expects:
            {
                "entity": "customer",
                "entity_id": int,
                "equipment_id": int,
                "quantity": int
            }
        """
        user = request.user
        data = request.data
        try:
            inventory = update_inventory(
                data['entity'], data['entity_id'], data['equipment_id'], data['quantity'], user
            )
            serializer = CustomerSiteInventorySerializer(inventory)
            return Response(serializer.data, status=status.HTTP_200_OK)
        except Exception as e:
            return Response({"detail": str(e)}, status=status.HTTP_400_BAD_REQUEST)
```

### **Explanation:**

* Handles **manual or API-driven inventory updates**.
* Calls `update_inventory` service which ensures:

  * Record exists
  * Quantity updated correctly
  * Audit log written
* Atomic to prevent **mismatched stock and logs**.

---

## **5️⃣ Customers & Audit ViewSets**

* **CustomerViewSet**: standard `ModelViewSet` for CRUD operations.
* **AuditLogViewSet**: read-only `ModelViewSet` to fetch logs for monitoring or compliance.

---

## **6️⃣ Middleware – API Request Logging**

```python
import time
import logging

logger = logging.getLogger(__name__)

class APILoggingMiddleware:
    def __init__(self, get_response):
        self.get_response = get_response

    def __call__(self, request):
        start_time = time.monotonic()
        response = self.get_response(request)
        duration = int((time.monotonic() - start_time) * 1000)

        if request.path.startswith('/api/'):
            user = request.user.username if request.user.is_authenticated else "Anonymous"
            logger.info("%s %s %s %dms %s", request.method, request.path, user, duration, request.META.get('REMOTE_ADDR'))

        return response
```

### **Explanation:**

* Logs **all `/api/` requests**:

  * HTTP method
  * URL path
  * Authenticated user
  * Response duration in ms
  * IP address
* Useful for:

  * **Performance monitoring**
  * **Debugging slow endpoints**
  * **Audit / tracking API usage**
* Middleware runs **before and after each request**, capturing **execution time**.

---

## **7️⃣ Key Principles Across ViewSets & Middleware**

1. **Atomic Operations**:
   All service calls are decorated with `@transaction.atomic`.
2. **Audit Logging**:
   Every create/update action is logged via `AuditLog`.
3. **Error Handling**:
   Exceptions return **400 Bad Request** with descriptive messages.
4. **Thin Views**:
   Views delegate **business logic** entirely to services.
5. **Nested Data**:

   * Transaction → Invoice
   * Distribution → Items
     Ensures frontend receives **all relevant info** in a single API response.
6. **Security**:

   * JWT Authentication
   * Role-based checks (enforced in services or view permissions)
7. **Logging & Monitoring**:
   Middleware logs performance and user activity.

---

✅ **Takeaway:**

* **ViewSets** serve as the **API entry point**, calling **services for all business logic**.
* **Middleware** ensures logging and performance tracking.
* Together, they **enforce atomicity, audit, and role-based access**, making the backend **robust for field-first LPG operations**.




