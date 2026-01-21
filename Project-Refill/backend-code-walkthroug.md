### Full Code Walkthrough: Understanding Your HSH LPG Backend

Hey Sean, since you're in Singapore and it's early morning on January 21, 2026 (right around the project's start date from your docs), I'll make this walkthrough as practical as possible for your local development setup. Based on the provided documents—including the full "Proposal HSH.pdf" (which outlines UI flows and requirements) and "HSH_Sales_System_Backend_Architecture.pdf" (a 15-page slide deck on architecture)—your backend is a Django 6 API for managing LPG sales and logistics. It's field-first, with emphasis on offline sync, atomic transactions, and audit traceability.

This walkthrough synthesizes everything: the high-level architecture from the PDF slides, code snippets from the tutorial and walkthrough docs, and how it aligns with the proposal's requirements (e.g., distribution/collection workflows, transaction billing). I'll break it down file-by-file and component-by-component, explaining **what it does**, **why it's there**, and **how it works**. Assume a standard Django project structure (from "Backend-Tutorial.md"):

```
hsh_lpg_system/
├── core/
│   ├── settings.py
│   ├── urls.py
│   └── wsgi.py
├── accounts/  # User models and auth
├── depots/    # Depot models
├── equipment/ # Equipment (cylinders) models
├── inventory/ # Inventory management
├── distribution/ # Logistics workflows
├── customers/ # Customer data
├── transactions/ # Sales workflows
├── invoices/  # Invoicing and outputs
├── audit/     # Traceability logs
├── middleware/ # Custom logging
├── manage.py
├── requirements.txt
└── .env
```

Key philosophy (from Slide 2 of architecture PDF): The "Triangle of Constraints" — Field-First Operation (offline queue), Transactional Integrity (atomic updates), Zero-Trust Traceability (audit logs). The backend is the "guardian of physical inventory and financial truth," ensuring digital records match physical assets like cylinders.

#### 1. Project Setup and Dependencies
From Slide 3 (technical stack) and "Backend-Tutorial.md":

- **Runtime**: Python 3.12 + Django 6 (LTS for stability).
- **API**: DRF 3.15+ for REST endpoints, drf-spectacular for Swagger docs.
- **Auth**: Simple JWT for stateless tokens.
- **Data**: MySQL 8 (prod, ACID-compliant) / SQLite (dev).
- **Outputs**: WeasyPrint (PDF from HTML templates), SMTP (email).
- **Other**: django-cors-headers for frontend integration.

**requirements.txt** (from addendum in tutorial):
```
Django==6.0.1
djangorestframework==3.16.1
djangorestframework_simplejwt==5.5.1
drf-spectacular==0.29.0
weasyprint==68.0
mysqlclient==2.2.7
python-decouple==3.8
django-cors-headers==4.9.0
# ... (full list as in docs, including Pillow for images, PyJWT, etc.)
```

**Setup Steps** (from Slide 14 quick-start):
1. `pip install -r requirements.txt`
2. Configure `.env` (SECRET_KEY, DB_ENGINE=mysql, EMAIL_HOST_USER, etc.).
3. `python manage.py migrate`
4. `python manage.py runserver`
5. Access Swagger: http://127.0.0.1:8000/api/schema/swagger-ui/

**Why?** This ensures a secure, scalable API (Slide 5: resource-oriented, versioned /api/v1/, filtering params like ?customer=3).

#### 2. Core Configuration (core/settings.py)
From "Backend-Tutorial.md" (truncated but key parts shown):

This file sets up the app, with layers from Slide 3: Output Engine (WeasyPrint/SMTP), API Interface (DRF), Core Logic (Django/Python), Auth (JWT), Data Layer (MySQL).

Key sections:
- **Installed Apps**: Includes 'rest_framework', 'drf_spectacular', 'corsheaders', and custom apps (accounts, depots, etc.).
- **Middleware**: Adds 'corsheaders.middleware.CorsMiddleware' for React frontend, and custom 'middleware.request_logging.APILoggingMiddleware' for API logging.
- **REST Framework**: Defaults to JWT auth, spectacular for schema.
- **JWT Settings**: Access token 8 hours, refresh 7 days (stateless for mobile, as in Slide 6 auth diagram).
- **Databases**: Conditional MySQL/SQLite based on .env.
- **Auth User Model**: 'accounts.User' (custom with roles).

```python
# Excerpt
INSTALLED_APPS = [
    'django.contrib.admin',
    'rest_framework',
    'drf_spectacular',
    'corsheaders',
    'accounts', 'depots', 'equipment', 'inventory', 'distribution', 'customers', 'transactions', 'invoices', 'audit', 'middleware',
]

REST_FRAMEWORK = {
    'DEFAULT_AUTHENTICATION_CLASSES': ('rest_framework_simplejwt.authentication.JWTAuthentication',),
    'DEFAULT_SCHEMA_CLASS': 'drf_spectacular.openapi.AutoSchema',
}

SIMPLE_JWT = {
    'ACCESS_TOKEN_LIFETIME': timedelta(hours=8),
    'REFRESH_TOKEN_LIFETIME': timedelta(days=7),
}

# Database conditional (MySQL for prod)
if DB_ENGINE == 'mysql':
    DATABASES = { 'default': { 'ENGINE': 'django.db.backends.mysql', ... } }
else:
    DATABASES = { 'default': { 'ENGINE': 'django.db.backends.sqlite3', 'NAME': BASE_DIR / 'db.sqlite3' } }
```

**Understanding**: This enforces security (JWT + RBAC from Slide 6) and integrity (ACID DB). CORS allows React frontend to connect.

#### 3. Models (Data Foundation)
From Slide 4 ("Two Worlds": Distribution vs Transaction) and "backend-code-walkthrough.md".

Models are separated into logistics (no invoices) and sales (triggers PDF). Equipment models like CYL 9kg, 50kg (from proposal).

Key models:
- **User (accounts/models.py)**: Custom AbstractUser with roles ('ADMIN', 'DRIVER', 'SUPERVISOR'), employee_id, vehicle_no (from proposal User Tab).
```python
class User(AbstractUser):
    ROLE_CHOICES = (('ADMIN', 'Admin'), ('DRIVER', 'Driver'), ('SUPERVISOR', 'Supervisor'))
    employee_id = models.CharField(max_length=50, unique=True)
    role = models.CharField(max_length=20, choices=ROLE_CHOICES)
    vehicle_no = models.CharField(max_length=20, blank=True)
```

- **Depot (depots/models.py)**: e.g., SINGGAS, UNION, HSH KB (from proposal).
```python
class Depot(models.Model):
    name = models.CharField(max_length=100)
    # ... location, etc.
```

- **Equipment (equipment/models.py)**: Cylinders like CYL 12.7 kgs.
```python
class Equipment(models.Model):
    name = models.CharField(max_length=100)  # e.g., 'CYL 12.7 kgs'
    type = models.CharField(max_length=50)  # 'Cylinder', 'Meter', etc.
    rate = models.DecimalField(max_digits=10, decimal_places=2)  # For billing
```

- **Inventory (inventory/models.py)**: DepotInventory and CustomerSiteInventory (Slide 4).
```python
class DepotInventory(models.Model):
    depot = models.ForeignKey(Depot, on_delete=models.CASCADE)
    equipment = models.ForeignKey(Equipment, on_delete=models.CASCADE)
    quantity = models.IntegerField(default=0)

class CustomerSiteInventory(models.Model):
    customer = models.ForeignKey('customers.Customer', on_delete=models.CASCADE)
    equipment = models.ForeignKey(Equipment, on_delete=models.CASCADE)
    quantity = models.IntegerField(default=0)
```

- **Distribution (distribution/models.py)**: Logistics (Depot ↔ Truck, no customer). Header + Items (multi-row from proposal).
```python
class Distribution(models.Model):
    user = models.ForeignKey(User, on_delete=models.SET_NULL, null=True)
    distribution_number = models.CharField(max_length=50, unique=True)
    status = models.CharField(max_length=20, default='Pending')  # Pending/Confirmed
    confirmed_at = models.DateTimeField(null=True)

class DistributionItem(models.Model):
    distribution = models.ForeignKey(Distribution, on_delete=models.CASCADE)
    depot = models.ForeignKey(Depot, on_delete=models.PROTECT)
    equipment = models.ForeignKey(Equipment, on_delete=models.PROTECT)
    quantity = models.IntegerField()
    movement_type = models.CharField(max_length=20)  # 'Collection', 'Empty Return'
```

- **Customer (customers/models.py)**: From proposal Customer Tab, with payment_type, rates.
```python
class Customer(models.Model):
    name = models.CharField(max_length=100)
    customer_id = models.CharField(max_length=50, unique=True)
    payment_type = models.CharField(max_length=20)  # 'Monthly', 'Cash'
    meter_rate = models.DecimalField(max_digits=10, decimal_places=2)
    # Rates for each equipment type (or use related model)
```

- **Transaction (transactions/models.py)**: Sales, with meter, cylinders, services (from proposal Transaction Tab).
```python
class Transaction(models.Model):
    user = models.ForeignKey(User, on_delete=models.SET_NULL, null=True)
    customer = models.ForeignKey(Customer, on_delete=models.PROTECT)
    transaction_number = models.CharField(max_length=50, unique=True)
    status = models.CharField(max_length=20, default='Pending')
    last_meter_reading = models.DecimalField(max_digits=10, decimal_places=2)
    current_meter_reading = models.DecimalField(max_digits=10, decimal_places=2)
    total_amount = models.DecimalField(max_digits=12, decimal_places=2)
    payment_received = models.BooleanField(default=False)

class TransactionItem(models.Model):
    transaction = models.ForeignKey(Transaction, on_delete=models.CASCADE)
    equipment = models.ForeignKey(Equipment, on_delete=models.PROTECT)
    quantity = models.IntegerField()
    rate = models.DecimalField(max_digits=10, decimal_places=2)
    type = models.CharField(max_length=20)  # 'Delivery', 'Empty Return', 'Service'
```

- **Invoice (invoices/models.py)**: Linked to Transaction, with lifecycle (generated, printed, emailed, paid from Slide 11).
```python
class Invoice(models.Model):
    transaction = models.OneToOneField(Transaction, on_delete=models.CASCADE)
    invoice_number = models.CharField(max_length=50, unique=True)
    status = models.CharField(max_length=20, default='generated')  # generated, printed, emailed, paid
    pdf_file = models.FileField(upload_to='invoices/', null=True)
```

- **AuditLog (audit/models.py)**: Immutable "black box" (Slide 12).
```python
class AuditLog(models.Model):
    user = models.ForeignKey(User, on_delete=models.SET_NULL, null=True)
    action = models.CharField(max_length=20)  # 'Create', 'Confirm', 'Update'
    entity_type = models.CharField(max_length=50)  # 'Transaction', 'Distribution'
    entity_id = models.IntegerField()
    timestamp = models.DateTimeField(auto_now_add=True)
    payload = models.JSONField()  # Snapshot {before: ..., after: ...}

    class Meta:
        ordering = ['-timestamp']
```

**Understanding**: Models enforce "Two Worlds" (Slide 4): Distributions update DepotInventory (internal), Transactions update CustomerSiteInventory + MeterReading + Invoice (customer-facing). No manual overrides—locations derived from confirmed actions (Slide 7/8 workflows).

#### 4. Services (Business Logic Layer)
From "backend-code-walkthrough.md" and Slides 7-10: Services are the "authoritative core" — all mutations happen here, wrapped in @transaction.atomic.

- **audit/services.py**: Logs every action (Slide 12).
```python
from django.db import transaction
from .models import AuditLog

@transaction.atomic
def log_action(user, action, entity_type, entity_id, payload):
    AuditLog.objects.create(
        user=user,
        action=action,
        entity_type=entity_type,
        entity_id=entity_id,
        payload=payload
    )
```

- **distribution/services.py**: For logistics (Slide 7).
```python
from django.db import transaction
from django.db.models import F
from .models import Distribution, DistributionItem, DepotInventory
from audit.services import log_action
from utils import generate_number

@transaction.atomic
def create_distribution(user, depot, items, remarks=None):
    distribution = Distribution.objects.create(
        user=user,
        distribution_number=generate_number('DIST'),
        status='Pending'
    )
    for item in items:
        DistributionItem.objects.create(
            distribution=distribution,
            depot=depot,
            equipment_id=item['equipment_id'],
            quantity=item['quantity'],
            movement_type=item['movement_type']
        )
    log_action(user, 'Create', 'Distribution', distribution.id, {'items': items})
    return distribution

@transaction.atomic
def confirm_distribution(distribution_id, user):
    distribution = Distribution.objects.select_for_update().get(id=distribution_id)
    if distribution.status == 'Confirmed':
        raise ValueError('Already confirmed')
    distribution.status = 'Confirmed'
    distribution.confirmed_at = timezone.now()
    distribution.save()
    for item in distribution.distributionitem_set.all():
        inventory = DepotInventory.objects.select_for_update().get(depot=item.depot, equipment=item.equipment)
        if item.movement_type == 'Collection':
            inventory.quantity = F('quantity') - item.quantity  # Out
        elif item.movement_type == 'Empty Return':
            inventory.quantity = F('quantity') + item.quantity  # In
        inventory.save()
    log_action(user, 'Confirm', 'Distribution', distribution.id, {'status': 'Confirmed'})
    return distribution
```

- **inventory/services.py**: Updates stock (linked to workflows).
```python
@transaction.atomic
def update_inventory(entity, entity_id, equipment_id, quantity, user):
    if entity == 'customer':
        inventory, created = CustomerSiteInventory.objects.get_or_create(
            customer_id=entity_id, equipment_id=equipment_id
        )
        inventory.quantity += quantity  # Delta
        if inventory.quantity < 0:
            raise ValueError('Negative inventory not allowed')
        inventory.save()
        log_action(user, 'Update', 'Inventory', inventory.id, {'delta': quantity})
        return inventory
    # Extend for depot if needed
```

- **transactions/services.py**: For sales (Slide 8).
```python
@transaction.atomic
def create_customer_transaction_and_invoice(user, data):
    # Data: customer, current_meter, items (list of dicts), client_temp_id
    transaction = Transaction.objects.create(
        user=user,
        customer_id=data['customer'],
        transaction_number=generate_number('TRX'),
        status='Pending',
        last_meter_reading=Customer.objects.get(id=data['customer']).last_meter or 0,  # From history
        current_meter_reading=data['current_meter'],
        total_amount=0  # Calculate below
    )
    total = (transaction.current_meter_reading - transaction.last_meter_reading) * transaction.customer.meter_rate
    for item in data['items']:
        TransactionItem.objects.create(
            transaction=transaction,
            equipment_id=item['equipment_id'],
            quantity=item['quantity'],
            rate=item['rate'],
            type=item['type']
        )
        total += item['quantity'] * item['rate']
        # Update inventory
        update_inventory('customer', data['customer'], item['equipment_id'], item['quantity'] if item['type'] == 'Delivery' else -item['quantity'], user)
    transaction.total_amount = total
    transaction.save()
    # Invoice (stub for MVP; add WeasyPrint in Phase 2)
    invoice = Invoice.objects.create(
        transaction=transaction,
        invoice_number=generate_number('INV')
    )
    log_action(user, 'Create', 'Transaction', transaction.id, data)
    return transaction, invoice
```

- **utils.py**: Helpers.
```python
import datetime

def generate_number(prefix):
    return f"{prefix}-{datetime.datetime.now().year}-{datetime.datetime.now().strftime('%j')}-{datetime.datetime.now().strftime('%H%M%S')}"
```

**Understanding**: Services enforce atomicity (Slide 9) and offline protocol (Slide 10: client_temp_id reconciliation). They call log_action for traceability. Distribution is internal (no customer), Transaction triggers inventory + invoice (proposal flows).

#### 5. Views and ViewSets (API Layer)
From "backend-code-walkthrough.md" and Slide 5 (API strategy: resource-oriented, /confirm/ actions).

Views are thin—delegate to services. Use @action for custom endpoints.

- **distribution/views.py**:
```python
from rest_framework import viewsets, status
from rest_framework.decorators import action
from rest_framework.response import Response
from .serializers import DistributionSerializer
from .services import create_distribution, confirm_distribution

class DistributionViewSet(viewsets.ModelViewSet):
    serializer_class = DistributionSerializer
    queryset = Distribution.objects.all()  # Filter by user in production

    def create(self, request):
        try:
            distribution = create_distribution(request.user, request.data['depot'], request.data['items'])
            return Response(DistributionSerializer(distribution).data, status=status.HTTP_201_CREATED)
        except Exception as e:
            return Response({"detail": str(e)}, status=status.HTTP_400_BAD_REQUEST)

    @action(detail=True, methods=['post'])
    def confirm(self, request, pk=None):
        try:
            distribution = confirm_distribution(pk, request.user)
            return Response(DistributionSerializer(distribution).data)
        except Exception as e:
            return Response({"detail": str(e)}, status=status.HTTP_400_BAD_REQUEST)
```

Similar for InventoryViewSet, TransactionViewSet (with /create_transaction/), etc.

**urls.py** (core/urls.py):
```python
from django.urls import path, include
from rest_framework.routers import DefaultRouter
from distribution.views import DistributionViewSet
# ... import others

router = DefaultRouter()
router.register('distributions', DistributionViewSet)
# ... register others

urlpatterns = [
    path('api/', include(router.urls)),
    path('api/token/', TokenObtainPairView.as_view()),  # JWT login
    path('api/token/refresh/', TokenRefreshView.as_view()),
]
```

**Understanding**: Endpoints like POST /distributions/ (pending), /distributions/{id}/confirm/ (atomic update). Supports filtering (Slide 5). Permissions: IsAuthenticated + role checks.

#### 6. Middleware (middleware/request_logging.py)
From tutorial: Logs API requests for performance/audit.

```python
import logging, time

logger = logging.getLogger(__name__)

class APILoggingMiddleware:
    def __call__(self, request):
        start = time.monotonic()
        response = self.get_response(request)
        duration = (time.monotonic() - start) * 1000
        if request.path.startswith('/api/'):
            user = request.user.username or "Anonymous"
            logger.info(f"{request.method} {request.path} {user} {duration}ms {request.META.get('REMOTE_ADDR')}")
        return response
```

**Understanding**: Tracks usage (e.g., slow confirms), integrates with audit for full traceability.

#### 7. Workflows and Integration
From Slides 7-8, 11:
- **Distribution Workflow**: Input → Submission (Pending) → Confirmation (lock, update inventory, log, generate number).
- **Transaction Workflow**: Meter math + cylinders + services = total → Create record + invoice (PDF/email in Phase 2).
- **Offline**: Frontend uses temp_123 → Backend assigns TRX-999 on sync (Slide 10).
- **Audit**: Every confirm/create logs who/what/where/context (Slide 12).
- **Invoicing**: Triggered on confirm, HTML → PDF → Email/Print (Slide 11).

#### 8. Testing and Deployment
From Slide 14 and "Postman-Test-Plan.md": Use Postman for chains (login → create → confirm). Deploy with Docker + MySQL.

**Roadmap** (Slide 13): Your 4-month plan (now compressed to 2 months as discussed)—focus on MVP, defer QuickBooks/CRDTs.

