Ah — now the error has moved from Postman variables to your **backend model/admin configuration**:

```json
{
    "error": "Cannot resolve keyword 'inventory_type' into field. Choices are: customer, customer_id, depot, depot_id, equipment, equipment_id, id, last_updated, quantity"
}
```

This happens because somewhere in your **Inventory model or `InventoryAdmin`** you are referencing a field called `inventory_type`, but your `Inventory` model **doesn’t have a field by that name**.

Looking at your model:

```python
class Inventory(models.Model):
    depot = models.ForeignKey(Depot, null=True, blank=True, on_delete=models.CASCADE)
    customer = models.ForeignKey(Customer, null=True, blank=True, on_delete=models.CASCADE)
    equipment = models.ForeignKey(Equipment, on_delete=models.CASCADE)
    quantity = models.IntegerField(default=0)
    last_updated = models.DateTimeField(auto_now=True)
```

✅ Fields are: `id, depot, customer, equipment, quantity, last_updated`
❌ There is **no `inventory_type` field**.

---

### Likely causes

1. **`InventoryAdmin.list_display` references `inventory_type`**
   Check `inventory/admin.py`. You probably have something like:

```python
class InventoryAdmin(admin.ModelAdmin):
    list_display = ["inventory_type", "equipment", "quantity", "last_updated"]
```

Since `inventory_type` doesn’t exist, Django throws the error.

---

### How to fix

#### Option 1 – Add a property to `Inventory`:

```python
class Inventory(models.Model):
    # ... existing fields ...

    @property
    def inventory_type(self):
        if self.customer:
            return "customer"
        elif self.depot:
            return "depot"
        return "unknown"
```

Then `InventoryAdmin.list_display = ["inventory_type", "equipment", "quantity", "last_updated"]` works.

#### Option 2 – Remove `inventory_type` from `list_display`:

```python
class InventoryAdmin(admin.ModelAdmin):
    list_display = ["customer", "depot", "equipment", "quantity", "last_updated"]
```

---

✅ Either way, this **fixes the “Cannot resolve keyword 'inventory_type'” error** and lets your `confirm distribution` POST succeed (provided inventory quantities are sufficient).

---

If you want, I can rewrite your **InventoryAdmin + Serializer** so it fully supports `inventory_type` for both admin and API responses. This also makes Postman testing for `update-inventory` and `confirm distribution` smoother.

Do you want me to do that?
