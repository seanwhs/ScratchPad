### **1. Accounts (`accounts/admin.py`)**

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

### **2. Depots & Equipment (`depots/admin.py` & `equipment/admin.py`)**

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

### **3. Customers & Inventory (`customers/admin.py` & `inventory/admin.py`)**

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

### **4. Transactions & Invoices (`transactions/admin.py` & `invoices/admin.py`)**

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

### **5. Distribution (`distribution/admin.py`)**

This uses **Inlines** so you can manage the items directly within the distribution record.

```python
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

```

---

### **6. Audit (`audit/admin.py`)**

Audit logs should be **read-only** in the admin to prevent tampering.

```python
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

### **Why this matters**

By setting up these files, you can now use the `/admin` panel to:

* **Onboard Drivers**: Create User accounts and assign them roles.
* **Stock Management**: Manually adjust `CustomerSiteInventory` if a physical count differs from the system.
* **Audit**: Review the `payload` JSON of every transaction to troubleshoot billing errors.

**Would you like me to help you create a custom Admin Action that allows you to "Bulk Resend Emails" for selected Invoices?**
