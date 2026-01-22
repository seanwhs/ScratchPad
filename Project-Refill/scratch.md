Ah — now the errors are clear. You have **three separate issues** in your Django project:

---

### 1️⃣ `inventory_type` in `InventoryAdmin`

Error:

```
<class 'inventory.admin.InventoryAdmin'>: (admin.E108) The value of 'list_display[0]' refers to 'inventory_type', which is not a callable or attribute of 'InventoryAdmin', or an attribute, method, or field on 'inventory.Inventory'.
```

✅ Explanation:

* In your `InventoryAdmin` (probably in `inventory/admin.py`), you have something like:

```python
class InventoryAdmin(admin.ModelAdmin):
    list_display = ['inventory_type', 'depot', 'customer', 'equipment', 'quantity']
```

* But your `Inventory` model **does not have a field called `inventory_type`**.
* Django is complaining because it can't find it.

✅ Fix:

Either remove it or replace it with a **computed property** that indicates the type (`depot` or `customer`):

```python
class InventoryAdmin(admin.ModelAdmin):
    list_display = ['get_inventory_type', 'depot', 'customer', 'equipment', 'quantity']

    def get_inventory_type(self, obj):
        if obj.customer:
            return "Customer"
        elif obj.depot:
            return "Depot"
        return "Unknown"

    get_inventory_type.short_description = "Inventory Type"
```

---

### 2️⃣ Equipment FK issue

Errors:

```
inventory.Inventory.equipment: (fields.E300) Field defines a relation with model 'equipments.Equipment', which is either not installed, or is abstract.
inventory.Inventory.equipment: (fields.E307) The field inventory.Inventory.equipment was declared with a lazy reference to 'equipments.equipment', but app 'equipments' isn't installed.
```

✅ Explanation:

* In your `inventory/models.py` you have:

```python
equipment = models.ForeignKey("equipments.Equipment", on_delete=models.CASCADE)
```

* But the app **name is probably `equipment`**, not `equipments`. Django model strings are `'<app_label>.<ModelName>'`.

✅ Fix:

```python
equipment = models.ForeignKey("equipment.Equipment", on_delete=models.CASCADE)
```

Also check your `INSTALLED_APPS` in `settings.py` — make sure you have:

```python
INSTALLED_APPS = [
    ...
    "equipment",
    "inventory",
    "customers",
    "depots",
    ...
]
```

---

### 3️⃣ Summary of Steps to Fix All Errors

1. **Fix `equipment` FK in `Inventory` model**:

```python
# inventory/models.py
equipment = models.ForeignKey("equipment.Equipment", on_delete=models.CASCADE)
```

2. **Fix `InventoryAdmin` to remove `inventory_type` or replace with a property**:

```python
# inventory/admin.py
class InventoryAdmin(admin.ModelAdmin):
    list_display = ['get_inventory_type', 'depot', 'customer', 'equipment', 'quantity']

    def get_inventory_type(self, obj):
        if obj.customer:
            return "Customer"
        elif obj.depot:
            return "Depot"
        return "Unknown"

    get_inventory_type.short_description = "Inventory Type"
```

3. **Check `INSTALLED_APPS`** in `settings.py` to ensure all apps exist (`equipment`, `inventory`, `customers`, `depots`).

4. **Run migrations**:

```bash
python manage.py makemigrations
python manage.py migrate
```

5. Then run the server:

```bash
python manage.py runserver
```

---

If you want, I can write the **full corrected `inventory/models.py` + `admin.py` + `serializers.py`** so that your **CRUD + Postman test plan works** immediately without errors.

Do you want me to do that?
