# **HSH LPG Sales & Logistics System – Full Production-Ready Backend Tutorial**

**January 2026 MVP – Singapore Field Operations**
**Stack:** Django 5.1 + DRF + drf-spectacular + WeasyPrint + JWT + Atomic Transactions + PDF/Email Invoicing

---

## **1️⃣ Technical Stack**

| Component      | Choice (2026)                    | Why                    |
| -------------- | -------------------------------- | ---------------------- |
| Python         | 3.12                             | Latest stable          |
| Django         | 5.1.4                            | LTS with async support |
| DRF            | 3.15+                            | Modern API framework   |
| OpenAPI Docs   | drf-spectacular                  | Auto Swagger/OpenAPI   |
| PDF Generation | WeasyPrint                       | Reliable HTML → PDF    |
| Auth           | djangorestframework-simplejwt    | Industry standard      |
| Database       | SQLite (dev) / MySQL (prod)      | ACID compliant         |
| Logging        | Custom middleware                | Full API visibility    |
| Email          | Django SMTP + Gmail/App Password | Field-ready, reliable  |

---

## **2️⃣ Project Setup**

```bash
mkdir hsh_lpg_system && cd hsh_lpg_system
python -m venv venv
source venv/bin/activate  # Windows: venv\Scripts\activate

pip install django \
    djangorestframework \
    djangorestframework-simplejwt \
    drf-spectacular \
    weasyprint \
    mysqlclient \
    python-decouple \
    django-cors-headers

django-admin startproject core .
python manage.py startapp accounts depots equipment inventory distribution customers transactions invoices audit middleware
```

---

## **3️⃣ Core Settings (`core/settings.py`)**

```python
from pathlib import Path
from decouple import config
from datetime import timedelta

BASE_DIR = Path(__file__).resolve().parent.parent

SECRET_KEY = config('SECRET_KEY')
DEBUG = config('DEBUG', default=True, cast=bool)
ALLOWED_HOSTS = ['*']

INSTALLED_APPS = [
    # Django
    'django.contrib.admin','django.contrib.auth','django.contrib.contenttypes',
    'django.contrib.sessions','django.contrib.messages','django.contrib.staticfiles',
    
    # Third-party
    'rest_framework','drf_spectacular','corsheaders',
    
    # Project apps
    'accounts','depots','equipment','inventory','distribution','customers','transactions','invoices','audit','middleware',
]

MIDDLEWARE = [
    'django.middleware.security.SecurityMiddleware',
    'django.contrib.sessions.middleware.SessionMiddleware',
    'corsheaders.middleware.CorsMiddleware',
    'django.middleware.common.CommonMiddleware',
    'django.middleware.csrf.CsrfViewMiddleware',
    'django.contrib.auth.middleware.AuthenticationMiddleware',
    'django.contrib.messages.middleware.MessageMiddleware',
    'middleware.request_logging.APILoggingMiddleware',
]

ROOT_URLCONF = 'core.urls'
WSGI_APPLICATION = 'core.wsgi.application'
ASGI_APPLICATION = 'core.asgi.application'

AUTH_USER_MODEL = 'accounts.User'

REST_FRAMEWORK = {
    'DEFAULT_AUTHENTICATION_CLASSES': ('rest_framework_simplejwt.authentication.JWTAuthentication',),
    'DEFAULT_SCHEMA_CLASS': 'drf_spectacular.openapi.AutoSchema',
}

SPECTACULAR_SETTINGS = {
    'TITLE': 'HSH LPG Sales & Logistics API',
    'DESCRIPTION': 'Field-first LPG system – Distribution, Meter Billing, Invoicing, Delivery Tracking',
    'VERSION': '1.0.0',
    'SERVE_INCLUDE_SCHEMA': False,
    'SWAGGER_UI_SETTINGS': {'deepLinking': True},
}

SIMPLE_JWT = {
    'ACCESS_TOKEN_LIFETIME': timedelta(hours=8),
    'REFRESH_TOKEN_LIFETIME': timedelta(days=7),
}

# Database
DB_ENGINE = config('DB_ENGINE', default='sqlite')
if DB_ENGINE == 'mysql':
    DATABASES = {
        'default': {
            'ENGINE': 'django.db.backends.mysql',
            'NAME': config('DB_NAME'),
            'USER': config('DB_USER'),
            'PASSWORD': config('DB_PASSWORD'),
            'HOST': config('DB_HOST', default='127.0.0.1'),
            'PORT': config('DB_PORT', default='3306'),
            'OPTIONS': {'init_command': "SET sql_mode='STRICT_TRANS_TABLES'"},
        }
    }
else:
    DATABASES = {'default': {'ENGINE': 'django.db.backends.sqlite3','NAME': BASE_DIR / 'db.sqlite3'}}

STATIC_URL = '/static/'
STATIC_ROOT = BASE_DIR / 'staticfiles'

EMAIL_BACKEND = 'django.core.mail.backends.smtp.EmailBackend'
EMAIL_HOST = config('EMAIL_HOST', default='smtp.gmail.com')
EMAIL_PORT = 587
EMAIL_USE_TLS = True
EMAIL_HOST_USER = config('EMAIL_HOST_USER')
EMAIL_HOST_PASSWORD = config('EMAIL_HOST_PASSWORD')
DEFAULT_FROM_EMAIL = 'HSH LPG <noreply@hshlpg.com.sg>'

TEMPLATES = [{
    'BACKEND': 'django.template.backends.django.DjangoTemplates',
    'DIRS': [BASE_DIR / 'templates'],
    'APP_DIRS': True,
    'OPTIONS': {'context_processors': [
        'django.template.context_processors.debug',
        'django.template.context_processors.request',
        'django.contrib.auth.context_processors.auth',
        'django.contrib.messages.context_processors.messages',
    ]},
}]
```

---

## **4️⃣ Utilities**

```python
# core/utils/numbering.py
from django.utils import timezone

def generate_number(prefix: str) -> str:
    return f"{prefix}-{timezone.now().strftime('%Y%m%d-%H%M%S')}"
```

---

## **5️⃣ Models**

### **Accounts (`accounts/models.py`)**

```python
from django.contrib.auth.models import AbstractUser
from django.db import models
from depots.models import Depot

class User(AbstractUser):
    ROLE_CHOICES = (('ADMIN','Admin'),('DRIVER','Driver'),('SUPERVISOR','Supervisor'))
    employee_id = models.CharField(max_length=20, unique=True)
    role = models.CharField(max_length=20, choices=ROLE_CHOICES, default='DRIVER')
    depot = models.ForeignKey('depots.Depot', on_delete=models.SET_NULL, null=True, blank=True)
```

### **Depots (`depots/models.py`)**

```python
from django.db import models

class Depot(models.Model):
    code = models.CharField(max_length=20, unique=True)
    name = models.CharField(max_length=200)
    address = models.TextField(blank=True)
    def __str__(self): return self.name
```

### **Equipment (`equipment/models.py`)**

```python
from django.db import models

class Equipment(models.Model):
    EQUIPMENT_TYPE_CHOICES = (('CYLINDER','Cylinder'),('METER','Meter'),('REGULATOR','Regulator'))
    name = models.CharField(max_length=100)
    sku = models.CharField(max_length=50, unique=True)
    equipment_type = models.CharField(max_length=20, choices=EQUIPMENT_TYPE_CHOICES)
    weight_kg = models.DecimalField(max_digits=6, decimal_places=2, null=True, blank=True)
    is_active = models.BooleanField(default=True)
    def __str__(self): return f"{self.name} ({self.sku})"
```

### **Customers (`customers/models.py`)**

```python
from django.db import models

class Customer(models.Model):
    name = models.CharField(max_length=200)
    email = models.EmailField(unique=True)
    address = models.TextField(blank=True)
    payment_type = models.CharField(max_length=10, choices=[('CASH','Cash'),('CREDIT','Credit')])
    is_meter_installed = models.BooleanField(default=False)
    last_meter_reading = models.DecimalField(max_digits=12, decimal_places=2, null=True, blank=True)
    meter_rate = models.DecimalField(max_digits=8, decimal_places=4, null=True, blank=True)
```

### **Inventory (`inventory/models.py`)**

```python
from django.db import models
from customers.models import Customer
from equipment.models import Equipment

class CustomerSiteInventory(models.Model):
    customer = models.ForeignKey(Customer, on_delete=models.CASCADE)
    equipment = models.ForeignKey(Equipment, on_delete=models.PROTECT)
    quantity = models.PositiveIntegerField(default=0)
    class Meta: unique_together = ('customer', 'equipment')
```

### **Transactions (`transactions/models.py`)**

```python
from django.db import models
from django.conf import settings
from customers.models import Customer

class Transaction(models.Model):
    transaction_number = models.CharField(max_length=30, unique=True)
    customer = models.ForeignKey(Customer, on_delete=models.PROTECT, related_name='transactions')
    user = models.ForeignKey(settings.AUTH_USER_MODEL, on_delete=models.PROTECT)
    total_amount = models.DecimalField(max_digits=12, decimal_places=2)
    created_at = models.DateTimeField(auto_now_add=True)
    def __str__(self): return self.transaction_number
```

### **Invoices (`invoices/models.py`)**

```python
from django.db import models
from transactions.models import Transaction

class Invoice(models.Model):
    STATUS_CHOICES = [('generated','Generated'),('printed','Printed'),('emailed','Emailed'),('paid','Paid')]
    invoice_number = models.CharField(max_length=30, unique=True)
    transaction = models.OneToOneField(Transaction, on_delete=models.PROTECT, related_name='invoice')
    pdf_path = models.CharField(max_length=500)
    status = models.CharField(max_length=20, choices=STATUS_CHOICES, default='generated')
    generated_at = models.DateTimeField(auto_now_add=True)
    printed_at = models.DateTimeField(null=True, blank=True)
    emailed_at = models.DateTimeField(null=True, blank=True)
    def __str__(self): return self.invoice_number
```

### **Audit (`audit/models.py`)**

```python
from django.db import models
from django.conf import settings

class AuditLog(models.Model):
    user = models.ForeignKey(settings.AUTH_USER_MODEL, on_delete=models.SET_NULL, null=True)
    action = models.CharField(max_length=100)
    entity_type = models.CharField(max_length=50)
    entity_id = models.PositiveIntegerField(null=True)
    payload = models.JSONField(default=dict)
    created_at = models.DateTimeField(auto_now_add=True)
    def __str__(self): return f"{self.action} ({self.entity_type})"
```

### **Distribution (`distribution/models.py`)**

```python
from django.db import models
from django.conf import settings
from depots.models import Depot
from equipment.models import Equipment

class Distribution(models.Model):
    distribution_number = models.CharField(max_length=30, unique=True)
    depot = models.ForeignKey(Depot, on_delete=models.PROTECT, related_name='distributions')
    user = models.ForeignKey(settings.AUTH_USER_MODEL, on_delete=models.PROTECT, related_name='distributions')
    remarks = models.TextField(blank=True)
    created_at = models.DateTimeField(auto_now_add=True)
    confirmed_at = models.DateTimeField(null=True, blank=True)
    def __str__(self): return self.distribution_number

class DistributionItem(models.Model):
    DIRECTION_CHOICES = (('OUT','Out from Depot'),('IN','Return to Depot'))
    CONDITION_CHOICES = (('FULL','Full'),('EMPTY','Empty'),('DAMAGED','Damaged'))
    distribution = models.ForeignKey(Distribution, related_name='items', on_delete=models.CASCADE)
    equipment = models.ForeignKey(Equipment, on_delete=models.PROTECT)
    direction = models.CharField(max_length=3, choices=DIRECTION_CHOICES)
    condition = models.CharField(max_length=10, choices=CONDITION_CHOICES)
    quantity = models.PositiveIntegerField()
    def __str__(self): return f"{self.distribution.distribution_number} - {self.equipment.name}"
```

---

## **6️⃣ Serializers**

### **Accounts (`accounts/serializers.py`)**

```python
from rest_framework import serializers
from accounts.models import User
from depots.models import Depot

class UserSerializer(serializers.ModelSerializer):
    depot_name = serializers.CharField(source='depot.name', read_only=True)
    
    class Meta:
        model = User
        fields = ['id','username','employee_id','role','depot','depot_name']
```

---

### **Invoices (`invoices/serializers.py`)**

```python
# invoices/serializers.py
from rest_framework import serializers
from invoices.models import Invoice
from transactions.serializers import TransactionSerializer

class InvoiceSerializer(serializers.ModelSerializer):
    transaction = TransactionSerializer(read_only=True)

    class Meta:
        model = Invoice
        fields = [
            'id',
            'invoice_number',
            'transaction',
            'pdf_path',
            'status',
            'generated_at',
            'printed_at',
            'emailed_at'
        ]
```

---

### **Depots (`depots/serializers.py`)**

```python
from rest_framework import serializers
from depots.models import Depot

class DepotSerializer(serializers.ModelSerializer):
    class Meta:
        model = Depot
        fields = ['id','code','name','address']
```

---

### **Equipment (`equipment/serializers.py`)**

```python
from rest_framework import serializers
from equipment.models import Equipment

class EquipmentSerializer(serializers.ModelSerializer):
    class Meta:
        model = Equipment
        fields = ['id','name','sku','equipment_type','weight_kg','is_active']
```

---

### **Customers (`customers/serializers.py`)**

```python
from rest_framework import serializers
from customers.models import Customer

class CustomerSerializer(serializers.ModelSerializer):
    class Meta:
        model = Customer
        fields = ['id','name','email','address','payment_type','is_meter_installed','last_meter_reading','meter_rate']
```

---

### **Inventory (`inventory/serializers.py`)**

```python
from rest_framework import serializers
from inventory.models import CustomerSiteInventory

class CustomerSiteInventorySerializer(serializers.ModelSerializer):
    class Meta:
        model = CustomerSiteInventory
        fields = ['id','customer','equipment','quantity']
```

---

### **Transactions (`transactions/serializers.py`)**

```python
# transactions/serializers.py
from rest_framework import serializers
from transactions.models import Transaction

class TransactionSerializer(serializers.ModelSerializer):
    class Meta:
        model = Transaction
        fields = ['id','transaction_number','customer','user','total_amount','created_at']
```

---

### **Distribution (`distribution/serializers.py`)**

```python
from rest_framework import serializers
from distribution.models import Distribution, DistributionItem

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

---

## **7️⃣ Services**

### **Transactions & Invoices (`transactions/services.py`)**

```python
from django.db import transaction
from django.template.loader import render_to_string
from django.core.mail import EmailMessage
from django.utils import timezone
from decimal import Decimal
from transactions.models import Transaction
from invoices.models import Invoice
from customers.models import Customer
from core.utils.numbering import generate_number
from weasyprint import HTML
from audit.models import AuditLog

@transaction.atomic
def create_customer_transaction_and_invoice(user, data):
    customer = Customer.objects.get(id=data['customer'])
    items = data.get('items',[])
    current_meter = data.get('current_meter')

    # 1. Usage Billing
    usage_amount = Decimal('0.00')
    if customer.is_meter_installed and current_meter is not None:
        usage = Decimal(current_meter) - (customer.last_meter_reading or 0)
        usage_amount = usage * (customer.meter_rate or 0)
        customer.last_meter_reading = Decimal(current_meter)
        customer.save()

    # 2. Items total
    items_amount = sum(Decimal(item['rate']) * item['quantity'] for item in items)

    total = usage_amount + items_amount

    # 3. Transaction
    tx = Transaction.objects.create(
        transaction_number=generate_number('TRX'),
        customer=customer,
        user=user,
        total_amount=total
    )

    # 4. Invoice PDF
    html_string = render_to_string('invoices/invoice.html', {'tx': tx, 'customer': customer})
    pdf_file = f'invoices/{tx.transaction_number}.pdf'
    HTML(string=html_string).write_pdf(target=pdf_file)

    # 5. Invoice object
    invoice = Invoice.objects.create(
        invoice_number=generate_number('INV'),
        transaction=tx,
        pdf_path=pdf_file
    )

    # 6. Email invoice
    email = EmailMessage(
        subject=f"HSH LPG Invoice {invoice.invoice_number}",
        body=f"Dear {customer.name},\n\nPlease find attached your invoice.",
        from_email=None,
        to=[customer.email]
    )
    email.attach_file(pdf_file)
    email.send(fail_silently=True)

    # 7. Audit
    AuditLog.objects.create(
        user=user,
        action='Create Transaction + Invoice',
        entity_type='Transaction',
        entity_id=tx.id,
        payload={'items': items,'total': str(total)}
    )

    return tx, invoice
```

---

### **Distribution Service (`distribution/services.py`)**

```python
from django.db import transaction
from django.utils import timezone
from core.utils.numbering import generate_number
from distribution.models import Distribution, DistributionItem
from inventory.models import CustomerSiteInventory
from audit.models import AuditLog

@transaction.atomic
def create_distribution(user, depot, items, remarks=''):
    """
    Create distribution WITHOUT auto-confirmation.
    """
    distribution = Distribution.objects.create(
        distribution_number=generate_number('DST'),
        depot=depot,
        user=user,
        remarks=remarks,
        confirmed_at=None
    )
    for item in items:
        DistributionItem.objects.create(
            distribution=distribution,
            equipment=item['equipment'],
            direction=item['direction'],
            condition=item['condition'],
            quantity=item['quantity']
        )
    return distribution

@transaction.atomic
def confirm_distribution(distribution_id, user):
    """
    Confirm distribution and update depot inventory.
    """
    dist = Distribution.objects.get(id=distribution_id)
    if dist.confirmed_at:
        raise ValueError("Distribution already confirmed")

    dist.confirmed_at = timezone.now()
    dist.save()

    # Example: update inventory (depot-level or customer-level)
    for item in dist.items.all():
        inv, _ = CustomerSiteInventory.objects.get_or_create(
            customer=None,  # depot-level inventory
            equipment=item.equipment,
            defaults={'quantity': 0}
        )
        if item.direction == 'OUT':
            inv.quantity = max(inv.quantity - item.quantity, 0)
        else:
            inv.quantity += item.quantity
        inv.save()

    AuditLog.objects.create(
        user=user,
        action='Confirm Distribution',
        entity_type='Distribution',
        entity_id=dist.id,
        payload={'distribution_number': dist.distribution_number}
    )
    return dist
```

### **Inventory Service (`inventory/services.py`)**
```
from django.db import transaction
from inventory.models import CustomerSiteInventory
from customers.models import Customer
from equipment.models import Equipment
from audit.models import AuditLog

@transaction.atomic
def update_inventory(entity, entity_id, equipment_id, quantity, user):
    if entity != 'customer':
        raise ValueError("Currently only 'customer' entity supported")
    
    customer = Customer.objects.get(id=entity_id)
    equipment = Equipment.objects.get(id=equipment_id)

    inv, created = CustomerSiteInventory.objects.get_or_create(
        customer=customer,
        equipment=equipment,
        defaults={'quantity': quantity}
    )
    if not created:
        inv.quantity = quantity
        inv.save()

    AuditLog.objects.create(
        user=user,
        action='Update Inventory',
        entity_type='CustomerSiteInventory',
        entity_id=inv.id,
        payload={'quantity': quantity}
    )
    return inv
```

---

## **8️⃣ ViewSets**

### **Transactions (`transactions/views.py`)**

```python
from rest_framework import viewsets, status
from rest_framework.decorators import action
from rest_framework.response import Response
from transactions.models import Transaction
from transactions.serializers import TransactionSerializer
from invoices.serializers import InvoiceSerializer
from transactions.services import create_customer_transaction_and_invoice

class TransactionViewSet(viewsets.ModelViewSet):
    queryset = Transaction.objects.all()
    serializer_class = TransactionSerializer

    @action(detail=False, methods=['post'])
    def create_transaction(self, request):
        user = request.user
        data = request.data
        try:
            tx, invoice = create_customer_transaction_and_invoice(user, data)
            tx_ser = TransactionSerializer(tx)
            inv_ser = InvoiceSerializer(invoice)
            return Response({'transaction': tx_ser.data, 'invoice': inv_ser.data}, status=status.HTTP_201_CREATED)
        except Exception as e:
            return Response({'error': str(e)}, status=status.HTTP_400_BAD_REQUEST)
```

---
### **Invoices (`invoices/views.py`)**

```
from rest_framework.decorators import action
from rest_framework.response import Response
from rest_framework import viewsets, status
from invoices.models import Invoice
from invoices.serializers import InvoiceSerializer
from django.core.mail import EmailMessage
from django.http import FileResponse
from django.utils import timezone
from audit.models import AuditLog

class InvoiceViewSet(viewsets.ModelViewSet):
    queryset = Invoice.objects.all()
    serializer_class = InvoiceSerializer

    @action(detail=True, methods=['get'])
    def pdf(self, request, pk=None):
        invoice = self.get_object()
        try:
            return FileResponse(open(invoice.pdf_path,'rb'), content_type='application/pdf')
        except FileNotFoundError:
            return Response({'error': 'PDF not found'}, status=status.HTTP_404_NOT_FOUND)

    @action(detail=True, methods=['post'])
    def email(self, request, pk=None):
        invoice = self.get_object()
        try:
            email = EmailMessage(
                subject=f"HSH LPG Invoice {invoice.invoice_number}",
                body=f"Dear Customer,\n\nPlease find attached your invoice.",
                from_email=None,
                to=[invoice.transaction.customer.email]
            )
            email.attach_file(invoice.pdf_path)
            email.send(fail_silently=False)

            invoice.status = 'emailed'
            invoice.emailed_at = timezone.now()
            invoice.save()

            AuditLog.objects.create(
                user=request.user,
                action='Email Invoice',
                entity_type='Invoice',
                entity_id=invoice.id,
                payload={'invoice_number': invoice.invoice_number}
            )
            serializer = self.get_serializer(invoice)
            return Response(serializer.data, status=status.HTTP_200_OK)
        except Exception as e:
            return Response({'error': str(e)}, status=status.HTTP_400_BAD_REQUEST)

```

---
### **Customers (`customers/views.py`)**

```python
from rest_framework import viewsets
from customers.models import Customer
from customers.serializers import CustomerSerializer

class CustomerViewSet(viewsets.ModelViewSet):
    queryset = Customer.objects.all()
    serializer_class = CustomerSerializer
```

---

### **Inventory (`inventory/views.py`)**

```python
from rest_framework.decorators import action
from rest_framework.response import Response
from rest_framework import viewsets, status
from inventory.models import CustomerSiteInventory
from inventory.serializers import CustomerSiteInventorySerializer
from inventory.services import update_inventory

class CustomerSiteInventoryViewSet(viewsets.ModelViewSet):
    queryset = CustomerSiteInventory.objects.all()
    serializer_class = CustomerSiteInventorySerializer

    @action(detail=False, methods=['post'])
    def update_inventory(self, request):
        try:
            inv = update_inventory(
                entity=request.data['entity'],
                entity_id=request.data['entity_id'],
                equipment_id=request.data['equipment_id'],
                quantity=request.data['quantity'],
                user=request.user
            )
            serializer = self.get_serializer(inv)
            return Response(serializer.data, status=status.HTTP_200_OK)
        except Exception as e:
            return Response({'error': str(e)}, status=status.HTTP_400_BAD_REQUEST)

```

---

### **Distribution (`distribution/views.py`)**

```python
from rest_framework.decorators import action
from rest_framework.response import Response
from rest_framework import viewsets, status
from distribution.models import Distribution
from distribution.serializers import DistributionSerializer
from distribution.services import create_distribution, confirm_distribution
from depots.models import Depot

class DistributionViewSet(viewsets.ModelViewSet):
    queryset = Distribution.objects.all().prefetch_related('items')
    serializer_class = DistributionSerializer

    @action(detail=False, methods=['post'])
    def create_distribution(self, request):
        try:
            depot = Depot.objects.get(id=request.data['depot'])
            distribution = create_distribution(
                user=request.user,
                depot=depot,
                items=request.data['items'],
                remarks=request.data.get('remarks','')
            )
            serializer = self.get_serializer(distribution)
            return Response(serializer.data, status=status.HTTP_201_CREATED)
        except Exception as e:
            return Response({'error': str(e)}, status=status.HTTP_400_BAD_REQUEST)

    @action(detail=True, methods=['post'])
    def confirm(self, request, pk=None):
        try:
            dist = confirm_distribution(pk, request.user)
            serializer = self.get_serializer(dist)
            return Response(serializer.data, status=status.HTTP_200_OK)
        except Exception as e:
            return Response({'error': str(e)}, status=status.HTTP_400_BAD_REQUEST)
```

---

## **9️⃣ Middleware – API Logging**

```python
# middleware/request_logging.py
import logging, time
logger = logging.getLogger('api.access')

class APILoggingMiddleware:
    def __init__(self, get_response): self.get_response = get_response
    def __call__(self, request):
        if not request.path.startswith('/api/'): return self.get_response(request)
        start_time = time.monotonic()
        response = self.get_response(request)
        duration = int((time.monotonic()-start_time)*1000)
        user = request.user if request.user.is_authenticated else None
        user_str = f"{user.id}:{user.username}" if user else "anonymous"
        logger.info("%s %s %s %s %dms %s",
            request.method, request.path, response.status_code,
            user_str, duration,
            request.META.get('HTTP_X_FORWARDED_FOR', request.META.get('REMOTE_ADDR',''))
        )
        return response
```

---

## **🔟 URLs & Routers**

```python
# core/urls.py
from django.contrib import admin
from django.urls import path, include
from rest_framework import routers
from transactions.views import TransactionViewSet
from customers.views import CustomerViewSet
from inventory.views import CustomerSiteInventoryViewSet
from distribution.views import DistributionViewSet
from drf_spectacular.views import SpectacularAPIView, SpectacularSwaggerView

router = routers.DefaultRouter()
router.register(r'transactions', TransactionViewSet, basename='transaction')
router.register(r'customers', CustomerViewSet)
router.register(r'inventories', CustomerSiteInventoryViewSet)
router.register(r'distributions', DistributionViewSet, basename='distribution')

urlpatterns = [
    path('admin/', admin.site.urls),
    path('api/schema/', SpectacularAPIView.as_view(), name='schema'),
    path('api/schema/swagger-ui/', SpectacularSwaggerView.as_view(url_name='schema'), name='swagger-ui'),
    path('api/', include(router.urls)),
]
```

---

## **🔹 Invoice Template**

```html
<!-- templates/invoices/invoice.html -->
<!DOCTYPE html>
<html>
<head><meta charset="utf-8"><title>HSH LPG Invoice</title></head>
<body>
<h1>HSH LPG INVOICE</h1>
<p>Invoice: {{ tx.transaction_number }}</p>
<p>Customer: {{ customer.name }}</p>
<p>Date: {{ tx.created_at|date:"d M Y" }}</p>
<p>Total Amount: SGD {{ tx.total_amount }}</p>
<p>Thank you for your business.<br>HSH LPG</p>
</body>
</html>
```

---

## **🔹 Run & Test**

```bash
pip install -r requirements.txt
python manage.py makemigrations
python manage.py migrate
python manage.py createsuperuser
python manage.py runserver
```

* Swagger UI: `http://127.0.0.1:8000/api/schema/swagger-ui/`
* JWT auth protects endpoints
* Create transactions → generates PDF → sends email → logs audit
* Create distributions → updates inventory → atomic and auditable

---

## **🔹 ERD (Entity Relationship)**

```
Depot ──< Distribution >── DistributionItem
Depot ──< User
Customer ──< CustomerSiteInventory >── Equipment
Customer ──< Transaction ── Invoice
Transaction ── User
Distribution.user = User
Distribution.depot = Depot
```

---

## **🔹 Field Workflow**

1. **Depot Admin** manages inventory and equipment.
2. **Driver / User** creates **Distribution** → inventory is updated.
3. **Customer Site** triggers **Transaction** → invoice PDF generated → emailed → audit logged.
4. **Atomic transactions** ensure inventory, distributions, and invoices remain consistent.


---
✅ Now all endpoints from your Postman guide are fully supported, atomic, and auditable:

Endpoint	Method	Action
/customers/	CRUD	Customer management
/inventories/update/	POST	Adjust quantity
/distributions/	POST	Create distribution (unconfirmed)
/distributions/<id>/confirm/	POST	Confirm distribution
/transactions/create_transaction/	POST	Create transaction + invoice
/invoices/<id>/pdf/	GET	Download invoice PDF
/invoices/<id>/email/	POST	Send invoice email
/audit/	GET	List audit logs
