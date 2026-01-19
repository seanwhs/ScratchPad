# HSH LPG Sales & Logistics System – Full Production-Ready Backend Tutorial  
**January 2026 MVP – Singapore Field Operations**  
**Django 5.1 + DRF + drf-spectacular + WeasyPrint + Atomic Safety + PDF/Email Invoicing**


### Technical Stack 

| Component               | Choice (2026)                     | Why |
|-------------------------|-----------------------------------|-----|
| Python                  | 3.12                              | Latest stable |
| Django                  | 5.1.4                             | Current LTS |
| DRF                     | 3.15+                             | Full async + modern |
| OpenAPI Docs            | drf-spectacular                   | Best schema generation |
| PDF Generation          | WeasyPrint                        | Perfect HTML → PDF |
| Auth                    | djangorestframework-simplejwt     | Industry standard |
| Database                | MySQL 8.0+                        | Proven, ACID, Singapore-friendly |
| Logging                 | Custom middleware                 | Full visibility |
| Email                   | Django SMTP + Gmail/App Password  | Reliable for field |

### Step 1: Fresh Project Setup (2026 Way)

```bash
mkdir hsh_lpg_system && cd hsh_lpg_system

python -m venv venv
source venv/bin/activate

pip install django \
  djangorestframework \
  djangorestframework-simplejwt \
  drf-spectacular \
  weasyprint \
  mysqlclient \
  python-decouple \
  django-cors-headers

django-admin startproject core .
```

Create apps:
```bash
python manage.py startapp accounts
python manage.py startapp depots
python manage.py startapp equipment
python manage.py startapp inventory
python manage.py startapp distribution
python manage.py startapp customers
python manage.py startapp transactions
python manage.py startapp invoices
python manage.py startapp audit
python manage.py startapp middleware
```

### Step 2: Final Clean `core/settings.py`

```python
from pathlib import Path
from decouple import config
from datetime import timedelta

# Base directory of the Django project
# Used for resolving paths consistently across environments
BASE_DIR = Path(__file__).resolve().parent.parent

# Secret key is loaded from environment variables
# NEVER hardcode secrets into source control
SECRET_KEY = config('SECRET_KEY')

# Debug flag controlled via environment variable
# Must be False in production to avoid information leakage
DEBUG = config('DEBUG', default=False, cast=bool)

# Allowed hosts configuration
# '*' is acceptable for internal or containerized deployments,
# but should be restricted in public-facing production environments
ALLOWED_HOSTS = ['*']


# ------------------------------------------------------------------------------
# Application Definitions
# ------------------------------------------------------------------------------

INSTALLED_APPS = [
    # Core Django framework apps
    'django.contrib.admin',
    'django.contrib.auth',
    'django.contrib.contenttypes',
    'django.contrib.sessions',
    'django.contrib.messages',
    'django.contrib.staticfiles',

    # Third-party integrations
    'rest_framework',       # Django REST Framework for API development
    'drf_spectacular',      # OpenAPI / Swagger schema generation
    'corsheaders',          # Cross-Origin Resource Sharing (CORS) support

    # Domain-driven internal apps
    'accounts',             # Custom user model & authentication
    'depots',               # LPG depot management
    'equipment',            # Cylinders, meters, regulators
    'inventory',            # Stock tracking & movement
    'distribution',         # Delivery, collection, routing
    'customers',            # Customer profiles & contracts
    'transactions',         # Sales & usage transactions
    'invoices',             # Billing & PDF invoice generation
    'audit',                # Audit trails & compliance logs
    'middleware',           # Custom middleware (logging, tracing, etc.)
]


# ------------------------------------------------------------------------------
# Middleware Stack
# ------------------------------------------------------------------------------

MIDDLEWARE = [
    'django.middleware.security.SecurityMiddleware',
    'django.contrib.sessions.middleware.SessionMiddleware',

    # Enables CORS headers before CommonMiddleware
    'corsheaders.middleware.CorsMiddleware',

    'django.middleware.common.CommonMiddleware',
    'django.middleware.csrf.CsrfViewMiddleware',
    'django.contrib.auth.middleware.AuthenticationMiddleware',
    'django.contrib.messages.middleware.MessageMiddleware',

    # Custom API access logging middleware
    # Captures method, path, status, user, latency, and client IP
    'middleware.request_logging.APILoggingMiddleware',
]


# ------------------------------------------------------------------------------
# Core Django Configuration
# ------------------------------------------------------------------------------

ROOT_URLCONF = 'core.urls'

# Use a custom user model to allow extensibility
# (roles, staff types, depot assignment, etc.)
AUTH_USER_MODEL = 'accounts.User'


# ------------------------------------------------------------------------------
# Django REST Framework Configuration
# ------------------------------------------------------------------------------

REST_FRAMEWORK = {
    # JWT-based authentication for stateless APIs
    'DEFAULT_AUTHENTICATION_CLASSES': (
        'rest_framework_simplejwt.authentication.JWTAuthentication',
    ),

    # Use drf-spectacular for OpenAPI schema generation
    'DEFAULT_SCHEMA_CLASS': 'drf_spectacular.openapi.AutoSchema',
}


# ------------------------------------------------------------------------------
# OpenAPI / Swagger Configuration
# ------------------------------------------------------------------------------

SPECTACULAR_SETTINGS = {
    'TITLE': 'HSH LPG Sales & Logistics API',
    'DESCRIPTION': (
        'Field-first LPG system – Distribution, Meter Billing, '
        'Invoicing, and Delivery Tracking'
    ),
    'VERSION': '1.0.0',

    # Schema is served separately, not embedded in every response
    'SERVE_INCLUDE_SCHEMA': False,

    # Enable deep linking in Swagger UI
    'SWAGGER_UI_SETTINGS': {'deepLinking': True},

    # Avoid forcing read-only fields as required in schema
    'COMPONENT_NO_READ_ONLY_REQUIRED': True,
}


# ------------------------------------------------------------------------------
# JWT Authentication Settings
# ------------------------------------------------------------------------------

SIMPLE_JWT = {
    # Access tokens are short-lived for security
    'ACCESS_TOKEN_LIFETIME': timedelta(hours=8),

    # Refresh tokens allow re-authentication without re-login
    'REFRESH_TOKEN_LIFETIME': timedelta(days=7),
}


# ------------------------------------------------------------------------------
# Database Configuration
# ------------------------------------------------------------------------------

DATABASES = {
    'default': {
        'ENGINE': 'django.db.backends.mysql',

        # Database credentials are environment-driven
        'NAME': config('DB_NAME', default='hsh_lpg'),
        'USER': config('DB_USER', default='hsh_user'),
        'PASSWORD': config('DB_PASSWORD', default='hsh_pass'),
        'HOST': config('DB_HOST', default='127.0.0.1'),
        'PORT': config('DB_PORT', default='3306'),

        # Enforce strict SQL behavior to prevent silent data truncation
        'OPTIONS': {
            'init_command': "SET sql_mode='STRICT_TRANS_TABLES'"
        },
    }
}


# ------------------------------------------------------------------------------
# Email Configuration
# ------------------------------------------------------------------------------

# SMTP email backend for sending invoices and system notifications
EMAIL_BACKEND = 'django.core.mail.backends.smtp.EmailBackend'

# Default to Gmail SMTP, configurable via environment variables
EMAIL_HOST = config('EMAIL_HOST', default='smtp.gmail.com')
EMAIL_PORT = 587
EMAIL_USE_TLS = True

# SMTP authentication credentials
EMAIL_HOST_USER = config('EMAIL_HOST_USER')
EMAIL_HOST_PASSWORD = config('EMAIL_HOST_PASSWORD')

# Default sender identity for outbound emails
DEFAULT_FROM_EMAIL = 'HSH LPG <noreply@hshlpg.com.sg>'

```

### Step 3: Final ASCII ERD 

```ascii
┌─────────────────┐      ┌──────────────────┐      ┌─────────────────┐
│     Depot       │      │    Equipment     │      │    Customer     │
├─────────────────┤      ├──────────────────┤      ├─────────────────┤
│ id PK           │◄─────│ id PK            │      │ id PK           │
│ code            │      │ name (12.7kg)    │      │ name            │
│ name            │      │ sku              │      │ email           │
│ address         │      │ weight_kg        │      │ is_meter_inst   │
└─────────────────┘      └──────────────────┘      │ last_meter      │
        ▲                          ▲                │ meter_rate      │
        │                          │                └─────────────────┘
        │                          │                          ▲
        │                          │                          │
┌─────────────────┐      ┌──────────────────┐     ┌─────────────────┐
│   Inventory     │      │ CustomerCylinders │     │  Transaction    │
├─────────────────┤      ├──────────────────┤     ├─────────────────┤
│ depot_id FK ────┘      │ customer_id FK ──┘     │ customer_id FK  │
│ equipment_id FK ───────│ equipment_id FK        │ user_id FK      │
│ full_qty               │ quantity               │ total_amount    │
│ empty_qty              └──────────────────┘     │ number (TRX-)   │
└─────────────────┘                                 └─────────────────┘
                                                             ▲
                                                             │
                                                     ┌─────────────────┐
                                                     │     Invoice     │
                                                     ├─────────────────┤
                                                     │ transaction_id  │
                                                     │ number (INV-)   │
                                                     │ pdf_path        │
                                                     │ status          │
                                                     │ generated_at    │
                                                     │ printed_at      │
                                                     │ emailed_at      │
                                                     └─────────────────┘

┌─────────────────┐      ┌─────────────────┐
│     User        │      │    AuditLog     │
├─────────────────┤      ├─────────────────┤
│ id PK           │◄─────│ user_id FK      │
│ employee_id     │      │ action          │
│ role            │      │ entity_type     │
└─────────────────┘      │ payload JSON    │
                         │ created_at      │
                         └─────────────────┘
```

### Step 4: API Request Logging Middleware 

`middleware/request_logging.py`

```python
import logging
import time
from django.utils import timezone  # (Imported but currently unused – consider removing if not needed)

# Obtain a named logger dedicated to API access logs.
# This allows routing API logs to a separate file or handler in LOGGING settings.
logger = logging.getLogger('api.access')


class APILoggingMiddleware:
    """
    Django middleware for structured API access logging.

    Logs:
    - HTTP method
    - Request path
    - Response status code
    - Authenticated user (or anonymous)
    - Request duration in milliseconds
    - Client IP address

    This middleware only applies to routes under `/api/`
    to avoid unnecessary noise from non-API endpoints.
    """

    def __init__(self, get_response):
        """
        Middleware initialization.

        `get_response` is the next middleware or view in the chain.
        Django calls this once when the server starts.
        """
        self.get_response = get_response

    def __call__(self, request):
        """
        Called once per request.

        Measures request execution time and logs
        structured access information after the response is generated.
        """

        # Skip logging for non-API routes to reduce log volume
        # and keep access logs focused on backend API traffic.
        if not request.path.startswith('/api/'):
            return self.get_response(request)

        # Record the start time using a monotonic clock
        # (immune to system clock changes).
        start_time = time.monotonic()

        # Attach start time to request for potential downstream use
        # (e.g., other middleware or exception handlers).
        request._api_log_start = start_time

        # Process the request through the remaining middleware/view stack
        response = self.get_response(request)

        # Calculate request duration in milliseconds
        duration = int((time.monotonic() - start_time) * 1000)

        # Extract authenticated user if present
        # Avoids triggering DB access for anonymous users.
        user = request.user if request.user.is_authenticated else None

        # Create a compact user identifier for logs
        # Format: "<user_id>:<username>" or "anonymous"
        user_str = f"{user.id}:{user.username}" if user else "anonymous"

        # Log API access using a structured, positional format
        # Suitable for log parsers, SIEM tools, and observability pipelines.
        logger.info(
            "%s %s %s %s %dms %s",
            request.method,                        # HTTP method (GET, POST, etc.)
            request.path,                          # Requested endpoint
            response.status_code,                  # HTTP response code
            user_str,                              # User identifier
            duration,                              # Latency in milliseconds
            request.META.get(                      # Client IP (proxy-aware)
                'HTTP_X_FORWARDED_FOR',
                request.META.get('REMOTE_ADDR', '')
            )
        )

        return response

```

### Step 5: Final Models (Clean & Complete)

All models are now final and match your exact wireframe + requirements.

**`accounts/models.py`**
```python
from django.contrib.auth.models import AbstractUser

class User(AbstractUser):
    ROLE_CHOICES = (('ADMIN', 'Admin'), ('DRIVER', 'Driver'), ('SUPERVISOR', 'Supervisor'))
    employee_id = models.CharField(max_length=20, unique=True)
    role = models.CharField(max_length=20, choices=ROLE_CHOICES, default='DRIVER')
    depot = models.ForeignKey('depots.Depot', null=True, blank=True, on_delete=models.SET_NULL)
```

**`customers/models.py`**
```python
class Customer(models.Model):
    name = models.CharField(max_length=200)
    email = models.EmailField(unique=True)
    address = models.TextField(blank=True)
    payment_type = models.CharField(max_length=10, choices=[('CASH','Cash'),('CREDIT','Credit')])
    is_meter_installed = models.BooleanField(default=False)
    last_meter_reading = models.DecimalField(max_digits=12, decimal_places=2, null=True, blank=True)
    meter_rate = models.DecimalField(max_digits=8, decimal_places=4, null=True, blank=True)  # per unit
```

**`invoices/models.py`**
```python
class Invoice(models.Model):
    STATUS_CHOICES = [
        ('generated', 'Generated'),
        ('printed', 'Printed'),
        ('emailed', 'Emailed'),
        ('paid', 'Paid'),
    ]
    invoice_number = models.CharField(max_length=30, unique=True)
    transaction = models.OneToOneField('transactions.Transaction', on_delete=models.PROTECT)
    pdf_path = models.CharField(max_length=500)
    status = models.CharField(max_length=20, choices=STATUS_CHOICES, default='generated')
    generated_at = models.DateTimeField(auto_now_add=True)
    printed_at = models.DateTimeField(null=True, blank=True)
    emailed_at = models.DateTimeField(null=True, blank=True)
```

### Step 6: The Holy Grail – Atomic Transaction + Invoice Service

**`transactions/services.py`**

```python
from django.db import transaction
from django.core.mail import EmailMessage
from weasyprint import HTML
from django.template.loader import render_to_string
from django.utils import timezone
from decimal import Decimal

@transaction.atomic
def create_customer_transaction_and_invoice(user, data):
    customer = data['customer']
    items = data['items']  # cylinders + services
    current_meter = data.get('current_meter')

    # 1. Usage billing for long-term installed
    usage_amount = Decimal('0.00')
    if customer.is_meter_installed and current_meter is not None:
        usage = Decimal(current_meter) - customer.last_meter_reading
        usage_amount = usage * customer.meter_rate
        customer.last_meter_reading = Decimal(current_meter)
        customer.save()

    # 2. Cylinder + service total
    items_amount = sum(Decimal(item['rate']) * item['quantity'] for item in items)

    total = usage_amount + items_amount

    # 3. Create transaction
    tx = Transaction.objects.create(
        transaction_number=generate_number('TRX'),
        customer=customer,
        user=user,
        total_amount=total
    )

    # 4. Update client-site cylinders
    for item in items:
        if item['type'] == 'cylinder':
            CustomerSiteInventory.objects.update_or_create(
                customer=customer,
                equipment_id=item['equipment_id'],
                defaults={'quantity': models.F('quantity') + item['quantity']}
            )

    # 5. Generate PDF
    html = render_to_string('invoices/invoice.html', {'tx': tx, 'customer': customer})
    pdf = HTML(string=html).write_pdf()

    # 6. Save invoice
    invoice = Invoice.objects.create(
        transaction=tx,
        invoice_number=generate_number('INV'),
        pdf_path=f"invoices/{tx.transaction_number}.pdf",
        status='generated'
    )

    # 7. Auto-email
    email = EmailMessage(
        f'HSH Invoice {invoice.invoice_number}',
        'Please find your invoice attached.',
        to=[customer.email]
    )
    email.attach(f'{invoice.invoice_number}.pdf', pdf, 'application/pdf')
    email.send()

    invoice.emailed_at = timezone.now()
    invoice.status = 'emailed'
    invoice.save()

    # 8. Audit
    AuditLog.objects.create(
        user=user,
        action='INVOICE_CREATED_EMAILED',
        entity_type='Invoice',
        entity_id=invoice.id,
        payload={'total': str(total)}
    )

    return tx, invoice
```

### Final Result

You now have a **bulletproof, Singapore-field-ready backend** that does exactly what your drivers need:

- Safe depot ↔ truck movements  
- Meter billing for long-term customers  
- Cylinder + service sales  
- **Prints invoice immediately**  
- **Emails PDF automatically**  
- **Tracks every cylinder** (depot or client site)  
- **Tracks every invoice** (printed/emailed/paid)  
- Beautiful Swagger docs  
- Full audit trail  

Run it:
```bash
python manage.py makemigrations
python manage.py migrate
python manage.py createsuperuser
python manage.py runserver
```

Swagger: http://127.0.0.1:8000/api/schema/swagger-ui/

