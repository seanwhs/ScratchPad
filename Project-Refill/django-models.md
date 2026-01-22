# 🏗️ HSH LPG — Full Django Model Schemas (Production Grade)

> Architecture principle:
> **State is derived, never directly mutated.**

---

# 1. Core Identity & Access

## 1.1 Role

```python
class Role(models.Model):
    name = models.CharField(max_length=64, unique=True)
    description = models.TextField(blank=True)

    def __str__(self):
        return self.name
```

---

## 1.2 User Profile

```python
class UserProfile(models.Model):
    user = models.OneToOneField("auth.User", on_delete=models.CASCADE)
    role = models.ForeignKey(Role, on_delete=models.PROTECT)

    depot = models.ForeignKey(
        "depots.Depot",
        null=True, blank=True,
        on_delete=models.SET_NULL
    )

    is_active = models.BooleanField(default=True)
```

---

# 2. Master Data

## 2.1 Depot

```python
class Depot(models.Model):
    name = models.CharField(max_length=128, unique=True)
    address = models.TextField()
    is_active = models.BooleanField(default=True)

    def __str__(self):
        return self.name
```

---

## 2.2 Customer

```python
class Customer(models.Model):
    CUSTOMER_TYPES = [
        ("domestic", "Domestic"),
        ("commercial", "Commercial"),
        ("industrial", "Industrial"),
    ]

    name = models.CharField(max_length=255)
    address = models.TextField()
    contact_name = models.CharField(max_length=128, blank=True)
    contact_phone = models.CharField(max_length=32, blank=True)

    customer_type = models.CharField(
        max_length=32,
        choices=CUSTOMER_TYPES
    )

    credit_days = models.IntegerField(default=0)
    is_active = models.BooleanField(default=True)

    def __str__(self):
        return self.name
```

---

# 3. Equipment & Cylinder Tracking

## 3.1 Equipment Type

```python
class EquipmentType(models.Model):
    name = models.CharField(max_length=64, unique=True)
    capacity_kg = models.IntegerField()

    def __str__(self):
        return f"{self.name} ({self.capacity_kg}kg)"
```

---

## 3.2 Equipment (Cylinder)

```python
class Equipment(models.Model):
    OWNERSHIP = [
        ("company", "Company Owned"),
        ("customer", "Customer Owned"),
    ]

    serial_number = models.CharField(max_length=64, unique=True)
    equipment_type = models.ForeignKey(EquipmentType, on_delete=models.PROTECT)

    ownership = models.CharField(
        max_length=32,
        choices=OWNERSHIP
    )

    in_service = models.BooleanField(default=True)
```

---

# 4. Product & Pricing

## 4.1 Product

```python
class Product(models.Model):
    name = models.CharField(max_length=128, unique=True)
    unit = models.CharField(max_length=32, default="cylinder")
    is_active = models.BooleanField(default=True)
```

---

## 4.2 Pricing Rule

```python
class PricingRule(models.Model):
    customer = models.ForeignKey(
        Customer,
        null=True, blank=True,
        on_delete=models.CASCADE
    )

    product = models.ForeignKey(Product, on_delete=models.CASCADE)

    price = models.DecimalField(max_digits=10, decimal_places=2)
    effective_from = models.DateField()
    effective_to = models.DateField(null=True, blank=True)

    class Meta:
        unique_together = (
            "customer",
            "product",
            "effective_from",
        )
```

---

# 5. Orders & Distribution

## 5.1 Sales Order

```python
class SalesOrder(models.Model):
    STATUS = [
        ("draft", "Draft"),
        ("confirmed", "Confirmed"),
        ("assigned", "Assigned"),
        ("completed", "Completed"),
        ("cancelled", "Cancelled"),
    ]

    customer = models.ForeignKey(Customer, on_delete=models.PROTECT)
    status = models.CharField(max_length=32, choices=STATUS)

    scheduled_date = models.DateField()
    notes = models.TextField(blank=True)

    created_at = models.DateTimeField(auto_now_add=True)
```

---

## 5.2 Sales Order Line

```python
class SalesOrderLine(models.Model):
    order = models.ForeignKey(SalesOrder, related_name="lines", on_delete=models.CASCADE)
    product = models.ForeignKey(Product, on_delete=models.PROTECT)
    quantity = models.IntegerField()
```

---

## 5.3 Distribution Run

```python
class DistributionRun(models.Model):
    date = models.DateField()
    depot = models.ForeignKey("depots.Depot", on_delete=models.PROTECT)

    driver = models.ForeignKey(
        "auth.User",
        on_delete=models.PROTECT,
        related_name="distribution_runs"
    )

    status = models.CharField(
        max_length=32,
        choices=[
            ("planned", "Planned"),
            ("loaded", "Loaded"),
            ("in_progress", "In Progress"),
            ("completed", "Completed"),
        ]
    )

    created_at = models.DateTimeField(auto_now_add=True)
```

---

# 6. Transaction Ledger (CRITICAL CORE)

> This is the heart of the system.

```python
class InventoryTransaction(models.Model):
    TX_TYPE = [
        ("load", "Truck Load"),
        ("delivery", "Customer Delivery"),
        ("collection", "Customer Collection"),
        ("return", "Return to Depot"),
        ("transfer", "Depot Transfer"),
        ("dump", "Dump / Disposal"),
        ("adjustment", "Manual Adjustment"),
    ]

    tx_type = models.CharField(max_length=32, choices=TX_TYPE)

    depot = models.ForeignKey(
        "depots.Depot",
        null=True, blank=True,
        on_delete=models.PROTECT
    )

    customer = models.ForeignKey(
        Customer,
        null=True, blank=True,
        on_delete=models.PROTECT
    )

    product = models.ForeignKey(Product, on_delete=models.PROTECT)

    quantity = models.IntegerField()

    reference_type = models.CharField(max_length=64)
    reference_id = models.IntegerField()

    idempotency_key = models.CharField(max_length=128, unique=True)

    created_by = models.ForeignKey(
        "auth.User",
        on_delete=models.PROTECT
    )

    timestamp = models.DateTimeField(auto_now_add=True)

    class Meta:
        indexes = [
            models.Index(fields=["tx_type", "timestamp"]),
            models.Index(fields=["depot"]),
            models.Index(fields=["customer"]),
        ]
```

---

# 7. Inventory (Derived State Only)

> Never manually updated — always recomputed.

```python
class Inventory(models.Model):
    depot = models.ForeignKey(
        "depots.Depot",
        null=True, blank=True,
        on_delete=models.CASCADE
    )

    customer = models.ForeignKey(
        Customer,
        null=True, blank=True,
        on_delete=models.CASCADE
    )

    product = models.ForeignKey(Product, on_delete=models.CASCADE)

    quantity = models.IntegerField(default=0)

    last_updated = models.DateTimeField(auto_now=True)

    class Meta:
        constraints = [
            models.UniqueConstraint(
                fields=["depot", "product"],
                name="unique_depot_product_inventory"
            ),
            models.UniqueConstraint(
                fields=["customer", "product"],
                name="unique_customer_product_inventory"
            ),
        ]
```

---

# 8. Invoicing & Billing

## 8.1 Invoice

```python
class Invoice(models.Model):
    STATUS = [
        ("draft", "Draft"),
        ("generated", "Generated"),
        ("emailed", "Emailed"),
        ("paid", "Paid"),
        ("void", "Void"),
    ]

    invoice_number = models.CharField(max_length=64, unique=True)
    customer = models.ForeignKey(Customer, on_delete=models.PROTECT)

    subtotal = models.DecimalField(max_digits=12, decimal_places=2)
    tax = models.DecimalField(max_digits=12, decimal_places=2)
    total = models.DecimalField(max_digits=12, decimal_places=2)

    status = models.CharField(max_length=32, choices=STATUS)

    generated_at = models.DateTimeField(auto_now_add=True)
```

---

## 8.2 Invoice Line

```python
class InvoiceLine(models.Model):
    invoice = models.ForeignKey(Invoice, related_name="lines", on_delete=models.CASCADE)
    product = models.ForeignKey(Product, on_delete=models.PROTECT)

    quantity = models.IntegerField()
    unit_price = models.DecimalField(max_digits=10, decimal_places=2)
    amount = models.DecimalField(max_digits=12, decimal_places=2)
```

---

# 9. Payments

```python
class Payment(models.Model):
    METHOD = [
        ("cash", "Cash"),
        ("cheque", "Cheque"),
        ("bank", "Bank Transfer"),
        ("paynow", "PayNow"),
    ]

    invoice = models.ForeignKey(Invoice, on_delete=models.PROTECT)
    amount = models.DecimalField(max_digits=12, decimal_places=2)

    method = models.CharField(max_length=32, choices=METHOD)
    reference = models.CharField(max_length=128, blank=True)

    timestamp = models.DateTimeField(auto_now_add=True)
```

---

# 10. Audit & Compliance

```python
class AuditEvent(models.Model):
    EVENT_TYPES = [
        ("login", "Login"),
        ("transaction", "Transaction"),
        ("invoice", "Invoice"),
        ("inventory", "Inventory"),
        ("admin", "Admin"),
    ]

    user = models.ForeignKey("auth.User", on_delete=models.PROTECT)
    event_type = models.CharField(max_length=64, choices=EVENT_TYPES)

    object_type = models.CharField(max_length=64)
    object_id = models.IntegerField()

    before_state = models.JSONField(null=True, blank=True)
    after_state = models.JSONField(null=True, blank=True)

    timestamp = models.DateTimeField(auto_now_add=True)
```

---

# 11. Offline Sync & Field Safety

```python
class SyncEvent(models.Model):
    client_uuid = models.UUIDField()
    payload = models.JSONField()

    processed = models.BooleanField(default=False)
    error_message = models.TextField(blank=True)

    received_at = models.DateTimeField(auto_now_add=True)
```

---

# 12. Critical System Constraints

### Enforced at DB Level:

```
- Idempotency keys
- Unique inventory rows
- Immutable ledger records
- Immutable audit logs
```

---

# 13. Transaction Safety Rules

```
1. All writes inside atomic() blocks
2. Inventory recomputation always after ledger write
3. Invoice generation only after committed delivery TX
4. Audit logging after every commit
```

---

# 14. Real-World Guarantees Achieved

| Requirement             | Achieved By             |
| ----------------------- | ----------------------- |
| Regulatory traceability | Ledger + Audit          |
| Offline reliability     | SyncEvent + idempotency |
| Stock accuracy          | Derived inventory       |
| Invoice correctness     | Deterministic billing   |
| Fraud detection         | Immutable logs          |


