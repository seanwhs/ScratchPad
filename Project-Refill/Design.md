**Understanding the Design of HSH Sales System**  
(based on the provided specification – Distribution flow focus)

The document you shared describes a **very pragmatic, field-oriented workflow** that is typical for LPG cylinder distribution businesses in Singapore/Malaysia region.

Here is a clear, structured breakdown of the most important design aspects visible from this specification:

### 1. Overall Philosophy & Target User

Target primary user  
→ **Field sales / delivery personnel** (drivers, sales reps)

Main goal of the system  
→ Make it **fast and reliable to record cylinder movements** in the field  
(even with poor/unstable internet)

Core mindset  
→ Mobile-first, simple forms, minimal typing, quick confirmation, physical receipt printing

### 2. Main Screen – Distribution Tab (the heart of the system)

```
[ Distribution ]  [ Transaction ]
```

**Current user** is always visible  
→ "User: Account 001" (auto-filled from login)

**Three-column input table** (very characteristic of field apps):

| Depots (Dropdown)     | Equipment Name (Dropdown)     | Quantity        | Status (Dropdown)    |
|-----------------------|-------------------------------|-----------------|----------------------|
| SINGGAS               | CYL 9 kgs                     | [number input]  | Collection / Empty Return |
| UNION                 | CYL 12.7 kgs                  |                 |                      |
| HSH KB                | CYL 14 kgs                    |                 |                      |
|                       | CYL 50 kgs (POL)              |                 |                      |
|                       | CYL 50 kgs (L)                |                 |                      |

**Important UX patterns used here:**

- Add row button **[+] Add item** → dynamic table (most field apps use this pattern)
- Very few fields per row → fast to fill on mobile
- All dropdowns → minimize typing errors
- Quantity → number only + validation
- Two very distinct business actions: **Collection** vs **Empty Return**

### 3. Confirmation Screen – Very Strong Safety Pattern

After pressing **[Save]**, the system **does NOT** immediately commit.

Instead it shows:

**Distribution Confirmation** screen

```
User: Account 001
Timestamp: Date & Time
Distribution number: [auto-generated unique no]

Table summary:
Depots     Equipment      Qty    Status
SINGGAS    CYL 12.7 kgs   10     Collection
SINGGAS    CYL 14 kgs     8      Collection
SINGGAS    CYL 14 kgs     3      Empty Return

Total:
Collection: 18
Empty Return: 03

                 [ Back ]          [ Confirmed ]
```

**This is a very important design decision:**

- Two-step commit (review → confirm)
- Shows totals grouped by movement type
- Gives user chance to catch mistakes before affecting inventory
- Very common & recommended pattern in inventory/distribution systems

### 4. Visual Flow Summary (most important screens)

```
Login
   ↓
Landing page
   ↓
Distribution tab
   ↓
Fill table → [+] add rows as needed
   ↓
Press [Save]
   ↓
Confirmation popup / full screen
   • See what you are about to record
   • See totals Collection vs Return
   ↓
   [Confirmed]  →  save to server + print receipt (if online)
   or
   [Back]       →  edit again
```

### 5. Key Design Characteristics & Trade-offs

Characteristic                         | Meaning / Implication                                      | Good for field use? |
---------------------------------------|-------------------------------------------------------------|:-------------------:|
Very few free-text fields             | Almost everything is dropdown or number                     | ★★★★★              |
Dynamic row adding                    | Can record multiple cylinder types in one go                | ★★★★               |
Strong confirmation step              | Prevents accidental wrong inventory movements               | ★★★★★              |
Clear separation Collection ↔ Return  | Business-critical distinction                               | ★★★★★              |
Unique distribution number            | Easy to reference later / for paperwork                     | ★★★★               |
Totals grouped by type                | Quick sanity check (18 out vs 3 in)                         | ★★★★★              |
Offline not mentioned → probably basic| May lose data if connection dies during save               | ★★                 |
No customer selection in distribution | Distribution = depot ↔ truck movement (not yet to customer) | Makes sense        |

### 6. Quick Architecture Interpretation (from this spec)

Layer                  | Most likely responsibility
-----------------------|------------------------------------------------------
Frontend (React)       | Dynamic table, add/remove rows, confirmation dialog, print trigger
Backend (Django/DRF)   | Validate quantities, generate unique number, record movement, update inventory (hopefully atomically), create audit entry
Database               | Store distribution header + items, inventory per depot/equipment
Printing               | Browser → thermal printer (most probably Web Bluetooth + ESC/POS)

### Summary Table – What this design really cares about

Priority | Goal                                              | How it's achieved in the design
--------|---------------------------------------------------|-----------------------------------------------
1       | Prevent wrong inventory movement                  | Mandatory confirmation screen + totals
2       | Fast data entry in field                          | Dropdowns everywhere, minimal typing
3       | Clear what action is happening                    | Collection vs Empty Return very visible
4       | Traceability                                      | Unique number + timestamp + user
5       | Physical proof                                    | (Implied) print receipt after confirm

This is **classic field sales/distribution app design** — very focused, pragmatic, safety-oriented.

**Bottom line:**  
The most important thing in this design is **preventing mistakes that affect physical cylinder stock** → that's why the confirmation screen with grouped totals is the most critical UX element.

