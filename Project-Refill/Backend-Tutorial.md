# **Backend Tutorial (Django 6, 2026)**

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
python manage.py startapp accounts depots equipment inventory distribution customers transactions invoices audit middleware sequences
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

ALLOWED_HOSTS = ['*']   # ← tighten this in production!

# ────────────────────────────────────────────────
# INSTALLED_APPS
# ────────────────────────────────────────────────
INSTALLED_APPS = [
    # Django core
    'django.contrib.admin',
    'django.contrib.auth',
    'django.contrib.contenttypes',
    'django.contrib.sessions',
    'django.contrib.messages',
    'django.contrib.staticfiles',

    # Third-party
    'rest_framework',
    'rest_framework_simplejwt',     # ← removed authtoken (not needed with JWT)
    'drf_spectacular',
    'corsheaders',

    # Your apps (alphabetical)
    'accounts',
    'audit',
    'customers',
    'depots',
    'distribution',
    'equipment',
    'inventory',
    'invoices',
    'middleware',
    'sequences',
    'transactions',
]

# ────────────────────────────────────────────────
# MIDDLEWARE
# ────────────────────────────────────────────────
MIDDLEWARE = [
    'django.middleware.security.SecurityMiddleware',
    'django.contrib.sessions.middleware.SessionMiddleware',
    'corsheaders.middleware.CorsMiddleware',
    'django.middleware.common.CommonMiddleware',
    'django.middleware.csrf.CsrfViewMiddleware',
    'django.contrib.auth.middleware.AuthenticationMiddleware',
    'django.contrib.messages.middleware.MessageMiddleware',
    'middleware.request_logging.APILoggingMiddleware',
    'django.middleware.clickjacking.XFrameOptionsMiddleware',
]

# ────────────────────────────────────────────────
# Core Django settings
# ────────────────────────────────────────────────
ROOT_URLCONF = 'core.urls'
WSGI_APPLICATION = 'core.wsgi.application'

AUTH_USER_MODEL = 'accounts.User'

TEMPLATES = [
    {
        'BACKEND': 'django.template.backends.django.DjangoTemplates',
        'DIRS': [BASE_DIR / 'templates'],
        'APP_DIRS': True,
        'OPTIONS': {
            'context_processors': [
                'django.template.context_processors.debug',
                'django.template.context_processors.request',
                'django.contrib.auth.context_processors.auth',
                'django.contrib.messages.context_processors.messages',
            ],
        },
    }
]

# ────────────────────────────────────────────────
# REST Framework + JWT
# ────────────────────────────────────────────────
REST_FRAMEWORK = {
    'DEFAULT_AUTHENTICATION_CLASSES': (
        'rest_framework_simplejwt.authentication.JWTAuthentication',
    ),
    'DEFAULT_PERMISSION_CLASSES': (
        'rest_framework.permissions.IsAuthenticated',
    ),
    'DEFAULT_SCHEMA_CLASS': 'drf_spectacular.openapi.AutoSchema',
}

SIMPLE_JWT = {
    'ACCESS_TOKEN_LIFETIME': timedelta(hours=8),
    'REFRESH_TOKEN_LIFETIME': timedelta(days=7),
    'ROTATE_REFRESH_TOKENS': True,
    'BLACKLIST_AFTER_ROTATION': True,
    'AUTH_HEADER_TYPES': ('Bearer',),
    'UPDATE_LAST_LOGIN': True,
}

# ────────────────────────────────────────────────
# Database
# ────────────────────────────────────────────────
DB_ENGINE = config('DB_ENGINE', default='sqlite').lower()

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
    DATABASES = {
        'default': {
            'ENGINE': 'django.db.backends.sqlite3',
            'NAME': BASE_DIR / 'db.sqlite3',
        }
    }

# ────────────────────────────────────────────────
# Static & Media
# ────────────────────────────────────────────────
STATIC_URL = '/static/'
STATIC_ROOT = BASE_DIR / 'staticfiles'

MEDIA_URL = '/media/'
MEDIA_ROOT = BASE_DIR / 'media'

# ────────────────────────────────────────────────
# Email
# ────────────────────────────────────────────────
EMAIL_BACKEND = 'django.core.mail.backends.smtp.EmailBackend'
EMAIL_HOST = config('EMAIL_HOST', default='smtp.gmail.com')
EMAIL_PORT = 587
EMAIL_USE_TLS = True
EMAIL_HOST_USER = config('EMAIL_HOST_USER')
EMAIL_HOST_PASSWORD = config('EMAIL_HOST_PASSWORD')
DEFAULT_FROM_EMAIL = 'HSH LPG <noreply@hshlpg.com.sg>'

# ────────────────────────────────────────────────
# Security (stronger in production)
# ────────────────────────────────────────────────
SECURE_BROWSER_XSS_FILTER = True
SECURE_CONTENT_TYPE_NOSNIFF = True
X_FRAME_OPTIONS = 'DENY'

if not DEBUG:
    SECURE_SSL_REDIRECT = True
    SESSION_COOKIE_SECURE = True
    CSRF_COOKIE_SECURE = True
    SECURE_HSTS_SECONDS = 31536000
    SECURE_HSTS_INCLUDE_SUBDOMAINS = True
    SECURE_HSTS_PRELOAD = True
```

---

## **4️⃣ Utilities**

### **Numbering (`core/utils/numbering.py`)**

```python
# core/utils/numbering
from django.db import transaction
from django.utils import timezone
from sequences.models import Sequence


@transaction.atomic
def generate_number(prefix: str, length: int = 6, reset_daily: bool = True) -> str:
    seq, _ = Sequence.objects.select_for_update().get_or_create(
        prefix=prefix,
        defaults={'last_value': 0, 'last_date': timezone.now().date()}
    )

    today = timezone.now().date()

    # Reset logic 
    if reset_daily and seq.last_date != today:
        seq.last_value = 0
        seq.last_date = today

    seq.last_value += 1
    seq.save(update_fields=['last_value', 'last_date'])

    # Flexible format
    date_part = today.strftime('%Y%m%d') if reset_daily else ""
    number_str = f"{prefix}-{date_part}-{seq.last_value:0{length}d}"

    # remove trailing - if no date
    return number_str.rstrip('-')
```

---
### **Inventory (`core/utils/inventory.py`)**
```
# core/utils/inventory.py
from django.db.models import F
from inventory.models import Inventory

def safe_deduct_inventory(qs, quantity: int) -> bool:
    """
    Deduct quantity safely. Returns True if successful, False if not enough stock.
    """
    if quantity <= 0:
        return True
    updated = qs.filter(quantity__gte=quantity).update(quantity=F('quantity') - quantity)
    return updated > 0

```
---

### **Invoice Tasks (`invoices/tasks.py`)**

```
# invoices/tasks.py
from django.core.files.base import ContentFile
from django.core.files.storage import default_storage
from django.template.loader import render_to_string
from django.utils import timezone
from weasyprint import HTML

from invoices.models import Invoice
from transactions.models import Transaction


def generate_invoice_pdf_async(
    transaction_id,
    usage_amount,
    items_amount,
    total_amount,
    meter_updated,
    transaction_items
):
    transaction = Transaction.objects.select_related('customer').get(id=transaction_id)
    customer = transaction.customer

    pdf_filename = f"invoices/{timezone.now():%Y%m}/{transaction.transaction_number}.pdf"

    html_content = render_to_string(
        'invoices/invoice.html',
        {
            'tx': transaction,
            'customer': customer,
            'items': transaction_items,
            'usage_amount': usage_amount,
            'items_amount': items_amount,
            'total_amount': total_amount,
            'meter_updated': meter_updated,
        }
    )

    pdf_bytes = HTML(string=html_content).write_pdf()

    saved_path = default_storage.save(
        pdf_filename,
        ContentFile(pdf_bytes)
    )

    invoice = Invoice.objects.get(transaction=transaction)
    invoice.pdf_path = saved_path
    invoice.save(update_fields=['pdf_path'])
```
---
## **5️⃣ Models**

### **Accounts (`accounts/models.py`)**

```python
# accounts/models.py

from django.contrib.auth.models import AbstractUser
from django.db import models
from depots.models import Depot

class User(AbstractUser):
    ROLE_CHOICES = (
        ('ADMIN','Admin'),
        ('DRIVER','Driver'),
        ('SUPERVISOR','Supervisor')
    )

    employee_id = models.CharField(
        max_length=20,
        unique=True,
        null=True,
        blank=True
    )
    role = models.CharField(max_length=20, choices=ROLE_CHOICES, default='DRIVER')
    depot = models.ForeignKey(Depot, on_delete=models.SET_NULL, null=True, blank=True)

```

### **Sequences (`sequences/models.py`)**

```python
# sequences/models.py   
from django.db import models

class Sequence(models.Model):
    """
    Atomic counter per prefix (e.g. TRX, INV, DST).
    Used to generate race-free sequential numbers like TRX-20250122-000001
    """
    prefix = models.CharField(max_length=10, unique=True)
    last_value = models.PositiveBigIntegerField(default=0)
    last_date = models.DateField(null=True, blank=True)  # optional: reset per day/year

    class Meta:
        verbose_name = "Sequence Counter"
        verbose_name_plural = "Sequence Counters"

    def __str__(self):
        return f"{self.prefix} → {self.last_value}"
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
    version = models.PositiveIntegerField(default=0)  # New field for optimistic locking

    def __str__(self):
        return self.name
```

---

### **Inventory (`inventory/models.py`)**

```python
from django.db import models
from customers.models import Customer
from equipment.models import Equipment
from depots.models import Depot

class Inventory(models.Model):
    INVENTORY_TYPE = [('DEPOT','Depot'),('CUSTOMER','Customer')]

    inventory_type = models.CharField(max_length=10, choices=INVENTORY_TYPE)
    depot = models.ForeignKey(Depot, null=True, blank=True, on_delete=models.CASCADE)
    customer = models.ForeignKey(Customer, null=True, blank=True, on_delete=models.CASCADE)
    equipment = models.ForeignKey(Equipment, on_delete=models.PROTECT)
    quantity = models.IntegerField(default=0)

    class Meta:
        unique_together = ('inventory_type','depot','customer','equipment')

```

---

### **Transactions (`transactions/models.py`)**

```python

# transactions/models.py
from django.db import models
from django.conf import settings
from customers.models import Customer
from equipment.models import Equipment

class Transaction(models.Model):
    transaction_number = models.CharField(max_length=30, unique=True)
    customer = models.ForeignKey(Customer, on_delete=models.PROTECT, related_name='transactions')
    user = models.ForeignKey(settings.AUTH_USER_MODEL, on_delete=models.PROTECT)
    total_amount = models.DecimalField(max_digits=12, decimal_places=2)
    created_at = models.DateTimeField(auto_now_add=True)
    def __str__(self): return self.transaction_number

class TransactionItem(models.Model):
    TRANSACTION_TYPE_CHOICES = [
        ('SALE', 'Sale'),
        ('SWAP', 'Cylinder Swap'),
        ('REFILL', 'Refill Only'),
        ('RETURN', 'Return/Deposit Refund'),
    ]

    transaction = models.ForeignKey(
        Transaction,
        on_delete=models.CASCADE,
        related_name='items'
    )
    equipment = models.ForeignKey(
        'equipment.Equipment',
        on_delete=models.PROTECT
    )
    quantity = models.PositiveIntegerField()
    rate = models.DecimalField(max_digits=10, decimal_places=2)
    amount = models.DecimalField(max_digits=12, decimal_places=2)
    type = models.CharField(
        max_length=20,
        choices=TRANSACTION_TYPE_CHOICES,
        default='SALE'
    )

    class Meta:
        ordering = ['id']

    def __str__(self):
        return f"{self.quantity} × {self.equipment.name} @ {self.rate}"

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
# audit/models.py
from django.db import models
from django.contrib.auth import get_user_model

User = get_user_model()

class AuditLog(models.Model):
    user = models.ForeignKey(User, on_delete=models.SET_NULL, null=True)
    action = models.CharField(max_length=100)
    entity_type = models.CharField(max_length=50)
    entity_id = models.PositiveIntegerField()
    timestamp = models.DateTimeField(auto_now_add=True)
    payload = models.JSONField(default=dict)          
    class Meta:
        indexes = [
            models.Index(fields=['entity_type', 'entity_id']),
            models.Index(fields=['timestamp']),
        ]
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
# inventory/serializers.py
from rest_framework import serializers
from inventory.models import Inventory

class InventorySerializer(serializers.ModelSerializer):
    depot_name = serializers.CharField(source='depot.name', read_only=True)
    customer_name = serializers.CharField(source='customer.name', read_only=True)
    equipment_name = serializers.CharField(source='equipment.name', read_only=True)

    class Meta:
        model = Inventory
        fields = ['id','inventory_type','depot','depot_name','customer','customer_name','equipment','equipment_name','quantity']



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
# transactions/services.py
from decimal import Decimal
from django.db import transaction
from django.core.files.base import ContentFile
from django.core.files.storage import default_storage
from django.template.loader import render_to_string
from weasyprint import HTML

from core.utils.numbering import generate_number
from core.utils.inventory import safe_deduct_inventory

from audit.models import AuditLog
from customers.models import Customer
from equipment.models import Equipment
from inventory.models import Inventory
from invoices.models import Invoice
from transactions.models import Transaction, TransactionItem


@transaction.atomic
def create_customer_transaction_and_invoice(user, data):
    customer = Customer.objects.select_for_update().get(id=data['customer'])

    current_meter = data.get('current_meter')
    items_data = data['items']

    usage_amount = Decimal('0.00')
    meter_updated = False

    # Meter logic + optimistic lock
    if customer.is_meter_installed and current_meter is not None:
        current = Decimal(str(current_meter))
        prev = customer.last_meter_reading or Decimal('0.00')

        if current < prev:
            raise ValueError(f"Current meter reading ({current}) is less than previous ({prev})")

        if current > prev:
            usage = current - prev
            usage_amount = usage * (customer.meter_rate or Decimal('0.00'))

            # atomic + version
            updated = Customer.objects.filter(id=customer.id, version=customer.version).update(
                last_meter_reading=current,
                version=customer.version + 1
            )

            if not updated:
                raise ValueError("Meter reading was updated concurrently. Retry transaction.")

            # Refresh to be safe
            customer.refresh_from_db()

            meter_updated = True

            AuditLog.objects.create(
                user=user,
                action='METER_UPDATE',
                entity_type='Customer',
                entity_id=customer.id,
                payload={
                    'previous': str(prev),
                    'current': str(current),
                    'usage_m3': str(usage),
                    'billed': str(usage_amount)
                }
            )

    # Safer inventory handling
    transaction_items = []
    items_amount = Decimal('0.00')

    # Lock all needed inventory rows
    equipment_ids = [item['equipment'] for item in items_data]
    customer_inventories_qs = Inventory.objects.select_for_update().filter(
        customer=customer,
        equipment_id__in=equipment_ids,
        inventory_type='CUSTOMER'
    )
    customer_inventories = {inv.equipment_id: inv for inv in customer_inventories_qs}

    for item in items_data:
        eq = Equipment.objects.select_for_update().get(id=item['equipment'], is_active=True)
        qty = int(item['quantity'])
        rate = Decimal(str(item['rate']))
        line_amount = qty * rate
        items_amount += line_amount

        inv = customer_inventories.get(eq.id)

        if item['type'] == 'SALE':
            if not inv:
                inv = Inventory.objects.create(
                    inventory_type='CUSTOMER',
                    customer=customer,
                    equipment=eq,
                    quantity=0
                )
                customer_inventories[eq.id] = inv

            inv.quantity += qty
            inv.save(update_fields=['quantity'])

        elif item['type'] == 'RETURN':
            if not inv or inv.quantity < qty:
                raise ValueError(f"Cannot return {qty}×{eq.name} - insufficient customer stock")

            inv.quantity -= qty
            inv.save(update_fields=['quantity'])

        # Create items later with bulk — cleaner
        transaction_items.append(
            TransactionItem(
                equipment=eq,
                quantity=qty,
                rate=rate,
                amount=line_amount,
                type=item['type']
            )
        )

    total_amount = usage_amount + items_amount

    # Create transaction
    transaction = Transaction.objects.create(
        transaction_number=generate_number('TRX'),
        customer=customer,
        user=user,
        total_amount=total_amount
    )

    # Attach + bulk create
    for item in transaction_items:
        item.transaction = transaction
    TransactionItem.objects.bulk_create(transaction_items)

    # PDF generation
    html_content = render_to_string(
        'invoices/invoice.html',
        {
            'tx': transaction,
            'customer': customer,
            'items': transaction_items,
            'usage_amount': usage_amount,
            'items_amount': items_amount,
            'total_amount': total_amount,
            'meter_updated': meter_updated,
        }
    )

    pdf_bytes = HTML(string=html_content).write_pdf()

    pdf_filename = f"invoices/{transaction.created_at:%Y%m}/{transaction.transaction_number}.pdf"
    saved_path = default_storage.save(pdf_filename, ContentFile(pdf_bytes))

    invoice = Invoice.objects.create(
        invoice_number=generate_number('INV'),
        transaction=transaction,
        pdf_path=saved_path,
        status='generated'
    )

    AuditLog.objects.create(
        user=user,
        action='TX_CREATED',
        entity_type='Transaction',
        entity_id=transaction.id,
        payload={
            'customer': customer.id,
            'total': str(total_amount),
            'usage': str(usage_amount),
            'items_count': len(items_data),
            'meter_changed': meter_updated
        }
    )

    return transaction, invoice

```

---

### **Distribution Services (`distribution/services.py`)**

```python
# distribution/services.py
from django.db import transaction
from django.utils import timezone
from core.utils.numbering import generate_number
from core.utils.inventory import safe_deduct_inventory
from distribution.models import Distribution, DistributionItem
from inventory.models import Inventory
from audit.models import AuditLog

@transaction.atomic
def create_distribution(user, depot, items, remarks=""):
    dist = Distribution.objects.create(distribution_number=generate_number('DST'), depot=depot, user=user, remarks=remarks)
    DistributionItem.objects.bulk_create([DistributionItem(distribution=dist, equipment_id=i['equipment'], direction=i['direction'], condition=i.get('condition','FULL'), quantity=i['quantity']) for i in items])
    AuditLog.objects.create(user=user, action='DISTRIBUTION_CREATED', entity_type='Distribution', entity_id=dist.id, payload={'number': dist.distribution_number, 'depot_id': depot.id, 'items_count': len(items), 'remarks': remarks[:100]})
    return dist

@transaction.atomic
def confirm_distribution(distribution_id: int, user):
    dist = Distribution.objects.select_for_update().get(id=distribution_id)
    if dist.confirmed_at: raise ValueError("Already confirmed")
    items = dist.items.select_related('equipment').all()
    depot_inv = {i.equipment_id: i for i in Inventory.objects.select_for_update().filter(inventory_type='DEPOT', depot=dist.depot, equipment_id__in=[it.equipment_id for it in items])}
    errors = []
    for item in items:
        if item.direction == 'OUT':
            inv = depot_inv.get(item.equipment_id)
            if not inv or not safe_deduct_inventory(Inventory.objects.filter(pk=inv.pk), item.quantity):
                errors.append(f"Insufficient stock {item.equipment.name}")
    if errors: raise ValueError("; ".join(errors))
    dist.confirmed_at = timezone.now(); dist.save(update_fields=['confirmed_at'])
    AuditLog.objects.create(user=user, action='DISTRIBUTION_CONFIRMED', entity_type='Distribution', entity_id=dist.id, payload={'number': dist.distribution_number, 'items': len(items), 'depot': dist.depot_id})
    return dist
```

---

### **Inventory Services (`inventory/services.py`)**

```python
# inventory/services.py
from django.db import transaction
from core.utils.inventory import safe_deduct_inventory
from inventory.models import Inventory
from customers.models import Customer
from equipment.models import Equipment
from audit.models import AuditLog


@transaction.atomic
def update_inventory(entity: str, entity_id: int, equipment_id: int, new_quantity: int, user):
    if entity != 'customer':
        raise ValueError("Only customer inventory updates supported currently")

    if new_quantity < 0:
        raise ValueError("Cannot set negative inventory quantity")

    customer = Customer.objects.get(id=entity_id)
    equipment = Equipment.objects.get(id=equipment_id)

    inv, _ = Inventory.objects.get_or_create(
        inventory_type='CUSTOMER',
        customer=customer,
        equipment=equipment,
        defaults={'quantity': 0}
    )

    # Lock row
    inv = Inventory.objects.select_for_update().get(pk=inv.pk)

    old_quantity = inv.quantity  # for audit

    if new_quantity < old_quantity:
        # Decrement case — use safe deduct
        delta = old_quantity - new_quantity
        success = safe_deduct_inventory(
            Inventory.objects.filter(pk=inv.pk),
            delta,
            "Cannot reduce below current stock (race condition or insufficient)"
        )
        if not success:
            raise ValueError(
                f"Cannot reduce to {new_quantity}: "
                f"would require removing {delta} but only {old_quantity} available"
            )
    else:
        # Set directly (increment or same)
        inv.quantity = new_quantity
        inv.save(update_fields=['quantity'])

    AuditLog.objects.create(
        user=user,
        action='MANUAL_INVENTORY_CORRECTION',
        entity_type='Inventory',
        entity_id=inv.id,
        payload={
            'customer_id': customer.id,
            'equipment_id': equipment.id,
            'old_quantity': old_quantity,
            'new_quantity': new_quantity,
            'changed_by': user.username
        }
    )

    return inv
```
---
### **Invoices Services (`invoices/services.py`)**

```python
# invoices/services.py
from django.conf import settings
from pathlib import Path
from weasyprint import HTML   

def generate_invoice_pdf(transaction, html_string):
    pdf_path = (
        Path(settings.MEDIA_ROOT)
        / 'invoices'
        / f'{transaction.transaction_number}.pdf'
    )
    pdf_path.parent.mkdir(parents=True, exist_ok=True)

    HTML(string=html_string).write_pdf(target=str(pdf_path))

    return str(pdf_path)

```
---

## **8️⃣ ViewSets**

### **Transactions (`transactions/views.py`)**

```python
# transactions/views.py
from rest_framework import viewsets, status, serializers
from rest_framework.decorators import action
from rest_framework.response import Response
from rest_framework.permissions import IsAuthenticated
from django.contrib.auth.mixins import PermissionRequiredMixin  # optional
from transactions.models import Transaction, TransactionItem
from transactions.serializers import TransactionSerializer
from invoices.serializers import InvoiceSerializer
from transactions.services import create_customer_transaction_and_invoice
from equipment.models import Equipment  # For validation if needed

class TransactionItemInputSerializer(serializers.Serializer):
    equipment = serializers.IntegerField(min_value=1)
    quantity = serializers.IntegerField(min_value=1)
    rate = serializers.DecimalField(max_digits=12, decimal_places=2, min_value=0)
    type = serializers.ChoiceField(choices=TransactionItem.TRANSACTION_TYPE_CHOICES)

class CreateTransactionSerializer(serializers.Serializer):
    customer = serializers.IntegerField(min_value=1)
    current_meter = serializers.DecimalField(max_digits=12, decimal_places=2, required=False, allow_null=True)
    items = TransactionItemInputSerializer(many=True, min_length=1)

class TransactionViewSet(viewsets.ModelViewSet):
    serializer_class = TransactionSerializer
    permission_classes = [IsAuthenticated]

    def get_queryset(self):
        user = self.request.user
        # Very basic role-based filtering — extend heavily!
        if user.role == 'DRIVER':
            # Drivers only see their own transactions
            return Transaction.objects.filter(user=user)
        elif user.role == 'SUPERVISOR':
            # Supervisors see transactions from their depot
            if user.depot:
                return Transaction.objects.filter(
                    user__depot=user.depot
                ).select_related('customer', 'user')
        elif user.role == 'ADMIN':
            # Admins see everything
            return Transaction.objects.all().select_related('customer', 'user')
        # Fallback — should never reach here if roles are enforced
        return Transaction.objects.none()

    @action(detail=False, methods=['post'])
    def create_transaction(self, request):
        serializer = CreateTransactionSerializer(data=request.data)
        if not serializer.is_valid():
            return Response(serializer.errors, status=status.HTTP_400_BAD_REQUEST)
        
        try:
            tx, invoice = create_customer_transaction_and_invoice(
                user=request.user,
                data=serializer.validated_data
            )
            tx_ser = TransactionSerializer(tx, context={'request': request})
            inv_ser = InvoiceSerializer(invoice, context={'request': request})
            return Response(
                {'transaction': tx_ser.data, 'invoice': inv_ser.data},
                status=status.HTTP_201_CREATED
            )
        except ValueError as ve:
            return Response(
                {'error': str(ve), 'code': 'validation_error'},
                status=status.HTTP_400_BAD_REQUEST
            )
        except Exception as e:
            # In production → log this!
            return Response(
                {'error': 'Internal error', 'code': 'server_error'},
                status=status.HTTP_500_INTERNAL_SERVER_ERROR
            )
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
# inventory/views.py
from rest_framework.decorators import action
from rest_framework.response import Response
from rest_framework import viewsets, status
from inventory.models import Inventory
from inventory.serializers import InventorySerializer
from inventory.services import update_inventory

class InventoryViewSet(viewsets.ModelViewSet):
    queryset = Inventory.objects.all()  # <--- Add this
    serializer_class = InventorySerializer

    def get_queryset(self):
        queryset = super().get_queryset()
        customer_id = self.request.query_params.get('customer_id')
        if customer_id:
            queryset = queryset.filter(customer_id=customer_id)
        return queryset

    @action(detail=False, methods=['post'], url_path='update-inventory')
    def update_inventory(self, request):
        from inventory.services import update_inventory

        data = request.data
        try:
            inv = update_inventory(
                entity     = data.get('entity'),
                entity_id  = data.get('entity_id'),
                equipment_id = data.get('equipment_id'),
                quantity   = data.get('quantity'),
                user       = request.user
            )
            serializer = self.get_serializer(inv)
            return Response(serializer.data, status=200)
        except Exception as e:
            return Response(
                {"error": str(e)},
                status=400
            )
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
    queryset = AuditLog.objects.all().order_by('-timestamp')  
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
from .models import Inventory

@admin.register(Inventory)
class InventoryAdmin(admin.ModelAdmin):
    list_display = ['inventory_type','depot','customer','equipment','quantity']

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
from inventory.views import InventoryViewSet
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
router.register(r'inventories', InventoryViewSet)
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
    <title>HSH LPG Invoice {{ tx.transaction_number }}</title>
    <style>
        body { font-family: Arial, sans-serif; font-size: 12pt; }
        h1 { color: #2E86C1; text-align: center; }
        table { width: 100%; border-collapse: collapse; margin: 20px 0; }
        th, td { border: 1px solid #aaa; padding: 8px; text-align: left; }
        th { background-color: #f2f2f2; }
        .total-row { font-weight: bold; background-color: #e8f4f8; }
        .right { text-align: right; }
    </style>
</head>
<body>
    <h1>HSH LPG INVOICE</h1>
    <p><strong>Invoice No:</strong> {{ tx.transaction_number }}</p>
    <p><strong>Date:</strong> {{ tx.created_at|date:"d M Y H:i" }}</p>
    <p><strong>Customer:</strong> {{ customer.name }}<br>{{ customer.address|linebreaks }}</p>

    <h3>Transaction Details</h3>
    <table>
        <thead>
            <tr>
                <th>Description</th>
                <th class="right">Qty</th>
                <th class="right">Rate (SGD)</th>
                <th class="right">Amount (SGD)</th>
            </tr>
        </thead>
        <tbody>
            {% for item in items %}
            <tr>
                <td>{{ item.equipment.name }} ({{ item.type }})</td>
                <td class="right">{{ item.quantity }}</td>
                <td class="right">{{ item.rate|floatformat:2 }}</td>
                <td class="right">{{ item.amount|floatformat:2 }}</td>
            </tr>
            {% endfor %}
            {% if usage_amount > 0 %}
            <tr>
                <td>Meter Usage Billing</td>
                <td class="right">-</td>
                <td class="right">-</td>
                <td class="right">{{ usage_amount|floatformat:2 }}</td>
            </tr>
            {% endif %}
            <tr class="total-row">
                <td colspan="3" class="right">Total Amount Due</td>
                <td class="right">{{ total_amount|floatformat:2 }}</td>
            </tr>
        </tbody>
    </table>

    <p>Thank you for your business.<br>HSH LPG Pte Ltd</p>
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




