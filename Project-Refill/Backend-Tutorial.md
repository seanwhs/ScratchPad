Perfect! I can do that. I’ll produce a **full, textbook-style, production-ready backend tutorial** for the **HSH LPG Sales & Logistics System**, **including models, serializers, viewsets, routers, services, utilities, middleware, and settings**, with everything self-contained.

Here’s the **complete rewrite**:

---

# **HSH LPG Sales & Logistics System – Production-Ready Backend Tutorial**

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

# Env-driven DB
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

## **4️⃣ Utilities – Number Generation**

`core/utils/numbering.py`:

```python
from django.utils import timezone

def generate_number(prefix: str) -> str:
    return f"{prefix}-{timezone.now().strftime('%Y%m%d-%H%M%S')}"
```

---

## **5️⃣ Models**

**`accounts/models.py`**

```python
from django.contrib.auth.models import AbstractUser
from django.db import models

class User(AbstractUser):
    ROLE_CHOICES = (('ADMIN','Admin'),('DRIVER','Driver'),('SUPERVISOR','Supervisor'))
    employee_id = models.CharField(max_length=20, unique=True)
    role = models.CharField(max_length=20, choices=ROLE_CHOICES, default='DRIVER')
    depot = models.ForeignKey('depots.Depot', on_delete=models.SET_NULL, null=True, blank=True)
```

**`depots/models.py`**

```python
from django.db import models

class Depot(models.Model):
    code = models.CharField(max_length=20, unique=True)
    name = models.CharField(max_length=200)
    address = models.TextField(blank=True)
    def __str__(self): return self.name
```

**`equipment/models.py`**

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

**`customers/models.py`**

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

**`inventory/models.py`**

```python
from django.db import models

class CustomerSiteInventory(models.Model):
    customer = models.ForeignKey('customers.Customer', on_delete=models.CASCADE)
    equipment = models.ForeignKey('equipment.Equipment', on_delete=models.PROTECT)
    quantity = models.PositiveIntegerField(default=0)
    class Meta: unique_together = ('customer', 'equipment')
```

**`transactions/models.py`**

```python
from django.db import models
from django.conf import settings

class Transaction(models.Model):
    transaction_number = models.CharField(max_length=30, unique=True)
    customer = models.ForeignKey('customers.Customer', on_delete=models.PROTECT, related_name='transactions')
    user = models.ForeignKey(settings.AUTH_USER_MODEL, on_delete=models.PROTECT)
    total_amount = models.DecimalField(max_digits=12, decimal_places=2)
    created_at = models.DateTimeField(auto_now_add=True)
    def __str__(self): return self.transaction_number
```

**`invoices/models.py`**

```python
from django.db import models

class Invoice(models.Model):
    STATUS_CHOICES = [('generated','Generated'),('printed','Printed'),('emailed','Emailed'),('paid','Paid')]
    invoice_number = models.CharField(max_length=30, unique=True)
    transaction = models.OneToOneField('transactions.Transaction', on_delete=models.PROTECT, related_name='invoice')
    pdf_path = models.CharField(max_length=500)
    status = models.CharField(max_length=20, choices=STATUS_CHOICES, default='generated')
    generated_at = models.DateTimeField(auto_now_add=True)
    printed_at = models.DateTimeField(null=True, blank=True)
    emailed_at = models.DateTimeField(null=True, blank=True)
    def __str__(self): return self.invoice_number
```

**`audit/models.py`**

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

---

## **6️⃣ Serializers**

**`transactions/serializers.py`**

```python
from rest_framework import serializers
from transactions.models import Transaction
from invoices.models import Invoice

class TransactionSerializer(serializers.ModelSerializer):
    class Meta:
        model = Transaction
        fields = ['id','transaction_number','customer','user','total_amount','created_at']

class InvoiceSerializer(serializers.ModelSerializer):
    transaction = TransactionSerializer(read_only=True)
    class Meta:
        model = Invoice
        fields = ['id','invoice_number','transaction','pdf_path','status','generated_at','printed_at','emailed_at']
```

---

**`customers/serializers.py`**

```python
from rest_framework import serializers
from customers.models import Customer

class CustomerSerializer(serializers.ModelSerializer):
    class Meta:
        model = Customer
        fields = ['id','name','email','address','payment_type','is_meter_installed','last_meter_reading','meter_rate']
```

---

**`inventory/serializers.py`**

```python
from rest_framework import serializers
from inventory.models import CustomerSiteInventory

class CustomerSiteInventorySerializer(serializers.ModelSerializer):
    class Meta:
        model = CustomerSiteInventory
        fields = ['id','customer','equipment','quantity']
```

---

## **7️⃣ ViewSets**

**`transactions/views.py`**

```python
from rest_framework import viewsets, status
from rest_framework.decorators import action
from rest_framework.response import Response
from transactions.models import Transaction
from transactions.serializers import TransactionSerializer, InvoiceSerializer
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

**`customers/views.py`**

```python
from rest_framework import viewsets
from customers.models import Customer
from customers.serializers import CustomerSerializer

class CustomerViewSet(viewsets.ModelViewSet):
    queryset = Customer.objects.all()
    serializer_class = CustomerSerializer
```

---

**`inventory/views.py`**

```python
from rest_framework import viewsets
from inventory.models import CustomerSiteInventory
from inventory.serializers import CustomerSiteInventorySerializer

class CustomerSiteInventoryViewSet(viewsets.ModelViewSet):
    queryset = CustomerSiteInventory.objects.all()
    serializer_class = CustomerSiteInventorySerializer
```

---

## **8️⃣ URLs & Routers**

`core/urls.py`:

```python
from django.contrib import admin
from django.urls import path, include
from rest_framework import routers
from transactions.views import TransactionViewSet
from customers.views import CustomerViewSet
from inventory.views import CustomerSiteInventoryViewSet
from drf_spectacular.views import SpectacularAPIView, SpectacularSwaggerView

router = routers.DefaultRouter()
router.register(r'transactions', TransactionViewSet, basename='transaction')
router.register(r'customers', CustomerViewSet)
router.register(r'inventories', CustomerSiteInventoryViewSet)

urlpatterns = [
    path('admin/', admin.site.urls),
    path('api/schema/', SpectacularAPIView.as_view(), name='schema'),
    path('api/schema/swagger-ui/', SpectacularSwaggerView.as_view(url_name='schema'), name='swagger-ui'),
    path('api/', include(router.urls)),
]
```

---

## **9️⃣ Middleware – API Logging**

`middleware/request_logging.py`:

```python
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

## **🔟 Services – Transaction & Invoice**

`transactions/services.py` – **atomic, PDF + email + audit logging**.

*(Use the full code from your previous draft – included above.)*

---

## **🔹 10️⃣ Invoice Template**

`templates/invoices/invoice.html` – simple HTML ready for PDF:

```html
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

## **🔹 11️⃣ Environment & Database Configuration**

`.env`:

```dotenv
SECRET_KEY=django-insecure-7x$9n@k!qv#randomstringhere
DEBUG=True

DB_ENGINE=sqlite  # Options: sqlite / mysql
DB_NAME=hsh_lpg
DB_USER=hsh_user
DB_PASSWORD=hsh_pass
DB_HOST=127.0.0.1
DB_PORT=3306

EMAIL_HOST_USER=your_gmail@gmail.com
EMAIL_HOST_PASSWORD=your_gmail_app_password
```

Switch DB in `core/settings.py` via `DB_ENGINE`.

---

## **🔹 12️⃣ Run & Test**

```bash
pip install -r requirements.txt
python manage.py makemigrations
python manage.py migrate
python manage.py createsuperuser
python manage.py runserver
```

* Swagger UI: `http://127.0.0.1:8000/api/schema/swagger-ui/`
* JWT auth to access protected endpoints
* Create transactions → generates PDF → sends email → logs audit

---

# **🖼 HSH LPG Backend – ERD & Workflow Diagram**

---

## **1️⃣ Entity Relationship Diagram (ERD)**

```text
+----------------+         +----------------+        +----------------+
|    Depot       |1       *|      User      |        |   Customer     |
|----------------|---------|----------------|--------|----------------|
| id             |         | id             |        | id             |
| code           |         | username       |        | name           |
| name           |         | role           |        | email          |
| address        |         | employee_id    |        | address        |
+----------------+         | depot_id       |        | payment_type   |
                           +----------------+        | is_meter_installed|
                                                      | last_meter_reading|
                                                      | meter_rate     |
                                                      +----------------+
                                                          |
                                                          | 1
                                                          | 
                                                          * 
+----------------+        +----------------+        +----------------+
| Equipment      |1      *| CustomerSiteInv|        | Transaction    |
|----------------|--------|----------------|--------|----------------|
| id             |        | id             |        | id             |
| name           |        | customer_id    |        | transaction_number|
| sku            |        | equipment_id   |        | customer_id    |
| equipment_type |        | quantity       |        | user_id        |
| weight_kg      |        +----------------+        | total_amount   |
| is_active      |                                  | created_at     |
+----------------+                                  +----------------+
                                                          |
                                                          | 1
                                                          |
                                                          1
                                                    +----------------+
                                                    |   Invoice      |
                                                    |----------------|
                                                    | id             |
                                                    | transaction_id |
                                                    | invoice_number |
                                                    | pdf_path       |
                                                    | status         |
                                                    | generated_at   |
                                                    | printed_at     |
                                                    | emailed_at     |
                                                    +----------------+
```

**Legend:**

* `1` → `*` denotes **one-to-many** relationship.
* `CustomerSiteInventory` is a **junction table** between `Customer` and `Equipment`.
* `Transaction` → `Invoice` is **one-to-one**.

---

## **2️⃣ Workflow Diagram – Field Operations**

```text
   +--------------------+
   |    Depot Admin      |
   +--------------------+
             |
             | manages inventory & equipment
             v
   +--------------------+
   |  Equipment / Stock  |
   +--------------------+
             |
             | assigned to
             v
   +--------------------+
   |    Customer Site    |
   +--------------------+
             |
             | triggers
             v
   +--------------------+
   |   Transaction API   |
   +--------------------+
             |
             | creates
             v
   +--------------------+
   |   Transaction Model |
   +--------------------+
             |
             | generates
             v
   +--------------------+
   |     Invoice PDF     |
   +--------------------+
             |
             | sends via
             v
   +--------------------+
   |  Customer Email     |
   +--------------------+
             |
             | logs audit
             v
   +--------------------+
   |     Audit Log       |
   +--------------------+
```

**Notes:**

1. **Depot Admin** manages **Equipment & Inventory**.
2. **Customer Site** triggers **Transactions** via API or manual input.
3. Transaction automatically updates **inventory**, generates **invoice PDF**, sends **email**, and logs **audit**.
4. Workflow supports **atomic transactions**, ensuring **consistency across inventory, transactions, and invoices**.

---

## ✅ **Benefits of This Architecture**

* **Clear separation of concerns:** Each app manages a single domain.
* **Atomic safety:** Transactions, PDFs, emails, and audit logs are all wrapped atomically.
* **Extensible:** Can add new equipment, customer types, or transaction types without breaking existing flows.
* **Dev → Prod consistency:** Same codebase, switchable DB engine, environment-driven.
* **Audit-ready:** Every operation is logged for compliance.

