# Step-by-Step Tutorial: Building the Backend for HSH LPG Cylinder Logistics System (MVP)

This tutorial guides you through building the Django-based backend for the HSH system, focusing on the MVP features as outlined in the architecture. We'll use Django 4.x (or latest stable in Jan 2026), Django REST Framework (DRF) for APIs, MySQL as the database, and JWT for authentication. The goal is to implement core business logic for distributions (collections and empty returns), inventory safety, unique ID generation, and audit logging.

**Prerequisites:**
- Python 3.10+ installed.
- Virtual environment tool (e.g., venv or poetry).
- MySQL server running (e.g., via Docker or local install).
- Basic familiarity with Django and REST APIs.
- Git for version control (optional but recommended).

**Estimated Time:** 4-6 hours for a basic setup, plus testing.

**Final Structure Overview:**
Your project will look like this:
```
hsh_backend/
├── manage.py
├── hsh_backend/
│   ├── __init__.py
│   ├── settings.py
│   ├── urls.py
│   └── wsgi.py
├── accounts/          # User auth
├── audit/             # Immutable logs
├── distribution/      # Core workflows
├── inventory/         # Stock tracking
├── requirements.txt
└── ... (other apps as needed)
```

---

## Step 1: Set Up the Django Project
1. Create a virtual environment:
   ```
   python -m venv hsh_env
   source hsh_env/bin/activate  # On Windows: hsh_env\Scripts\activate
   ```

2. Install core dependencies:
   ```
   pip install django django-rest-framework djangorestframework-simplejwt mysqlclient
   ```
   - `django`: Core framework.
   - `django-rest-framework`: For APIs.
   - `djangorestframework-simplejwt`: For JWT auth.
   - `mysqlclient`: MySQL adapter.

   Save to `requirements.txt`:
   ```
   pip freeze > requirements.txt
   ```

3. Start the Django project:
   ```
   django-admin startproject hsh_backend .
   ```

4. Create the apps:
   ```
   python manage.py startapp accounts
   python manage.py startapp audit
   python manage.py startapp distribution
   python manage.py startapp inventory
   ```
   (We'll add `transactions` and `reports` later if needed for MVP.)

5. Update `hsh_backend/settings.py` to include the apps and DRF/JWT:
   ```python
   INSTALLED_APPS = [
       # ... default apps ...
       'rest_framework',
       'accounts',
       'audit',
       'distribution',
       'inventory',
   ]

   REST_FRAMEWORK = {
       'DEFAULT_AUTHENTICATION_CLASSES': (
           'rest_framework_simplejwt.authentication.JWTAuthentication',
       ),
   }
   ```

6. Configure MySQL in `settings.py` (replace with your DB credentials):
   ```python
   DATABASES = {
       'default': {
           'ENGINE': 'django.db.backends.mysql',
           'NAME': 'hsh_db',
           'USER': 'your_mysql_user',
           'PASSWORD': 'your_password',
           'HOST': 'localhost',
           'PORT': '3306',
       }
   }
   ```
   Create the database in MySQL: `CREATE DATABASE hsh_db;`

---

## Step 2: Define the Models (Database Schema)
Based on the MVP schema, implement models in each app's `models.py`.

1. **inventory/models.py** (Depot, Equipment, Inventory):
   ```python
   from django.db import models

   class Depot(models.Model):
       name = models.CharField(max_length=100)
       code = models.CharField(max_length=20, unique=True)
       address = models.TextField()
       is_active = models.BooleanField(default=True)

       def __str__(self):
           return self.name

   class Equipment(models.Model):
       name = models.CharField(max_length=100)  # e.g., "CYL 12.7kg"
       sku = models.CharField(max_length=50, unique=True)
       weight_kg = models.DecimalField(max_digits=5, decimal_places=2)
       is_active = models.BooleanField(default=True)

       def __str__(self):
           return self.name

   class Inventory(models.Model):
       depot = models.ForeignKey(Depot, on_delete=models.PROTECT)
       equipment = models.ForeignKey(Equipment, on_delete=models.PROTECT)
       full_qty = models.PositiveIntegerField(default=0)
       empty_qty = models.PositiveIntegerField(default=0)
       last_updated = models.DateTimeField(auto_now=True)

       class Meta:
           unique_together = ('depot', 'equipment')

       def __str__(self):
           return f"{self.depot.name} - {self.equipment.name}"
   ```

2. **accounts/models.py** (Extend Django User):
   ```python
   from django.db import models
   from django.contrib.auth.models import AbstractUser
   from inventory.models import Depot

   class User(AbstractUser):
       employee_id = models.CharField(max_length=50, unique=True)
       role = models.CharField(max_length=50, choices=[('admin', 'Admin'), ('depot_staff', 'Depot Staff'), ('driver', 'Driver'), ('supervisor', 'Supervisor')])
       depot = models.ForeignKey(Depot, null=True, blank=True, on_delete=models.SET_NULL)

       def __str__(self):
           return self.username
   ```
   Update `settings.py`: `AUTH_USER_MODEL = 'accounts.User'`

3. **distribution/models.py** (DistributionHeader, DistributionItem):
   ```python
   from django.db import models
   from accounts.models import User
   from inventory.models import Depot, Equipment

   class DistributionHeader(models.Model):
       distribution_number = models.CharField(max_length=50, unique=True)
       user = models.ForeignKey(User, on_delete=models.PROTECT)
       created_at = models.DateTimeField(auto_now_add=True)
       total_collection = models.PositiveIntegerField(default=0)
       total_return = models.PositiveIntegerField(default=0)
       status = models.CharField(max_length=20, choices=[('draft', 'Draft'), ('confirmed', 'Confirmed'), ('synced', 'Synced')], default='draft')

       def __str__(self):
           return self.distribution_number

   class DistributionItem(models.Model):
       header = models.ForeignKey(DistributionHeader, related_name='items', on_delete=models.CASCADE)
       depot = models.ForeignKey(Depot, on_delete=models.PROTECT)
       equipment = models.ForeignKey(Equipment, on_delete=models.PROTECT)
       quantity = models.PositiveIntegerField()
       movement_type = models.CharField(max_length=20, choices=[('COLLECTION', 'Collection'), ('EMPTY_RETURN', 'Empty Return')])

       def __str__(self):
           return f"{self.header} - {self.movement_type} {self.quantity}"
   ```

4. **audit/models.py** (AuditLog):
   ```python
   from django.db import models
   from accounts.models import User

   class AuditLog(models.Model):
       user = models.ForeignKey(User, on_delete=models.PROTECT)
       action = models.CharField(max_length=100)
       entity_type = models.CharField(max_length=50)
       entity_id = models.PositiveIntegerField()
       payload = models.JSONField()  # Store details as JSON
       created_at = models.DateTimeField(auto_now_add=True)
       ip_address = models.GenericIPAddressField(null=True)

       def __str__(self):
           return f"{self.action} by {self.user} at {self.created_at}"
   ```

5. Run migrations:
   ```
   python manage.py makemigrations
   python manage.py migrate
   ```

---

## Step 3: Implement Business Logic (Services)
Create utility functions for core operations.

1. In `distribution/utils.py` (create the file):
   ```python
   from datetime import datetime

   def generate_unique_distribution_number():
       today = datetime.now().strftime('%Y%m%d')
       last = DistributionHeader.objects.filter(distribution_number__startswith=f'DIST-{today}-').order_by('-id').first()
       seq = int(last.distribution_number.split('-')[-1]) + 1 if last else 1
       return f'DIST-{today}-{seq:04d}'
   ```

2. In `distribution/services.py` (create the file):
   ```python
   from django.db import transaction
   from django.core.exceptions import ValidationError
   from .models import DistributionHeader, DistributionItem
   from inventory.models import Inventory
   from audit.models import AuditLog
   from .utils import generate_unique_distribution_number

   @transaction.atomic
   def create_distribution(user, items: list[dict]) -> DistributionHeader:
       if not items:
           raise ValidationError("Cannot create empty distribution")

       header = DistributionHeader.objects.create(
           user=user,
           distribution_number=generate_unique_distribution_number(),
           total_collection=0,
           total_return=0,
           status="confirmed"
       )

       collection_sum = 0
       return_sum = 0

       for item_data in items:
           depot_id = item_data["depot"]
           equipment_id = item_data["equipment"]
           qty = int(item_data["quantity"])
           movement = item_data["status"].upper()

           if qty <= 0:
               raise ValidationError("Quantity must be positive")

           inventory = Inventory.objects.select_for_update().get(
               depot_id=depot_id,
               equipment_id=equipment_id
           )

           item = DistributionItem.objects.create(
               header=header,
               depot_id=depot_id,
               equipment_id=equipment_id,
               quantity=qty,
               movement_type=movement
           )

           if movement == "COLLECTION":
               inventory.full_qty += qty
               collection_sum += qty
           elif movement == "EMPTY_RETURN":
               inventory.empty_qty += qty
               return_sum += qty
           else:
               raise ValidationError(f"Invalid movement type: {movement}")

           inventory.save(update_fields=["full_qty", "empty_qty", "last_updated"])

       header.total_collection = collection_sum
       header.total_return = return_sum
       header.save(update_fields=["total_collection", "total_return"])

       AuditLog.objects.create(
           user=user,
           action="CREATE_DISTRIBUTION",
           entity_type="DistributionHeader",
           entity_id=header.id,
           payload={"number": header.distribution_number, "items": len(items)},  # Simple dict; expand as needed
       )

       return header
   ```

This is the heart of the backend—ensuring atomicity and safety.

---

## Step 4: Set Up Serializers and Views (DRF APIs)
1. **distribution/serializers.py** (create the file):
   ```python
   from rest_framework import serializers
   from .models import DistributionHeader, DistributionItem

   class DistributionItemSerializer(serializers.ModelSerializer):
       class Meta:
           model = DistributionItem
           fields = ['depot', 'equipment', 'quantity', 'movement_type']

   class DistributionCreateSerializer(serializers.Serializer):
       items = DistributionItemSerializer(many=True)

       def create(self, validated_data):
           user = self.context['request'].user
           return create_distribution(user, validated_data['items'])

   class DistributionHeaderSerializer(serializers.ModelSerializer):
       items = DistributionItemSerializer(many=True, read_only=True)

       class Meta:
           model = DistributionHeader
           fields = '__all__'
   ```

2. **distribution/views.py**:
   ```python
   from rest_framework import generics, permissions
   from rest_framework.response import Response
   from .serializers import DistributionCreateSerializer, DistributionHeaderSerializer
   from .models import DistributionHeader

   class DistributionCreateView(generics.CreateAPIView):
       serializer_class = DistributionCreateSerializer
       permission_classes = [permissions.IsAuthenticated]

       def perform_create(self, serializer):
           header = serializer.save()
           return Response(DistributionHeaderSerializer(header).data)

   class DistributionListView(generics.ListAPIView):
       serializer_class = DistributionHeaderSerializer
       permission_classes = [permissions.IsAuthenticated]
       queryset = DistributionHeader.objects.all()  # Filter by user/role as needed
   ```

3. Update `hsh_backend/urls.py`:
   ```python
   from django.contrib import admin
   from django.urls import path, include
   from rest_framework_simplejwt.views import TokenObtainPairView, TokenRefreshView
   from distribution.views import DistributionCreateView, DistributionListView

   urlpatterns = [
       path('admin/', admin.site.urls),
       path('api/token/', TokenObtainPairView.as_view(), name='token_obtain_pair'),
       path('api/token/refresh/', TokenRefreshView.as_view(), name='token_refresh'),
       path('api/distributions/', DistributionCreateView.as_view(), name='distribution-create'),
       path('api/distributions/list/', DistributionListView.as_view(), name='distribution-list'),
   ]
   ```

---

## Step 5: Authentication and Access Control
1. Add to `settings.py` for JWT:
   ```python
   from datetime import timedelta

   SIMPLE_JWT = {
       'ACCESS_TOKEN_LIFETIME': timedelta(minutes=60),
       'REFRESH_TOKEN_LIFETIME': timedelta(days=1),
   }
   ```

2. Create a superuser:
   ```
   python manage.py createsuperuser
   ```

3. For role-based access, add custom permissions later (e.g., in views: check `request.user.role`).

---

## Step 6: Testing and Running
1. Run the server:
   ```
   python manage.py runserver
   ```
   Test API at `http://127.0.0.1:8000/api/distributions/` (use Postman or curl with JWT token).

2. Basic test: Create a distribution via POST:
   - Get token: POST to `/api/token/` with username/password.
   - Payload example:
     ```json
     {
         "items": [
             {"depot": 1, "equipment": 1, "quantity": 10, "movement_type": "COLLECTION"},
             {"depot": 1, "equipment": 2, "quantity": 5, "movement_type": "EMPTY_RETURN"}
         ]
     }
     ```
   Verify inventory updates and audit log in DB.

3. Add seed data (optional script in `management/commands/seed.py` for depots/equipment).

4. Unit tests: In `distribution/tests.py`, test `create_distribution` for atomicity (use `TransactionTestCase`).

---

## Step 7: Polish and Next Steps
- **Error Handling:** Add try/except in services for better ValidationErrors.
- **IP Logging:** In AuditLog, capture `request.META['REMOTE_ADDR']` in views.
- **Payload Serialization:** Implement `to_simple_dict()` on models for audit payload.
- **Offline Sync:** Frontend handles queuing; backend just needs idempotency (e.g., check if distribution_number exists).
- **Expansion:** Add views for inventory queries, reports (post-MVP).
- **Deployment:** Use Gunicorn + Nginx, Dockerize for production. Consider Celery for background tasks if needed.

This backend ensures safety through atomic transactions and audits. Test thoroughly with concurrent requests to verify locking. 
