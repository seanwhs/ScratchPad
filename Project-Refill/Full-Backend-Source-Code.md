# **Full Backend Source Code**
---

### **core/settings.py**

```python
from pathlib import Path
from decouple import config
from datetime import timedelta

BASE_DIR = Path(__file__).resolve().parent.parent

SECRET_KEY = config('SECRET_KEY', default='django-insecure-secret')
DEBUG = config('DEBUG', default=True, cast=bool)

ALLOWED_HOSTS = []

INSTALLED_APPS = [
    'django.contrib.admin',
    'django.contrib.auth',
    'django.contrib.contenttypes',
    'django.contrib.sessions',
    'django.contrib.messages',
    'django.contrib.staticfiles',
    'rest_framework',
    'rest_framework_simplejwt',
    'drf_spectacular',
    'corsheaders',
    'accounts',
    'customers',
    'depots',
    'equipment',
    'inventory',
    'transactions',
    'distribution',
    'invoices',
    'audit',
]

MIDDLEWARE = [
    'django.middleware.security.SecurityMiddleware',
    'corsheaders.middleware.CorsMiddleware',
    'django.contrib.sessions.middleware.SessionMiddleware',
    'django.middleware.common.CommonMiddleware',
    'django.middleware.csrf.CsrfViewMiddleware',
    'django.contrib.auth.middleware.AuthenticationMiddleware',
    'django.contrib.messages.middleware.MessageMiddleware',
    'django.middleware.clickjacking.XFrameOptionsMiddleware',
    'middleware.request_logging.APILoggingMiddleware',
]

ROOT_URLCONF = 'core.urls'

TEMPLATES = [
    {
        'BACKEND': 'django.template.backends.django.DjangoTemplates',
        'DIRS': [BASE_DIR / 'templates'],
        'APP_DIRS': True,
        'OPTIONS': {'context_processors': [
            'django.template.context_processors.debug',
            'django.template.context_processors.request',
            'django.contrib.auth.context_processors.auth',
            'django.contrib.messages.context_processors.messages',
        ]},
    },
]

WSGI_APPLICATION = 'core.wsgi.application'

DATABASES = {
    'default': {
        'ENGINE': 'django.db.backends.sqlite3' if config('DB_ENGINE', default='sqlite') == 'sqlite' else 'django.db.backends.mysql',
        'NAME': config('DB_NAME', default='hsh_lpg'),
        'USER': config('DB_USER', default=''),
        'PASSWORD': config('DB_PASSWORD', default=''),
        'HOST': config('DB_HOST', default='127.0.0.1'),
        'PORT': config('DB_PORT', default='3306'),
    }
}

AUTH_PASSWORD_VALIDATORS = [
    {'NAME': 'django.contrib.auth.password_validation.UserAttributeSimilarityValidator'},
    {'NAME': 'django.contrib.auth.password_validation.MinimumLengthValidator'},
    {'NAME': 'django.contrib.auth.password_validation.CommonPasswordValidator'},
    {'NAME': 'django.contrib.auth.password_validation.NumericPasswordValidator'},
]

LANGUAGE_CODE = 'en-us'
TIME_ZONE = 'Asia/Singapore'
USE_I18N = True
USE_TZ = True

STATIC_URL = '/static/'
MEDIA_URL = '/media/'
MEDIA_ROOT = BASE_DIR / 'media'

DEFAULT_AUTO_FIELD = 'django.db.models.BigAutoField'

AUTH_USER_MODEL = 'accounts.User'

REST_FRAMEWORK = {
    'DEFAULT_AUTHENTICATION_CLASSES': (
        'rest_framework_simplejwt.authentication.JWTAuthentication',
    ),
    'DEFAULT_SCHEMA_CLASS': 'drf_spectacular.openapi.AutoSchema',
}

SPECTACULAR_SETTINGS = {
    'TITLE': 'HSH LPG API',
    'DESCRIPTION': 'API Documentation for HSH LPG System',
    'VERSION': '1.0.0',
}

EMAIL_BACKEND = 'django.core.mail.backends.smtp.EmailBackend'
EMAIL_HOST = 'smtp.gmail.com'
EMAIL_PORT = 587
EMAIL_USE_TLS = True
EMAIL_HOST_USER = config('EMAIL_HOST_USER')
EMAIL_HOST_PASSWORD = config('EMAIL_HOST_PASSWORD')
```

---

### **middleware/request_logging.py**

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

### **accounts/models.py**

```python
from django.contrib.auth.models import AbstractUser
from django.db import models
from depots.models import Depot

class User(AbstractUser):
    ROLES = [
        ('ADMIN', 'Admin'),
        ('SUPERVISOR', 'Supervisor'),
        ('DRIVER', 'Driver'),
    ]
    employee_id = models.CharField(max_length=20, null=True, blank=True)
    role = models.CharField(max_length=20, choices=ROLES, default='DRIVER')
    depot = models.ForeignKey(Depot, null=True, blank=True, on_delete=models.SET_NULL)
```

---

### **accounts/admin.py**

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

### **customers/models.py**

```python
from django.db import models

class Customer(models.Model):
    PAYMENT_TYPES = [('CASH','Cash'),('CREDIT','Credit')]
    name = models.CharField(max_length=200)
    email = models.EmailField()
    payment_type = models.CharField(max_length=20, choices=PAYMENT_TYPES)
    is_meter_installed = models.BooleanField(default=False)
    address = models.TextField(blank=True)
```

---

### **customers/views.py**

```python
from rest_framework import viewsets
from customers.models import Customer
from customers.serializers import CustomerSerializer

class CustomerViewSet(viewsets.ModelViewSet):
    queryset = Customer.objects.all()
    serializer_class = CustomerSerializer
```

---

### **depots/models.py**

```python
from django.db import models

class Depot(models.Model):
    code = models.CharField(max_length=20, unique=True)
    name = models.CharField(max_length=200)
```

---

### **depots/views.py**

```python
from rest_framework import viewsets
from rest_framework.permissions import IsAuthenticated
from .models import Depot
from .serializers import DepotSerializer

class DepotViewSet(viewsets.ModelViewSet):
    queryset = Depot.objects.all().order_by('code')
    serializer_class = DepotSerializer
    permission_classes = [IsAuthenticated]
```

---

### **equipment/models.py**

```python
from django.db import models

class Equipment(models.Model):
    EQUIPMENT_TYPES = [('CYLINDER','Cylinder'),('ACCESSORY','Accessory'),('METER','Meter')]
    sku = models.CharField(max_length=50, unique=True)
    name = models.CharField(max_length=200)
    equipment_type = models.CharField(max_length=50, choices=EQUIPMENT_TYPES)
    is_active = models.BooleanField(default=True)
```

---

### **equipment/views.py**

```python
from rest_framework import viewsets
from rest_framework.permissions import IsAuthenticated
from equipment.models import Equipment
from equipment.serializers import EquipmentSerializer

class EquipmentViewSet(viewsets.ModelViewSet):
    queryset = Equipment.objects.filter(is_active=True).order_by('name')
    serializer_class = EquipmentSerializer
    permission_classes = [IsAuthenticated]
```

---

### **inventory/models.py**

```python
from django.db import models
from customers.models import Customer
from equipment.models import Equipment
from depots.models import Depot

class Inventory(models.Model):
    depot = models.ForeignKey("depots.Depot", null=True, blank=True, on_delete=models.CASCADE)
    customer = models.ForeignKey("customers.Customer", null=True, blank=True, on_delete=models.CASCADE)
    equipment = models.ForeignKey("equipment.Equipment", on_delete=models.CASCADE)
    quantity = models.IntegerField(default=0)
    last_updated = models.DateTimeField(auto_now=True)

    class Meta:
        constraints = [
            models.UniqueConstraint(fields=["depot","equipment"], name="unique_depot_equipment_inventory"),
            models.UniqueConstraint(fields=["customer","equipment"], name="unique_customer_equipment_inventory"),
        ]
```

---

### **inventory/views.py**

```python
from rest_framework import viewsets, status
from rest_framework.decorators import action
from rest_framework.response import Response
from inventory.models import Inventory
from inventory.serializers import InventorySerializer
from inventory.services import InventoryService

class InventoryViewSet(viewsets.ModelViewSet):
    queryset = Inventory.objects.all().select_related("depot","customer","equipment")
    serializer_class = InventorySerializer

    @action(detail=False, methods=["post"], url_path="update-inventory", url_name="update_inventory")
    def update_inventory(self, request):
        data = request.data
        required_fields = ["entity","entity_id","equipment_id","quantity"]
        missing = [f for f in required_fields if f not in data]
        if missing:
            return Response({"error": f"Missing required fields: {', '.join(missing)}"}, status=status.HTTP_400_BAD_REQUEST)
        try:
            inventory = InventoryService.update_inventory(
                entity=data["entity"],
                entity_id=int(data["entity_id"]),
                equipment_id=int(data["equipment_id"]),
                quantity=int(data["quantity"])
            )
            serializer = self.get_serializer(inventory)
            return Response(serializer.data, status=status.HTTP_200_OK)
        except Exception as e:
            return Response({"error": str(e)}, status=status.HTTP_400_BAD_REQUEST)
```

---

### **transactions/models.py**

```python
from django.db import models
from customers.models import Customer
from accounts.models import User

class Transaction(models.Model):
    transaction_number = models.CharField(max_length=50, unique=True)
    customer = models.ForeignKey(Customer, on_delete=models.CASCADE)
    user = models.ForeignKey(User, on_delete=models.CASCADE)
    total_amount = models.DecimalField(max_digits=12, decimal_places=2)
    created_at = models.DateTimeField(auto_now_add=True)

class TransactionItem(models.Model):
    TRANSACTION_TYPE_CHOICES = [('CYLINDER','Cylinder'),('ACCESSORY','Accessory'),('METER','Meter')]
    transaction = models.ForeignKey(Transaction, related_name='items', on_delete=models.CASCADE)
    equipment = models.ForeignKey("equipment.Equipment", on_delete=models.CASCADE)
    quantity = models.IntegerField()
    rate = models.DecimalField(max_digits=12, decimal_places=2)
    type = models.CharField(max_length=50, choices=TRANSACTION_TYPE_CHOICES)
    amount = models.DecimalField(max_digits=12, decimal_places=2)
```

---

### **transactions/views.py**

```python
from rest_framework import viewsets, status, serializers
from rest_framework.decorators import action
from rest_framework.response import Response
from rest_framework.permissions import IsAuthenticated
from transactions.models import Transaction, TransactionItem
from transactions.serializers import TransactionSerializer
from invoices.serializers import InvoiceSerializer
from transactions.services import create_customer_transaction_and_invoice

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
        if user.role == 'DRIVER':
            return Transaction.objects.filter(user=user)
        elif user.role == 'SUPERVISOR' and user.depot:
            return Transaction.objects.filter(user__depot=user.depot).select_related('customer','user')
        elif user.role == 'ADMIN':
            return Transaction.objects.all().select_related('customer','user')
        return Transaction.objects.none()

    @action(detail=False, methods=['post'])
    def create_transaction(self, request):
        serializer = CreateTransactionSerializer(data=request.data)
        serializer.is_valid(raise_exception=True)
        try:
            tx, invoice = create_customer_transaction_and_invoice(user=request.user, data=serializer.validated_data)
            tx_ser = TransactionSerializer(tx, context={'request':request})
            inv_ser = InvoiceSerializer(invoice, context={'request':request})
            return Response({'transaction':tx_ser.data,'invoice':inv_ser.data}, status=status.HTTP_201_CREATED)
        except ValueError as ve:
            return Response({'error':str(ve),'code':'validation_error'}, status=status.HTTP_400_BAD_REQUEST)
        except Exception as e:
            return Response({'error':'Internal error','code':'server_error'}, status=status.HTTP_500_INTERNAL_SERVER_ERROR)
```

---
Perfect! Here’s the **rest of the HSH LPG backend source code**, continuing from where we left off.

---

### **invoices/models.py**

```python
from django.db import models
from transactions.models import Transaction

class Invoice(models.Model):
    STATUS_CHOICES = [('generated','Generated'),('emailed','Emailed'),('printed','Printed')]
    invoice_number = models.CharField(max_length=50, unique=True)
    transaction = models.OneToOneField(Transaction, on_delete=models.CASCADE)
    pdf_path = models.CharField(max_length=500, blank=True)
    status = models.CharField(max_length=20, choices=STATUS_CHOICES, default='generated')
    generated_at = models.DateTimeField(auto_now_add=True)
    printed_at = models.DateTimeField(null=True, blank=True)
    emailed_at = models.DateTimeField(null=True, blank=True)
```

---

### **invoices/services.py**

```python
from django.conf import settings
from django.core.files.base import ContentFile
from django.core.files.storage import default_storage
from weasyprint import HTML
import os

def generate_invoice_pdf(transaction, html_string):
    pdf_folder = f"invoices/{transaction.created_at:%Y%m}/"
    full_folder_path = os.path.join(settings.MEDIA_ROOT, pdf_folder)
    os.makedirs(full_folder_path, exist_ok=True)
    pdf_filename = f"{transaction.transaction_number}.pdf"
    pdf_path = os.path.join(pdf_folder, pdf_filename)
    pdf_bytes = HTML(string=html_string).write_pdf()
    saved_path = default_storage.save(pdf_path, ContentFile(pdf_bytes))
    return saved_path
```

---

### **invoices/views.py**

```python
import os
from django.core.files.storage import default_storage
from django.http import FileResponse
from rest_framework.decorators import action
from rest_framework.response import Response
from rest_framework import viewsets, status
from django.utils import timezone
from django.core.mail import EmailMessage
from invoices.models import Invoice
from invoices.serializers import InvoiceSerializer
from audit.models import AuditLog

class InvoiceViewSet(viewsets.ModelViewSet):
    queryset = Invoice.objects.all()
    serializer_class = InvoiceSerializer

    @action(detail=True, methods=['get'])
    def pdf(self, request, pk=None):
        invoice = self.get_object()
        if not invoice.pdf_path:
            return Response({'error':'PDF not generated'}, status=status.HTTP_404_NOT_FOUND)
        if not default_storage.exists(invoice.pdf_path):
            return Response({'error':'PDF not found'}, status=status.HTTP_404_NOT_FOUND)
        pdf_file = default_storage.open(invoice.pdf_path, 'rb')
        return FileResponse(pdf_file, content_type='application/pdf', as_attachment=False, filename=os.path.basename(invoice.pdf_path))

    @action(detail=True, methods=['post'])
    def email(self, request, pk=None):
        invoice = self.get_object()
        if not invoice.pdf_path or not default_storage.exists(invoice.pdf_path):
            return Response({'error':'PDF not found'}, status=status.HTTP_404_NOT_FOUND)
        try:
            pdf_file_path = default_storage.path(invoice.pdf_path)
            email = EmailMessage(
                subject=f"HSH LPG Invoice {invoice.invoice_number}",
                body="Dear Customer,\n\nPlease find attached your invoice.",
                to=[invoice.transaction.customer.email]
            )
            email.attach_file(pdf_file_path)
            email.send(fail_silently=False)
            invoice.status = 'emailed'
            invoice.emailed_at = timezone.now()
            invoice.save(update_fields=['status','emailed_at'])
            AuditLog.objects.create(
                user=request.user,
                action='INVOICE_EMAILED',
                entity_type='Invoice',
                entity_id=invoice.id,
                payload={'invoice_number': invoice.invoice_number}
            )
            return Response(self.get_serializer(invoice).data)
        except Exception as e:
            return Response({'error': str(e)}, status=status.HTTP_400_BAD_REQUEST)
```

---

### **distribution/models.py**

```python
from django.db import models
from depots.models import Depot
from accounts.models import User
from equipment.models import Equipment

class Distribution(models.Model):
    distribution_number = models.CharField(max_length=50, unique=True)
    depot = models.ForeignKey(Depot, on_delete=models.CASCADE)
    user = models.ForeignKey(User, on_delete=models.CASCADE)
    confirmed_at = models.DateTimeField(null=True, blank=True)
    created_at = models.DateTimeField(auto_now_add=True)

class DistributionItem(models.Model):
    distribution = models.ForeignKey(Distribution, related_name='items', on_delete=models.CASCADE)
    equipment = models.ForeignKey(Equipment, on_delete=models.CASCADE)
    quantity = models.IntegerField()
```

---

### **distribution/views.py**

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
            dist = create_distribution(user=request.user, depot=depot, items=request.data['items'], remarks=request.data.get('remarks',''))
            return Response(self.get_serializer(dist).data, status=status.HTTP_201_CREATED)
        except Exception as e:
            return Response({'error': str(e)}, status=status.HTTP_400_BAD_REQUEST)

    @action(detail=True, methods=['post'])
    def confirm(self, request, pk=None):
        try:
            dist = confirm_distribution(pk, request.user)
            return Response(self.get_serializer(dist).data, status=status.HTTP_200_OK)
        except ValueError as ve:
            return Response({'error': str(ve)}, status=status.HTTP_400_BAD_REQUEST)
        except Exception:
            return Response({'error':'Internal server error'}, status=status.HTTP_500_INTERNAL_SERVER_ERROR)
```

---

### **audit/models.py**

```python
from django.db import models
from django.conf import settings
from django.contrib.postgres.fields import JSONField

class AuditLog(models.Model):
    timestamp = models.DateTimeField(auto_now_add=True)
    user = models.ForeignKey(settings.AUTH_USER_MODEL, null=True, on_delete=models.SET_NULL)
    action = models.CharField(max_length=200)
    entity_type = models.CharField(max_length=100)
    entity_id = models.IntegerField()
    payload = models.JSONField(null=True, blank=True)
```

---

### **audit/views.py**

```python
from rest_framework import viewsets
from audit.models import AuditLog
from audit.serializers import AuditLogSerializer

class AuditLogViewSet(viewsets.ReadOnlyModelViewSet):
    queryset = AuditLog.objects.all().order_by('-timestamp')
    serializer_class = AuditLogSerializer
```

---

### **core/urls.py**

```python
from django.contrib import admin
from django.urls import path, include
from rest_framework import routers
from rest_framework_simplejwt.views import TokenObtainPairView, TokenRefreshView
from audit.views import AuditLogViewSet
from transactions.views import TransactionViewSet
from customers.views import CustomerViewSet
from inventory.views import InventoryViewSet
from distribution.views import DistributionViewSet
from invoices.views import InvoiceViewSet
from equipment.views import EquipmentViewSet
from depots.views import DepotViewSet
from drf_spectacular.views import SpectacularAPIView, SpectacularSwaggerView

router = routers.DefaultRouter()
router.register(r'audit', AuditLogViewSet, basename='audit')
router.register(r'equipment', EquipmentViewSet, basename='equipment')
router.register(r'transactions', TransactionViewSet, basename='transaction')
router.register(r'customers', CustomerViewSet)
router.register(r'inventories', InventoryViewSet)
router.register(r'distributions', DistributionViewSet, basename='distribution')
router.register(r'invoices', InvoiceViewSet, basename='invoice')
router.register(r'depots', DepotViewSet, basename='depot')

urlpatterns = [
    path('admin/', admin.site.urls),
    path('api/token/', TokenObtainPairView.as_view(), name='token_obtain_pair'),
    path('api/token/refresh/', TokenRefreshView.as_view(), name='token_refresh'),
    path('api/schema/', SpectacularAPIView.as_view(), name='schema'),
    path('api/schema/swagger-ui/', SpectacularSwaggerView.as_view(url_name='schema'), name='swagger-ui'),
    path('api/', include(router.urls)),
]
```

---

### **templates/invoices/invoice.html**

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

    <h3>Items and Meter Usage</h3>
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
                {% if item.type == 'METER' %}
                    <td>Meter Usage Billing</td>
                    <td class="right">{{ item.quantity }}</td>
                    <td class="right">{{ item.rate|floatformat:2 }}</td>
                    <td class="right">{{ item.amount|floatformat:2 }}</td>
                {% else %}
                    <td>{{ item.equipment.name }} ({{ item.type }})</td>
                    <td class="right">{{ item.quantity }}</td>
                    <td class="right">{{ item.rate|floatformat:2 }}</td>
                    <td class="right">{{ item.amount|floatformat:2 }}</td>
                {% endif %}
            </tr>
            {% endfor %}
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

✅ This completes **the full backend source code** combining both prompts, including:

* All models, views, serializers, services, and middleware
* Admin registrations
* URL routing and JWT setup
* PDF invoice template

---

### **accounts/serializers.py**

```python
from rest_framework import serializers
from .models import User

class UserSerializer(serializers.ModelSerializer):
    class Meta:
        model = User
        fields = ['id', 'username', 'email', 'employee_id', 'role', 'depot']
```

---

### **customers/serializers.py**

```python
from rest_framework import serializers
from .models import Customer

class CustomerSerializer(serializers.ModelSerializer):
    class Meta:
        model = Customer
        fields = ['id', 'name', 'email', 'address', 'payment_type', 'is_meter_installed']
```

---

### **equipment/serializers.py**

```python
from rest_framework import serializers
from .models import Equipment

class EquipmentSerializer(serializers.ModelSerializer):
    class Meta:
        model = Equipment
        fields = ['id', 'sku', 'name', 'equipment_type', 'is_active']
```

---

### **depots/serializers.py**

```python
from rest_framework import serializers
from .models import Depot

class DepotSerializer(serializers.ModelSerializer):
    class Meta:
        model = Depot
        fields = ['id', 'code', 'name', 'address']
```

---

### **inventory/serializers.py**

```python
from rest_framework import serializers
from inventory.models import Inventory
from equipment.serializers import EquipmentSerializer
from depots.serializers import DepotSerializer
from customers.serializers import CustomerSerializer

class InventorySerializer(serializers.ModelSerializer):
    equipment = EquipmentSerializer(read_only=True)
    depot = DepotSerializer(read_only=True)
    customer = CustomerSerializer(read_only=True)

    class Meta:
        model = Inventory
        fields = ['id', 'equipment', 'depot', 'customer', 'quantity', 'last_updated']
```

---

### **transactions/serializers.py**

```python
from rest_framework import serializers
from transactions.models import Transaction, TransactionItem
from customers.serializers import CustomerSerializer

class TransactionItemSerializer(serializers.ModelSerializer):
    equipment_name = serializers.CharField(source='equipment.name', read_only=True)

    class Meta:
        model = TransactionItem
        fields = ['id', 'equipment', 'equipment_name', 'quantity', 'rate', 'amount', 'type']

class TransactionSerializer(serializers.ModelSerializer):
    customer = CustomerSerializer(read_only=True)
    items = TransactionItemSerializer(many=True, read_only=True)

    class Meta:
        model = Transaction
        fields = ['id', 'transaction_number', 'customer', 'total_amount', 'created_at', 'items']
```

---

### **invoices/serializers.py**

```python
from rest_framework import serializers
from invoices.models import Invoice
from transactions.serializers import TransactionSerializer

class InvoiceSerializer(serializers.ModelSerializer):
    transaction = TransactionSerializer(read_only=True)

    class Meta:
        model = Invoice
        fields = ['id', 'invoice_number', 'transaction', 'pdf_path', 'status', 'generated_at', 'printed_at', 'emailed_at']
```

---

### **distribution/serializers.py**

```python
from rest_framework import serializers
from distribution.models import Distribution, DistributionItem
from depots.serializers import DepotSerializer
from accounts.serializers import UserSerializer
from equipment.serializers import EquipmentSerializer

class DistributionItemSerializer(serializers.ModelSerializer):
    equipment = EquipmentSerializer(read_only=True)
    class Meta:
        model = DistributionItem
        fields = ['id', 'equipment', 'quantity']

class DistributionSerializer(serializers.ModelSerializer):
    depot = DepotSerializer(read_only=True)
    user = UserSerializer(read_only=True)
    items = DistributionItemSerializer(many=True, read_only=True)

    class Meta:
        model = Distribution
        fields = ['id', 'distribution_number', 'depot', 'user', 'confirmed_at', 'created_at', 'items']
```

---

### **audit/serializers.py**

```python
from rest_framework import serializers
from audit.models import AuditLog
from accounts.serializers import UserSerializer

class AuditLogSerializer(serializers.ModelSerializer):
    user = UserSerializer(read_only=True)
    class Meta:
        model = AuditLog
        fields = ['id', 'timestamp', 'user', 'action', 'entity_type', 'entity_id', 'payload']
```

---

### **transactions/services.py**

```python
from django.db import transaction as db_transaction
from transactions.models import Transaction, TransactionItem
from invoices.models import Invoice
from invoices.services import generate_invoice_pdf
from django.template.loader import render_to_string

def create_customer_transaction_and_invoice(user, data):
    from customers.models import Customer
    customer = Customer.objects.get(id=data['customer'])
    items_data = data['items']

    with db_transaction.atomic():
        tx = Transaction.objects.create(
            customer=customer,
            user=user,
            total_amount=0  # updated after items
        )
        total = 0
        for item in items_data:
            amount = item['quantity'] * float(item['rate'])
            total += amount
            TransactionItem.objects.create(
                transaction=tx,
                equipment_id=item['equipment'],
                quantity=item['quantity'],
                rate=item['rate'],
                amount=amount,
                type=item['type']
            )
        tx.total_amount = total
        tx.save(update_fields=['total_amount'])

        # Generate invoice
        html_string = render_to_string('invoices/invoice.html', {
            'tx': tx,
            'customer': customer,
            'items': tx.items.all(),
            'total_amount': total
        })
        pdf_path = generate_invoice_pdf(tx, html_string)
        invoice = Invoice.objects.create(
            invoice_number=f"INV-{tx.created_at:%Y%m%d}-{tx.id:06d}",
            transaction=tx,
            pdf_path=pdf_path,
        )
        return tx, invoice
```

---

### **inventory/services.py**

```python
from inventory.models import Inventory

class InventoryService:
    @staticmethod
    def update_inventory(entity, entity_id, equipment_id, quantity):
        if entity not in ['depot','customer']:
            raise ValueError("entity must be 'depot' or 'customer'")
        filters = {'equipment_id': equipment_id}
        if entity == 'depot':
            filters['depot_id'] = entity_id
        else:
            filters['customer_id'] = entity_id

        inventory, created = Inventory.objects.get_or_create(**filters)
        inventory.quantity = quantity
        inventory.save()
        return inventory
```

---

### **distribution/services.py**

```python
from distribution.models import Distribution, DistributionItem
from django.db import transaction as db_transaction

def create_distribution(user, depot, items, remarks=''):
    with db_transaction.atomic():
        dist = Distribution.objects.create(user=user, depot=depot)
        for item in items:
            DistributionItem.objects.create(
                distribution=dist,
                equipment_id=item['equipment'],
                quantity=item['quantity']
            )
        return dist

def confirm_distribution(distribution_id, user):
    from django.utils import timezone
    dist = Distribution.objects.get(id=distribution_id)
    if dist.confirmed_at:
        raise ValueError("Distribution already confirmed")
    dist.confirmed_at = timezone.now()
    dist.save(update_fields=['confirmed_at'])
    return dist
```

---




