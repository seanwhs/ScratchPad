# HSH LPG Cylinder Logistics System  
**MVP Technical Architecture & Core Workflow**  
Mobile-first • Safety-critical • Offline-capable  
**Target MVP Launch: January 2026**

A field-oriented application designed to eliminate inventory mismatches in LPG cylinder distribution operations between depots and trucks.

---

### 1. System Architecture Overview

**High-level layers (Decoupled & Mobile-first):**

```
Mobile App (React + Vite + Tailwind)  
      ↓↑   (REST API + JWT)
Django Backend + Django REST Framework  
      ↓↑   (Atomic transactions & business rules)
MySQL Database  
      ↳ Immutable Audit Logging
```

**Offline support:** IndexedDB-based queue with automatic background sync  
**Printing:** Thermal receipt via Web Bluetooth + ESC/POS protocol

---

### 2. Database Design (Core MVP Schema)

Here are the most important tables for the MVP phase with their key fields and relationships.

| Table                  | Purpose                              | Key Fields                                                                 | Relationships / Notes                              |
|------------------------|--------------------------------------|----------------------------------------------------------------------------|----------------------------------------------------|
| `Depot`                | Physical locations                   | `id`, `name`, `code`, `address`, `is_active`                               | Master data                                        |
| `Equipment`            | Cylinder types                       | `id`, `name`, `sku`, `weight_kg` (e.g. 12.7, 14), `is_active`              | Master data – different cylinder sizes/types       |
| `Inventory`            | Current stock per depot & type       | `id`, `depot_id` (FK), `equipment_id` (FK), `full_qty`, `empty_qty`, `last_updated` | **Critical** – `SELECT FOR UPDATE` used heavily    |
| `User`                 | System users (extended Django User)  | `id`, `employee_id`, `name`, `role`, `depot_id` (nullable)                 | Roles: admin, depot_staff, driver, supervisor     |
| `DistributionHeader`   | Main distribution record             | `id`, `distribution_number` (unique), `user_id` (FK), `created_at`, `total_collection`, `total_return`, `status` ('draft','confirmed','synced') | Generated unique number e.g. `DIST-20260119-0007`  |
| `DistributionItem`     | Line items of a distribution         | `id`, `header_id` (FK), `depot_id` (FK), `equipment_id` (FK), `quantity`, `movement_type` ('COLLECTION' / 'EMPTY_RETURN') | One header → many items                            |
| `AuditLog`             | Immutable history of changes         | `id`, `user_id` (FK), `action`, `entity_type`, `entity_id`, `payload` (JSON), `created_at`, `ip_address` | Never updated/deleted – append only                |

**Important relationships (simplified ER view):**

Here are some representative ER diagrams showing typical inventory + transaction structures (adapted for LPG cylinder context):








(These illustrate classic inventory + transaction header/detail patterns — very close to our needs.)

**Key invariants to enforce:**

- `full_qty` + `empty_qty` ≥ 0 at all times
- Sum of all COLLECTION → increases `full_qty`
- Sum of all EMPTY_RETURN → increases `empty_qty`
- Every write goes through atomic transaction + audit

---

### 3. Core Business Workflow – Distribution (Collection & Empty Return)

**Movement types:**

- **Collection** — Full cylinders: Depot → Truck  
- **Empty Return** — Empty cylinders: Truck → Depot

**Step-by-step flow:**

1. Driver/Staff opens **Distribution** screen  
2. Selects multiple lines: Depot + Cylinder Type + Quantity + Movement Type  
3. Sees running totals while entering  
4. Presses **Save** → shows **Confirmation Screen** (safety checkpoint)  
5. Reviews summary → enters PIN/pattern (optional future) → **Confirm**  
6. If online: → POST to backend → atomic commit → print receipt  
7. If offline: → queue locally → show "Queued" toast → auto-sync later

**Visual process reference** (LPG-related flow example):




(While not exact, this illustrates physical movement patterns we are digitizing.)

---

### 4. Backend Code Walkthrough – Critical Piece: Atomic Distribution Creation

File: `distribution/services.py`

```python
from django.db import transaction
from django.core.exceptions import ValidationError
from .models import DistributionHeader, DistributionItem, Inventory, AuditLog
from .utils import generate_unique_distribution_number


@transaction.atomic
def create_distribution(user, items: list[dict]) -> DistributionHeader:
    """
    Critical business function - MUST be atomic
    
    items = [
        {"depot": 1, "equipment": 3, "quantity": 10, "status": "Collection"},
        {"depot": 1, "equipment": 4, "quantity": 5,  "status": "EMPTY_RETURN"},
        ...
    ]
    """
    if not items:
        raise ValidationError("Cannot create empty distribution")

    # 1. Create header first
    header = DistributionHeader.objects.create(
        user=user,
        distribution_number=generate_unique_distribution_number(),
        total_collection=0,
        total_return=0,
        status="confirmed"
    )

    collection_sum = 0
    return_sum = 0

    # 2. Process each line item + update inventory atomically
    for item_data in items:
        depot_id = item_data["depot"]
        equipment_id = item_data["equipment"]
        qty = int(item_data["quantity"])
        movement = item_data["status"].upper()

        if qty <= 0:
            raise ValidationError("Quantity must be positive")

        # Lock the inventory row (prevents race conditions)
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

    # 3. Finalize header totals
    header.total_collection = collection_sum
    header.total_return = return_sum
    header.save(update_fields=["total_collection", "total_return"])

    # 4. Always audit (immutable)
    AuditLog.objects.create(
        user=user,
        action="CREATE_DISTRIBUTION",
        entity_type="DistributionHeader",
        entity_id=header.id,
        payload=header.to_simple_dict(),  # JSON friendly
    )

    return header
```

**Why this is safety-critical:**

- `select_for_update()` → pessimistic locking  
- Whole operation inside `@transaction.atomic` → all or nothing  
- Audit log is created even if something fails later (via rollback handling if needed)

---

### 5. Frontend – Key Safety & UX Components

**Most important safety component:** `ConfirmationDialog.jsx`

Shows clear summary before final commit:

```jsx
// ConfirmationDialog.jsx (simplified)
function ConfirmationDialog({ items, totals, onBack, onConfirm }) {
  return (
    <div className="fixed inset-0 bg-black/60 flex items-center justify-center p-4">
      <div className="bg-white rounded-xl max-w-md w-full p-6 shadow-2xl">
        <h2 className="text-xl font-bold mb-4">Confirm Distribution</h2>
        
        <div className="space-y-3 mb-6">
          {items.map((item, i) => (
            <div key={i} className="flex justify-between text-sm">
              <span>{item.depotName} • {item.equipmentName}</span>
              <span className={item.status === 'Collection' ? 'text-green-600' : 'text-amber-600'}>
                {item.quantity} × {item.status}
              </span>
            </div>
          ))}
        </div>

        <div className="border-t pt-4 mb-6">
          <div className="flex justify-between font-medium">
            <span>Collection:</span>
            <span className="text-green-700">{totals.collection} cylinders</span>
          </div>
          <div className="flex justify-between font-medium">
            <span>Empty Return:</span>
            <span className="text-amber-700">{totals.return} cylinders</span>
          </div>
        </div>

        <div className="flex gap-3">
          <button onClick={onBack} className="flex-1 py-3 border rounded-lg">← Back</button>
          <button 
            onClick={onConfirm}
            className="flex-1 py-3 bg-green-600 text-white rounded-lg font-medium"
          >
            CONFIRM →
          </button>
        </div>
      </div>
    </div>
  );
}
```

This screen is **the single most important UX safety net** in the entire application.

---

### 6. MVP Implementation Priority (Recommended Order)

| # | Component / Feature                  | Why Critical                              | Est. Complexity |
|---|--------------------------------------|--------------------------------------------|-----------------|
| 1 | Atomic backend create_distribution   | Prevents financial/inventory disasters     | ★★★★★          |
| 2 | Mandatory Confirmation Dialog        | Catches human input mistakes               | ★★★★☆          |
| 3 | Offline Queue + Auto-sync            | Field reliability in poor coverage areas   | ★★★★☆          |
| 4 | Unique number generation + Audit     | Traceability & dispute resolution          | ★★★☆☆          |
| 5 | Thermal receipt printing             | Physical proof for handovers               | ★★★☆☆          |
| 6 | Running totals + quick-add features  | Operator speed & reduced errors            | ★★☆☆☆          |

**Core Promise (repeat after every standup):**

> "No transaction lost.  
> No inventory mismatch.  
> Every movement confirmed by human eyes.  
> Everything traceable — even offline."

Focus relentlessly on **safety + atomicity + confirmation** first. Everything else is polish.
