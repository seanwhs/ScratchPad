### Full Backend Code Walkthrough  
**HSH LPG Sales & Logistics System – Django 6 + DRF Backend**

The backend is deliberately **field-first**, **transactionally strict**, and **audit-obsessed** — it acts as the single source of truth for physical LPG cylinders and financial reality in environments with unreliable connectivity.

Project structure (standard layout from the tutorial docs):

```
hsh_lpg_system/
├── core/                  # project settings & urls
│   ├── settings.py
│   ├── urls.py
│   └── wsgi.py
├── accounts/              # custom User + auth
├── depots/                # depot locations (SINGGAS, UNION, HSH KB…)
├── equipment/             # cylinders, meters (CYL 12.7 kg, etc.)
├── inventory/             # DepotInventory + CustomerSiteInventory
├── distribution/          # logistics movements (Depot ↔ Truck)
├── customers/             # customer master + rates
├── transactions/          # sales & billing
├── invoices/              # invoice records (PDF/email Phase 2)
├── audit/                 # immutable traceability logs
├── middleware/            # request logging
├── utils/                 # helpers (generate_number…)
├── manage.py
├── requirements.txt
└── .env
```

**Core Philosophy** (Architecture PDF Slide 2):  
The “Triangle of Constraints”:
- **Field-First Operation** → offline queue + client_temp_id sync
- **Transactional Integrity** → `@transaction.atomic` + row locks everywhere critical
- **Zero-Trust Traceability** → immutable audit log on every state change

The backend is **not just an API** — it is the guardian of physical inventory and financial truth.

#### 1. Setup & Dependencies

**Stack** (Slide 3):
- Python 3.12
- Django 6 (LTS)
- DRF 3.15+
- drf-spectacular (Swagger/OpenAPI)
- Simple JWT (stateless auth)
- MySQL 8 (prod) / SQLite (dev)
- WeasyPrint (PDF) + SMTP (email)
- django-cors-headers (React frontend)

**requirements.txt** (key lines from addendum):
```txt
Django==6.0.1
djangorestframework==3.16.1
djangorestframework_simplejwt==5.5.1
drf-spectacular==0.29.0
weasyprint==68.0
mysqlclient==2.2.7
python-decouple==3.8
django-cors-headers==4.9.0
# Pillow, PyJWT, etc. (full list in original addendum)
```

**Quick Setup**:
1. `pip install -r requirements.txt`
2. Create `.env` (SECRET_KEY, DB_ENGINE=mysql, EMAIL_HOST_USER, etc.)
3. `python manage.py makemigrations && migrate`
4. `python manage.py runserver`
5. Swagger UI → http://127.0.0.1:8000/api/schema/swagger-ui/

This gives you a secure, documented, versionable API (resource-oriented, /api/v1/ style per Slide 5).

#### 2. Core Configuration – settings.py

Key excerpts (from tutorial):

```python
from pathlib import Path
from decouple import config
from datetime import timedelta

BASE_DIR = Path(__file__).resolve().parent.parent

SECRET_KEY = config('SECRET_KEY')
DEBUG = config('DEBUG', default=True, cast=bool)
ALLOWED_HOSTS = ['*']

INSTALLED_APPS = [
    'django.contrib.admin', 'django.contrib.auth', 'django.contrib.contenttypes',
    'django.contrib.sessions', 'django.contrib.messages', 'django.contrib.staticfiles',
    'rest_framework', 'drf_spectacular', 'corsheaders',
    'accounts', 'depots', 'equipment', 'inventory', 'distribution',
    'customers', 'transactions', 'invoices', 'audit', 'middleware',
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

AUTH_USER_MODEL = 'accounts.User'

REST_FRAMEWORK = {
    'DEFAULT_AUTHENTICATION_CLASSES': ('rest_framework_simplejwt.authentication.JWTAuthentication',),
    'DEFAULT_SCHEMA_CLASS': 'drf_spectacular.openapi.AutoSchema',
}

SIMPLE_JWT = {
    'ACCESS_TOKEN_LIFETIME': timedelta(hours=8),
    'REFRESH_TOKEN_LIFETIME': timedelta(days=7),
}

# Database (conditional)
DB_ENGINE = config('DB_ENGINE', default='sqlite')
if DB_ENGINE == 'mysql':
    DATABASES = {'default': {
        'ENGINE': 'django.db.backends.mysql',
        'NAME': config('DB_NAME'),
        'USER': config('DB_USER'),
        'PASSWORD': config('DB_PASSWORD'),
        'HOST': config('DB_HOST', default='127.0.0.1'),
        'PORT': config('DB_PORT', default='3306'),
    }}
else:
    DATABASES = {'default': {
        'ENGINE': 'django.db.backends.sqlite3',
        'NAME': BASE_DIR / 'db.sqlite3',
    }}
```

**Why this matters**: Enforces JWT + RBAC (Slide 6), ACID integrity (MySQL), CORS for React, and logging for monitoring. Swagger auto-documents the entire API.

#### 3. Models – Data Foundation

Separated into **Logistics** (Distribution) vs **Sales** (Transaction) worlds (Slide 4).

**User** (accounts/models.py) – custom auth base

```python
from django.contrib.auth.models import AbstractUser
from django.db import models

class User(AbstractUser):
    ROLE_CHOICES = (('ADMIN', 'Admin'), ('DRIVER', 'Driver'), ('SUPERVISOR', 'Supervisor'))
    employee_id = models.CharField(max_length=50, unique=True)
    role = models.CharField(max_length=20, choices=ROLE_CHOICES)
    vehicle_no = models.CharField(max_length=20, blank=True)
```

→ Enables role-based permissions (DRIVER creates, ADMIN manages users/customers).

**Depot** (depots/models.py)

```python
class Depot(models.Model):
    name = models.CharField(max_length=100, unique=True)  # SINGGAS, UNION, HSH KB
    # location, contact, etc. optional
```

**Equipment** (equipment/models.py)

```python
class Equipment(models.Model):
    name = models.CharField(max_length=100, unique=True)  # CYL 12.7 kgs, CYL 50 kg (POL)
    category = models.CharField(max_length=50)           # Cylinder, Meter, Regulator
    default_rate = models.DecimalField(max_digits=10, decimal_places=2, default=0)
```

**Inventory** (inventory/models.py)

```python
class DepotInventory(models.Model):
    depot = models.ForeignKey('depots.Depot', on_delete=models.CASCADE)
    equipment = models.ForeignKey('equipment.Equipment', on_delete=models.CASCADE)
    quantity = models.IntegerField(default=0)

    class Meta:
        unique_together = ('depot', 'equipment')

class CustomerSiteInventory(models.Model):
    customer = models.ForeignKey('customers.Customer', on_delete=models.CASCADE)
    equipment = models.ForeignKey('equipment.Equipment', on_delete=models.CASCADE)
    quantity = models.IntegerField(default=0)

    class Meta:
        unique_together = ('customer', 'equipment')
```

**Distribution** (distribution/models.py) – internal logistics

```python
class Distribution(models.Model):
    user = models.ForeignKey('accounts.User', on_delete=models.SET_NULL, null=True)
    distribution_number = models.CharField(max_length=50, unique=True)
    status = models.CharField(max_length=20, default='Pending')  # Pending / Confirmed
    confirmed_at = models.DateTimeField(null=True, blank=True)
    created_at = models.DateTimeField(auto_now_add=True)

class DistributionItem(models.Model):
    distribution = models.ForeignKey(Distribution, on_delete=models.CASCADE, related_name='items')
    depot = models.ForeignKey('depots.Depot', on_delete=models.PROTECT)
    equipment = models.ForeignKey('equipment.Equipment', on_delete=models.PROTECT)
    quantity = models.PositiveIntegerField()
    movement_type = models.CharField(max_length=20)  # 'Collection', 'Empty Return'
```

**Customer** (customers/models.py)

```python
class Customer(models.Model):
    name = models.CharField(max_length=150)
    customer_code = models.CharField(max_length=50, unique=True)
    payment_type = models.CharField(max_length=20)  # 'Monthly', 'Cash'
    meter_rate = models.DecimalField(max_digits=10, decimal_places=2, default=0)
    # per-cylinder rates can be added as related model or JSONField
```

**Transaction** (transactions/models.py) – sales & billing

```python
class Transaction(models.Model):
    user = models.ForeignKey('accounts.User', on_delete=models.SET_NULL, null=True)
    customer = models.ForeignKey('customers.Customer', on_delete=models.PROTECT)
    transaction_number = models.CharField(max_length=50, unique=True)
    status = models.CharField(max_length=20, default='Pending')
    last_meter_reading = models.DecimalField(max_digits=12, decimal_places=2)
    current_meter_reading = models.DecimalField(max_digits=12, decimal_places=2)
    total_amount = models.DecimalField(max_digits=14, decimal_places=2)
    payment_received = models.BooleanField(default=False)
    created_at = models.DateTimeField(auto_now_add=True)

class TransactionItem(models.Model):
    transaction = models.ForeignKey(Transaction, on_delete=models.CASCADE, related_name='items')
    equipment = models.ForeignKey('equipment.Equipment', on_delete=models.PROTECT)
    quantity = models.IntegerField()
    rate = models.DecimalField(max_digits=10, decimal_places=2)
    item_type = models.CharField(max_length=30)  # 'Delivery', 'Empty Return', 'Service'
```

**Invoice** (invoices/models.py) – stub for MVP

```python
class Invoice(models.Model):
    transaction = models.OneToOneField('transactions.Transaction', on_delete=models.CASCADE)
    invoice_number = models.CharField(max_length=50, unique=True)
    status = models.CharField(max_length=30, default='generated')  # generated, printed, emailed, paid
    pdf_file = models.FileField(upload_to='invoices/', null=True, blank=True)
    created_at = models.DateTimeField(auto_now_add=True)
```

**AuditLog** (audit/models.py) – immutable black box

```python
class AuditLog(models.Model):
    user = models.ForeignKey('accounts.User', on_delete=models.SET_NULL, null=True)
    action = models.CharField(max_length=30)               # Create, Confirm, Update
    entity_type = models.CharField(max_length=50)          # Transaction, Distribution
    entity_id = models.PositiveIntegerField()
    timestamp = models.DateTimeField(auto_now_add=True)
    payload = models.JSONField(default=dict)               # snapshot before/after

    class Meta:
        ordering = ['-timestamp']
        indexes = [models.Index(fields=['entity_type', 'entity_id'])]
```

#### 4. Services – Business Logic Core

All critical operations live here — atomic, auditable, offline-aware.

**utils.py** (number generation)

```python
from datetime import datetime

def generate_number(prefix: str) -> str:
    now = datetime.now()
    return f"{prefix}-{now.year}-{now.strftime('%j')}-{now.strftime('%H%M%S')}"
```

**audit/services.py**

```python
from django.db import transaction
from .models import AuditLog

@transaction.atomic
def log_action(user, action: str, entity_type: str, entity_id: int, payload: dict):
    AuditLog.objects.create(
        user=user,
        action=action,
        entity_type=entity_type,
        entity_id=entity_id,
        payload=payload
    )
```

**distribution/services.py**

```python
from django.db import transaction
from django.db.models import F
from django.utils import timezone
from .models import Distribution, DistributionItem, DepotInventory
from audit.services import log_action
from utils import generate_number

@transaction.atomic
def create_distribution(user, depot_id: int, items: list[dict], remarks: str = None):
    dist = Distribution.objects.create(
        user=user,
        distribution_number=generate_number('DIST'),
        status='Pending',
        remarks=remarks
    )
    for item in items:
        DistributionItem.objects.create(
            distribution=dist,
            depot_id=depot_id,
            equipment_id=item['equipment_id'],
            quantity=item['quantity'],
            movement_type=item['movement_type']
        )
    log_action(user, 'Create', 'Distribution', dist.id, {'items': items})
    return dist

@transaction.atomic
def confirm_distribution(distribution_id: int, user):
    dist = Distribution.objects.select_for_update().get(id=distribution_id)
    if dist.status == 'Confirmed':
        raise ValueError("Already confirmed")
    
    dist.status = 'Confirmed'
    dist.confirmed_at = timezone.now()
    dist.save(update_fields=['status', 'confirmed_at'])
    
    for item in dist.items.all():
        inv = DepotInventory.objects.select_for_update().get(
            depot_id=item.depot_id, equipment_id=item.equipment_id
        )
        delta = item.quantity if item.movement_type == 'Empty Return' else -item.quantity
        inv.quantity = F('quantity') + delta
        inv.save(update_fields=['quantity'])
    
    log_action(user, 'Confirm', 'Distribution', dist.id, {'status': 'Confirmed'})
    return dist
```

**inventory/services.py**

```python
from django.db import transaction
from .models import CustomerSiteInventory, DepotInventory

@transaction.atomic
def update_inventory(entity: str, entity_id: int, equipment_id: int, delta: int, user):
    if entity == 'customer':
        inv, _ = CustomerSiteInventory.objects.select_for_update().get_or_create(
            customer_id=entity_id, equipment_id=equipment_id
        )
    else:  # depot
        inv, _ = DepotInventory.objects.select_for_update().get_or_create(
            depot_id=entity_id, equipment_id=equipment_id
        )
    
    inv.quantity += delta
    if inv.quantity < 0:
        raise ValueError("Negative inventory not allowed")
    
    inv.save(update_fields=['quantity'])
    log_action(user, 'Update', f'{entity.capitalize()}Inventory', inv.id, {'delta': delta})
    return inv
```

**transactions/services.py** (MVP version – PDF generation Phase 2)

```python
from django.db import transaction
from .models import Transaction, TransactionItem
from invoices.models import Invoice
from inventory.services import update_inventory
from audit.services import log_action
from utils import generate_number

@transaction.atomic
def create_customer_transaction_and_invoice(user, data: dict):
    # data: {'customer': int, 'current_meter': float, 'items': [...], 'client_temp_id': str}
    cust = Customer.objects.get(id=data['customer'])
    
    trx = Transaction.objects.create(
        user=user,
        customer=cust,
        transaction_number=generate_number('TRX'),
        status='Pending',
        last_meter_reading=cust.last_meter_reading or 0,  # assume cached field or history lookup
        current_meter_reading=data['current_meter'],
        total_amount=0
    )
    
    total = (trx.current_meter_reading - trx.last_meter_reading) * cust.meter_rate
    
    for item in data.get('items', []):
        TransactionItem.objects.create(
            transaction=trx,
            equipment_id=item['equipment_id'],
            quantity=item['quantity'],
            rate=item.get('rate', 0),
            item_type=item['type']
        )
        total += item['quantity'] * item.get('rate', 0)
        
        # direction: Delivery → + to customer site, Empty Return → - from customer site
        delta = item['quantity'] if item['type'] == 'Delivery' else -item['quantity']
        update_inventory('customer', cust.id, item['equipment_id'], delta, user)
    
    trx.total_amount = total
    trx.save(update_fields=['total_amount'])
    
    inv = Invoice.objects.create(
        transaction=trx,
        invoice_number=generate_number('INV')
    )
    
    log_action(user, 'Create', 'Transaction', trx.id, data)
    return trx, inv
```

#### 5. ViewSets – API Layer (Thin & Intent-Based)

ViewSets are deliberately **thin** — they parse input, call services, handle exceptions, and return responses. No business logic lives here (Slide 5 & Appendix E).

**distribution/views.py** (typical pattern)

```python
from rest_framework import viewsets, status
from rest_framework.decorators import action
from rest_framework.response import Response
from rest_framework.permissions import IsAuthenticated
from .models import Distribution
from .serializers import DistributionSerializer
from .services import create_distribution, confirm_distribution

class DistributionViewSet(viewsets.ModelViewSet):
    queryset = Distribution.objects.all()
    serializer_class = DistributionSerializer
    permission_classes = [IsAuthenticated]

    # Block generic mutations after creation
    def update(self, request, *args, **kwargs):
        return Response({"detail": "Use /confirm/ instead."}, status=status.HTTP_405_METHOD_NOT_ALLOWED)

    def destroy(self, request, *args, **kwargs):
        return Response({"detail": "Deletion not allowed."}, status=status.HTTP_405_METHOD_NOT_ALLOWED)

    def create(self, request, *args, **kwargs):
        try:
            dist = create_distribution(
                user=request.user,
                depot_id=request.data.get('depot'),
                items=request.data.get('items', []),
                remarks=request.data.get('remarks')
            )
            return Response(DistributionSerializer(dist).data, status=status.HTTP_201_CREATED)
        except Exception as e:
            return Response({"detail": str(e)}, status=status.HTTP_400_BAD_REQUEST)

    @action(detail=True, methods=['post'], url_path='confirm')
    def confirm(self, request, pk=None):
        try:
            dist = confirm_distribution(pk, request.user)
            return Response(DistributionSerializer(dist).data)
        except ValueError as ve:
            return Response({"detail": str(ve)}, status=status.HTTP_409_CONFLICT)
        except Exception as e:
            return Response({"detail": str(e)}, status=status.HTTP_400_BAD_REQUEST)

    def get_queryset(self):
        return Distribution.objects.filter(user=self.request.user)  # driver sees own only
```

**inventory/views.py** (custom action only)

```python
from rest_framework import viewsets
from rest_framework.decorators import action
from rest_framework.response import Response
from rest_framework.permissions import IsAuthenticated
from .serializers import CustomerSiteInventorySerializer
from .services import update_inventory

class InventoryViewSet(viewsets.GenericViewSet):
    permission_classes = [IsAuthenticated]

    @action(detail=False, methods=['post'], url_path='update')
    def adjust(self, request):
        try:
            inv = update_inventory(
                entity=request.data['entity'],
                entity_id=request.data['entity_id'],
                equipment_id=request.data['equipment_id'],
                delta=request.data['delta'],
                user=request.user
            )
            return Response(CustomerSiteInventorySerializer(inv).data)
        except Exception as e:
            return Response({"detail": str(e)}, status=status.HTTP_400_BAD_REQUEST)
```

**Router** (core/urls.py excerpt)

```python
from rest_framework.routers import DefaultRouter
from distribution.views import DistributionViewSet
from inventory.views import InventoryViewSet
# ...

router = DefaultRouter()
router.register(r'distributions', DistributionViewSet, basename='distribution')
router.register(r'inventories', InventoryViewSet, basename='inventory')
# add transactions, customers, etc.

urlpatterns = [
    path('api/', include(router.urls)),
    # JWT: /api/token/, /api/token/refresh/
]
```

#### 6. Middleware – Logging

```python
# middleware/request_logging.py
import time
import logging

logger = logging.getLogger(__name__)

class APILoggingMiddleware:
    def __init__(self, get_response):
        self.get_response = get_response

    def __call__(self, request):
        start = time.monotonic()
        response = self.get_response(request)
        duration_ms = int((time.monotonic() - start) * 1000)

        if request.path.startswith('/api/'):
            user = request.user.username if request.user.is_authenticated else "Anonymous"
            logger.info(
                "%s %s %s %dms %s",
                request.method, request.path, user, duration_ms,
                request.META.get('REMOTE_ADDR', 'unknown')
            )
        return response
```

#### 7. Summary – Key Takeaways for Your 2-Month Sprint

- **Prioritize**: Distribution → Inventory → basic Transaction (defer PDF/email to Phase 2)
- **Never mutate directly**: Always go through services
- **Atomic everywhere**: Wrap every confirm/create in `@transaction.atomic`
- **Audit obsessively**: Log every state change
- **Offline protocol**: Accept `client_temp_id`, return real ID mapping
- **ViewSets are thin**: Input validation + service call + response
- **Test ruthlessly**: Postman chains + concurrent confirm simulations


