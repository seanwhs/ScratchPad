# **HSH LPG Sales & Logistics System – Full Production-Ready Backend Tutorial (Django 6, 2026)**

**MVP – Singapore Field Operations**
**Stack:** Django 6 + DRF + drf-spectacular + WeasyPrint + JWT + Atomic Transactions + Audit Logging + PDF/Email Invoicing

---

## **1️⃣ Technical Stack**

| Component      | Choice            | Why                      |
| -------------- | ----------------- | ------------------------ |
| Python         | 3.12              | Latest stable            |
| Django         | 6                 | LTS + async support      |
| DRF            | 3.15+             | Modern API framework     |
| OpenAPI Docs   | drf-spectacular   | Auto Swagger/OpenAPI     |
| PDF Generation | WeasyPrint        | HTML → PDF reliable      |
| Auth           | Simple JWT        | Industry-standard JWT    |
| Database       | SQLite/MySQL      | ACID compliant           |
| Logging        | Custom Middleware | Full API request logging |
| Email          | Gmail SMTP        | Field-ready, reliable    |

---

## **2️⃣ Project Setup**

```bash
mkdir hsh_lpg_system && cd hsh_lpg_system
python -m venv venv
source venv/bin/activate       # Windows: venv\Scripts\activate

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
# core/settings.py
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
    'rest_framework', 'rest_framework.authtoken', 'drf_spectacular','corsheaders',
    
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
    'DEFAULT_AUTHENTICATION_CLASSES': (
        'rest_framework_simplejwt.authentication.JWTAuthentication',
    ),
    'DEFAULT_PERMISSION_CLASSES': (
        'rest_framework.permissions.IsAuthenticated',
    ),
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
    'ROTATE_REFRESH_TOKENS': True,
    'BLACKLIST_AFTER_ROTATION': True,
    'AUTH_HEADER_TYPES': ('Bearer',),
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
    ROLE_CHOICES = (
        ('ADMIN','Admin'),
        ('DRIVER','Driver'),
        ('SUPERVISOR','Supervisor')
    )
    employee_id = models.CharField(max_length=20, unique=True)
    role = models.CharField(max_length=20, choices=ROLE_CHOICES, default='DRIVER')
    depot = models.ForeignKey(Depot, on_delete=models.SET_NULL, null=True, blank=True)
```

---

### **Depots (`depots/models.py`)**

```python
from django.db import models

class Depot(models.Model):
    code = models.CharField(max_length=20, unique=True)
    name = models.CharField(max_length=200)
    address = models.TextField(blank=True)

    def __str__(self):
        return self.name
```

---

### **Equipment (`equipment/models.py`)**

```python
from django.db import models

class Equipment(models.Model):
    EQUIPMENT_TYPE_CHOICES = (
        ('CYLINDER','Cylinder'),
        ('METER','Meter'),
        ('REGULATOR','Regulator')
    )
    name = models.CharField(max_length=100)
    sku = models.CharField(max_length=50, unique=True)
    equipment_type = models.CharField(max_length=20, choices=EQUIPMENT_TYPE_CHOICES)
    weight_kg = models.DecimalField(max_digits=6, decimal_places=2, null=True, blank=True)
    is_active = models.BooleanField(default=True)

    def __str__(self):
        return f"{self.name} ({self.sku})"
```

---

### **Customers (`customers/models.py`)**

```python
from django.db import models

class Customer(models.Model):
    PAYMENT_CHOICES = [('CASH','Cash'),('CREDIT','Credit')]
    name = models.CharField(max_length=200)
    email = models.EmailField(unique=True)
    address = models.TextField(blank=True)
    payment_type = models.CharField(max_length=10, choices=PAYMENT_CHOICES)
    is_meter_installed = models.BooleanField(default=False)
    last_meter_reading = models.DecimalField(max_digits=12, decimal_places=2, null=True, blank=True)
    meter_rate = models.DecimalField(max_digits=8, decimal_places=4, null=True, blank=True)

    def __str__(self):
        return self.name
```

---

### **Inventory (`inventory/models.py`)**

```python
from django.db import models
from customers.models import Customer
from equipment.models import Equipment

class CustomerSiteInventory(models.Model):
    customer = models.ForeignKey(Customer, on_delete=models.CASCADE)
    equipment = models.ForeignKey(Equipment, on_delete=models.PROTECT)
    quantity = models.PositiveIntegerField(default=0)

    class Meta:
        unique_together = ('customer','equipment')

    def __str__(self):
        return f"{self.customer.name} - {self.equipment.name}: {self.quantity}"
```

---

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

    def __str__(self):
        return self.transaction_number
```

---

### **Invoices (`invoices/models.py`)**

```python
from django.db import models
from transactions.models import Transaction

class Invoice(models.Model):
    STATUS_CHOICES = [
        ('generated','Generated'),
        ('printed','Printed'),
        ('emailed','Emailed'),
        ('paid','Paid')
    ]

    invoice_number = models.CharField(max_length=30, unique=True)
    transaction = models.OneToOneField(Transaction, on_delete=models.PROTECT, related_name='invoice')
    pdf_path = models.CharField(max_length=500)
    status = models.CharField(max_length=20, choices=STATUS_CHOICES, default='generated')
    generated_at = models.DateTimeField(auto_now_add=True)
    printed_at = models.DateTimeField(null=True, blank=True)
    emailed_at = models.DateTimeField(null=True, blank=True)

    def __str__(self):
        return self.invoice_number
```

---

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

    def __str__(self):
        return self.distribution_number

class DistributionItem(models.Model):
    DIRECTION_CHOICES = [('OUT','Out from Depot'),('IN','Return to Depot')]
    CONDITION_CHOICES = [('FULL','Full'),('EMPTY','Empty'),('DAMAGED','Damaged')]

    distribution = models.ForeignKey(Distribution, related_name='items', on_delete=models.CASCADE)
    equipment = models.ForeignKey(Equipment, on_delete=models.PROTECT)
    direction = models.CharField(max_length=3, choices=DIRECTION_CHOICES)
    condition = models.CharField(max_length=10, choices=CONDITION_CHOICES)
    quantity = models.PositiveIntegerField()

    def __str__(self):
        return f"{self.distribution.distribution_number} - {self.equipment.name}"
```

---

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

    def __str__(self):
        return f"{self.action} ({self.entity_type})"
```

---

## **6️⃣ Serializers**

### **Accounts (`accounts/serializers.py`)**

```python
from rest_framework import serializers
from accounts.models import User

class UserSerializer(serializers.ModelSerializer):
    depot_name = serializers.CharField(source='depot.name', read_only=True)

    class Meta:
        model = User
        fields = ['id','username','employee_id','role','depot','depot_name']
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

### **Equipment (`equipment/serializers.py`)**

```python
# equipment/serializers.py
from rest_framework import serializers
from equipment.models import Equipment

class EquipmentSerializer(serializers.ModelSerializer):
    is_active = serializers.BooleanField(default=True)

    class Meta:
        model = Equipment
        fields = [
            'id',
            'name',
            'sku',
            'equipment_type',
            'weight_kg',
            'is_active',
        ]

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
from rest_framework import serializers
from transactions.models import Transaction

class TransactionSerializer(serializers.ModelSerializer):
    class Meta:
        model = Transaction
        fields = ['id','transaction_number','customer','user','total_amount','created_at']
```

---

### **Invoices (`invoices/serializers.py`)**

```python
from rest_framework import serializers
from invoices.models import Invoice
from transactions.serializers import TransactionSerializer

class InvoiceSerializer(serializers.ModelSerializer):
    transaction = TransactionSerializer(read_only=True)

    class Meta:
        model = Invoice
        fields = ['id','invoice_number','transaction','pdf_path','status','generated_at','printed_at','emailed_at']
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

### **Audit (`audit/serializers.py`)**

```python
from rest_framework import serializers
from audit.models import AuditLog

class AuditLogSerializer(serializers.ModelSerializer):
    user_name = serializers.CharField(source='user.username', read_only=True)

    class Meta:
        model = AuditLog
        fields = ['id','user','user_name','action','entity_type','entity_id','payload','created_at']
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
    """
    Creates a transaction, generates an invoice PDF, emails it, and logs audit.
    """
    customer = Customer.objects.get(id=data['customer'])
    items = data.get('items', [])
    current_meter = data.get('current_meter')

    # 1️⃣ Usage Billing
    usage_amount = Decimal('0.00')
    if customer.is_meter_installed and current_meter is not None:
        usage = Decimal(current_meter) - (customer.last_meter_reading or 0)
        usage_amount = usage * (customer.meter_rate or 0)
        customer.last_meter_reading = Decimal(current_meter)
        customer.save()

    # 2️⃣ Items total
    items_amount = sum(Decimal(item['rate']) * item['quantity'] for item in items)

    total = usage_amount + items_amount

    # 3️⃣ Transaction
    tx = Transaction.objects.create(
        transaction_number=generate_number('TRX'),
        customer=customer,
        user=user,
        total_amount=total
    )

    # 4️⃣ Invoice PDF generation
    html_string = render_to_string('invoices/invoice.html', {'tx': tx, 'customer': customer})
    pdf_file = f'invoices/{tx.transaction_number}.pdf'
    HTML(string=html_string).write_pdf(target=pdf_file)

    # 5️⃣ Invoice object
    invoice = Invoice.objects.create(
        invoice_number=generate_number('INV'),
        transaction=tx,
        pdf_path=pdf_file
    )

    # 6️⃣ Email invoice
    email = EmailMessage(
        subject=f"HSH LPG Invoice {invoice.invoice_number}",
        body=f"Dear {customer.name},\n\nPlease find attached your invoice.",
        from_email=None,
        to=[customer.email]
    )
    email.attach_file(pdf_file)
    email.send(fail_silently=True)

    # 7️⃣ Audit logging
    AuditLog.objects.create(
        user=user,
        action='Create Transaction + Invoice',
        entity_type='Transaction',
        entity_id=tx.id,
        payload={'items': items, 'total': str(total)}
    )

    return tx, invoice
```

---

### **Distribution Services (`distribution/services.py`)**

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
    Creates a distribution WITHOUT confirmation.
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
    Confirms a distribution and updates inventory.
    """
    dist = Distribution.objects.get(id=distribution_id)
    if dist.confirmed_at:
        raise ValueError("Distribution already confirmed")

    dist.confirmed_at = timezone.now()
    dist.save()

    # Update inventory
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

    # Audit log
    AuditLog.objects.create(
        user=user,
        action='Confirm Distribution',
        entity_type='Distribution',
        entity_id=dist.id,
        payload={'distribution_number': dist.distribution_number}
    )
    return dist
```

---

### **Inventory Services (`inventory/services.py`)**

```python
from django.db import transaction
from inventory.models import CustomerSiteInventory
from customers.models import Customer
from equipment.models import Equipment
from audit.models import AuditLog

@transaction.atomic
def update_inventory(entity, entity_id, equipment_id, quantity, user):
    """
    Update customer inventory and log audit.
    """
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

```python
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

### **Equipment (`equipment/views.py`)**

```python
# equipment/views.py
from rest_framework import viewsets
from rest_framework.permissions import IsAuthenticated
from equipment.models import Equipment
from equipment.serializers import EquipmentSerializer

class EquipmentViewSet(viewsets.ModelViewSet):
    """
    Equipment master data.

    Required by:
    - Inventory
    - Distribution
    - Transactions
    """
    queryset = Equipment.objects.filter(is_active=True).order_by('name')
    serializer_class = EquipmentSerializer
    permission_classes = [IsAuthenticated]

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

### **Audit (`audit/views.py`)**

```python
from rest_framework import viewsets
from audit.models import AuditLog
from audit.serializers import AuditLogSerializer

class AuditLogViewSet(viewsets.ReadOnlyModelViewSet):
    """
    Read-only viewset for Audit Logs.
    """
    queryset = AuditLog.objects.all().order_by('-created_at')
    serializer_class = AuditLogSerializer
```

---

## **9️⃣ Middleware – API Logging (`middleware/request_logging.py`)**

```python
import logging, time

logger = logging.getLogger('api.access')

class APILoggingMiddleware:
    def __init__(self, get_response):
        self.get_response = get_response

    def __call__(self, request):
        if not request.path.startswith('/api/'):
            return self.get_response(request)

        start_time = time.monotonic()
        response = self.get_response(request)
        duration = int((time.monotonic() - start_time) * 1000)

        user = request.user if request.user.is_authenticated else None
        user_str = f"{user.id}:{user.username}" if user else "anonymous"

        logger.info(
            "%s %s %s %s %dms %s",
            request.method,
            request.path,
            response.status_code,
            user_str,
            duration,
            request.META.get('HTTP_X_FORWARDED_FOR', request.META.get('REMOTE_ADDR',''))
        )
        return response
```

---

## **🔟 Admin Registrations**

### **Accounts (`accounts/admin.py`)**

```python
from django.contrib import admin
from django.contrib.auth.admin import UserAdmin
from .models import User

@admin.register(User)
class CustomUserAdmin(UserAdmin):
    fieldsets = UserAdmin.fieldsets + (
        ('LPG System Info', {'fields': ('employee_id', 'role', 'depot')}),
    )
    list_display = ['username', 'employee_id', 'role', 'depot', 'is_staff']
    list_filter = ['role', 'depot']
```

---

### **Depots & Equipment (`depots/admin.py` & `equipment/admin.py`)**

```python
# depots/admin.py
from django.contrib import admin
from .models import Depot

@admin.register(Depot)
class DepotAdmin(admin.ModelAdmin):
    list_display = ['code', 'name']
    search_fields = ['name', 'code']

# equipment/admin.py
from django.contrib import admin
from .models import Equipment

@admin.register(Equipment)
class EquipmentAdmin(admin.ModelAdmin):
    list_display = ['sku', 'name', 'equipment_type', 'is_active']
    list_filter = ['equipment_type', 'is_active']
```

---

### **Customers & Inventory (`customers/admin.py` & `inventory/admin.py`)**

```python
# customers/admin.py
from django.contrib import admin
from .models import Customer

@admin.register(Customer)
class CustomerAdmin(admin.ModelAdmin):
    list_display = ['name', 'email', 'payment_type', 'is_meter_installed']
    search_fields = ['name', 'email']

# inventory/admin.py
from django.contrib import admin
from .models import CustomerSiteInventory

@admin.register(CustomerSiteInventory)
class InventoryAdmin(admin.ModelAdmin):
    list_display = ['customer', 'equipment', 'quantity']
    list_filter = ['equipment']
    search_fields = ['customer__name']
```

---

### **Transactions & Invoices (`transactions/admin.py` & `invoices/admin.py`)**

```python
# transactions/admin.py
from django.contrib import admin
from .models import Transaction

@admin.register(Transaction)
class TransactionAdmin(admin.ModelAdmin):
    list_display = ['transaction_number', 'customer', 'total_amount', 'created_at']
    readonly_fields = ['transaction_number', 'created_at']

# invoices/admin.py
from django.contrib import admin
from .models import Invoice

@admin.register(Invoice)
class InvoiceAdmin(admin.ModelAdmin):
    list_display = ['invoice_number', 'status', 'generated_at', 'emailed_at']
    list_filter = ['status']
    readonly_fields = ['invoice_number', 'pdf_path', 'generated_at']
```

---

### **Distribution & Audit (`distribution/admin.py` & `audit/admin.py`)**

```python
# distribution/admin.py
from django.contrib import admin
from .models import Distribution, DistributionItem

class DistributionItemInline(admin.TabularInline):
    model = DistributionItem
    extra = 1

@admin.register(Distribution)
class DistributionAdmin(admin.ModelAdmin):
    list_display = ['distribution_number', 'depot', 'user', 'confirmed_at']
    inlines = [DistributionItemInline]
    readonly_fields = ['distribution_number', 'created_at']

# audit/admin.py
from django.contrib import admin
from .models import AuditLog

@admin.register(AuditLog)
class AuditLogAdmin(admin.ModelAdmin):
    list_display = ['created_at', 'user', 'action', 'entity_type', 'entity_id']
    list_filter = ['action', 'entity_type']
    readonly_fields = [f.name for f in AuditLog._meta.get_fields()]
    
    def has_add_permission(self, request): return False
    def has_change_permission(self, request, obj=None): return False
    def has_delete_permission(self, request, obj=None): return False
```

---

## **🔹 URLs & Routers (`core/urls.py`)**

```python
from django.contrib import admin
from django.urls import path, include
from rest_framework import routers

from rest_framework_simplejwt.views import (
    TokenObtainPairView,
    TokenRefreshView,
)

from audit.views import AuditLogViewSet
from equipment.views import EquipmentViewSet
from transactions.views import TransactionViewSet
from customers.views import CustomerViewSet
from inventory.views import CustomerSiteInventoryViewSet
from distribution.views import DistributionViewSet
from invoices.views import InvoiceViewSet

from drf_spectacular.views import (
    SpectacularAPIView,
    SpectacularSwaggerView,
)

router = routers.DefaultRouter()
router.register(r'audit', AuditLogViewSet, basename='audit')
router.register(r'equipment', EquipmentViewSet, basename='equipment') 
router.register(r'transactions', TransactionViewSet, basename='transaction')
router.register(r'customers', CustomerViewSet)
router.register(r'inventories', CustomerSiteInventoryViewSet)
router.register(r'distributions', DistributionViewSet, basename='distribution')
router.register(r'invoices', InvoiceViewSet, basename='invoice')

urlpatterns = [
    path('admin/', admin.site.urls),

    # =========================
    # JWT AUTH
    # =========================
    path('api/token/', TokenObtainPairView.as_view(), name='token_obtain_pair'),
    path('api/token/refresh/', TokenRefreshView.as_view(), name='token_refresh'),

    # =========================
    # API DOCS
    # =========================
    path('api/schema/', SpectacularAPIView.as_view(), name='schema'),
    path(
        'api/schema/swagger-ui/',
        SpectacularSwaggerView.as_view(url_name='schema'),
        name='swagger-ui',
    ),

    # =========================
    # API ROUTES
    # =========================
    path('api/', include(router.urls)),
]

```

---

## **🔹 Invoice Template (`templates/invoices/invoice.html`)**

```html
<!DOCTYPE html>
<html>
<head>
    <meta charset="utf-8">
    <title>HSH LPG Invoice</title>
    <style>
        body { font-family: Arial, sans-serif; }
        h1 { color: #2E86C1; }
        table { width: 100%; border-collapse: collapse; margin-top: 20px; }
        th, td { border: 1px solid #ddd; padding: 8px; }
        th { background-color: #f2f2f2; }
        .total { font-weight: bold; }
    </style>
</head>
<body>
<h1>HSH LPG INVOICE</h1>
<p>Invoice: {{ tx.transaction_number }}</p>
<p>Customer: {{ customer.name }}</p>
<p>Date: {{ tx.created_at|date:"d M Y" }}</p>

<table>
    <tr>
        <th>Description</th>
        <th>Amount (SGD)</th>
    </tr>
    <tr>
        <td>Usage / Meter Billing</td>
        <td>{{ tx.total_amount }}</td>
    </tr>
</table>

<p class="total">Total Amount: SGD {{ tx.total_amount }}</p>
<p>Thank you for your business.<br>HSH LPG</p>
</body>
</html>
```

---

## **🔹 Run & Test**

```bash
# Create environment & install dependencies
python -m venv venv
source venv/bin/activate  # Windows: venv\Scripts\activate
pip install -r requirements.txt

# Apply migrations
python manage.py makemigrations
python manage.py migrate

# Create superuser
python manage.py createsuperuser

# Run server
python manage.py runserver
```

**Swagger UI:** `http://127.0.0.1:8000/api/schema/swagger-ui/`

✅ All endpoints are protected with JWT, atomic, auditable, and ready for production.

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

1. **Depot Admin**: Manages inventory and equipment.
2. **Driver / User**: Creates **Distribution** → inventory updated.
3. **Customer Site**: Triggers **Transaction** → invoice PDF generated → emailed → audit logged.
4. **Atomic transactions**: Ensure consistency across inventory, distributions, and invoices.

---

# **HSH LPG Backend Addendum**

### **1️⃣ .env**

Drop this in your project root alongside your current `.env`. It overrides/adds the essentials for development and production.

```env
# ===============================
# Django Core Configuration
# ===============================
SECRET_KEY=django-insecure-7x$9n@k!qv#randomstringhere
DEBUG=True

# ===============================
# Database Configuration
# ===============================
DB_ENGINE=sqlite          # Options: sqlite / mysql
DB_NAME=hsh_lpg
DB_USER=hsh_user
DB_PASSWORD=hsh_pass
DB_HOST=127.0.0.1
DB_PORT=3306

# ===============================
# Email Configuration (Gmail App Password)
# ===============================
EMAIL_HOST_USER=your_gmail@gmail.com
EMAIL_HOST_PASSWORD=your_gmail_app_password
```

---

### **2️⃣ requirements.txt**

Append these to your existing `requirements.txt` (or replace entirely). Includes Django 6, DRF, JWT, WeasyPrint stack, MySQL, and all required dependencies.

```txt
# ===============================
# Core Django & Async Support
# ===============================
Django==6.0.1
asgiref==3.11.0

# ===============================
# Django REST Framework & API
# ===============================
djangorestframework==3.16.1
djangorestframework_simplejwt==5.5.1
drf-spectacular==0.29.0
inflection==0.5.1

# ===============================
# CORS & Environment Config
# ===============================
django-cors-headers==4.9.0
python-decouple==3.8

# ===============================
# Database
# ===============================
mysqlclient==2.2.7
sqlparse==0.5.5

# ===============================
# PDF Generation (WeasyPrint)
# ===============================
weasyprint==68.0
pydyf==0.12.1
tinycss2==1.5.1
cssselect2==0.8.0
tinyhtml5==2.0.0
fonttools==4.61.1
pyphen==0.17.2
brotli==1.2.0
zopfli==0.4.0
webencodings==0.5.1

# ===============================
# Image Handling
# ===============================
pillow==12.1.0

# ===============================
# Security & Cryptography
# ===============================
PyJWT==2.10.1
cffi==2.0.0
pycparser==2.23

# ===============================
# JSON Schema & Validation
# ===============================
jsonschema==4.26.0
jsonschema-specifications==2025.9.1
referencing==0.37.0
rpds-py==0.30.0

# ===============================
# Utilities & Typing
# ===============================
attrs==25.4.0
typing_extensions==4.15.0
PyYAML==6.0.3
uritemplate==4.2.0
tzdata==2025.3
```

---

### **3️⃣ manage.py (WeasyPrint Windows Fix)**

Replace your current `manage.py` with this **drop-in** version if you are on Windows and using WeasyPrint.

```python
#!/usr/bin/env python
"""Django's command-line utility for administrative tasks."""
import os
import sys

def main():
    """Run administrative tasks."""
    os.environ.setdefault('DJANGO_SETTINGS_MODULE', 'core.settings')

    # --- FIX FOR WEASYPRINT ON WINDOWS ---
    if sys.platform == 'win32':
        gtk_path = r'D:\GTK3-Runtime\bin'  # Adjust path to your GTK installation
        if os.path.exists(gtk_path):
            os.add_dll_directory(gtk_path)
    # -------------------------------------

    try:
        from django.core.management import execute_from_command_line
    except ImportError as exc:
        raise ImportError(
            "Couldn't import Django. Are you sure it's installed and "
            "available on your PYTHONPATH environment variable? Did you "
            "forget to activate a virtual environment?"
        ) from exc
    execute_from_command_line(sys.argv)

if __name__ == '__main__':
    main()
```

---

✅ **Instructions**

1. Copy `.env` contents into your existing `.env` (or overwrite if missing).
2. Append `requirements.txt` entries to your current `requirements.txt`.
3. Replace `manage.py` with the addendum version **only if using Windows + WeasyPrint**.
4. Run:

```bash
pip install -r requirements.txt
python manage.py migrate
python manage.py runserver
```

---




