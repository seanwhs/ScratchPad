**Understanding the Design of HSH Sales System**  

This is a **very practical, field-first system** built for real LPG operations in Singapore.  
It focuses on **fast, safe data entry in the field**, **correct invoicing on the spot**, **physical + digital proof**, and **proper tracking of cylinders at depots and client sites**.

### 1. Core Philosophy & Real Users

**Primary users**  
→ Delivery drivers & field sales staff (tablet + mobile thermal printer)

**Real main goals** (in order of importance)
1. Record cylinder movements (depot ↔ truck) safely & quickly
2. Deliver to customers, read meters, perform services
3. **Generate correct invoice immediately**
4. **Print invoice on the spot** (thermal printer)
5. **Automatically email PDF invoice** to customer
6. Track exactly where every cylinder is (depot or client site)
7. Track invoice status (generated / printed / emailed / paid)

**Core design mindset**  
Mobile-first + almost no typing + mandatory human confirmation + physical & email proof + full traceability

### 2. Two Completely Different Workflows

| Workflow              | Purpose                                      | Customer involved? | Creates Invoice? | Prints?          | Emails PDF?      | Tracks cylinders? |
|-----------------------|----------------------------------------------|:------------------:|:----------------:|:----------------:|:----------------:|:-----------------:|
| **Distribution**      | Move full/empty cylinders depot ↔ truck      | **No**             | **No**           | Receipt (opt.)   | **No**           | **Yes** (depot)   |
| **Transaction**       | Sell, deliver, bill customer (usage + items) | **Yes**            | **Yes**          | **Yes** (invoice)| **Yes**          | **Yes** (client)  |

### 3. Distribution – Pure Logistics (Depot ↔ Truck)

**Main screen**
```
[ Distribution ] [ Transaction ]
User: Account 001   (always shown – from login)
```

**Very fast input table** (classic field style)

| Depots            | Equipment Name         | Quantity     | Status             |
|-------------------|------------------------|--------------|--------------------|
| SINGGAS           | CYL 9 kgs              | [number]     | Collection         |
| UNION             | CYL 12.7 kgs           |              | Empty Return       |
| HSH KB            | CYL 14 kgs             |              |                    |
|                   | CYL 50 kgs (POL)       |              |                    |
|                   | CYL 50 kgs (L)         |              |                    |

**Key UX patterns**
- **[+] Add item** → dynamic rows (most important feature!)
- Almost everything = dropdown → very fast & low error
- Only two clear types: **Collection** (full out) vs **Empty Return** (empty in)

**Mandatory safety step – Confirmation screen**

```
Distribution Confirmation
User: Account 001
Timestamp: 19 Jan 2026 08:45
Distribution #: DIST-20260119-0042

Depots     Equipment        Qty   Status
SINGGAS    CYL 12.7 kgs     10    Collection
SINGGAS    CYL 14 kgs       8     Collection
SINGGAS    CYL 14 kgs       3     Empty Return

Total:
Collection     18 cylinders
Empty Return    3 cylinders

                     [ Back ]     [ Confirmed ]
```

**After Confirmed**
→ Atomic save + inventory update  
→ Audit log  
→ Optional small receipt print  
→ **Cylinders now tracked at correct depot**

### 4. Transaction – Customer Sales, Meter Billing & Invoicing

This is the **heart of the money flow** and the **original main intention** of the system.

**Main elements**
- Select **Customer** → auto-loads payment type & rates
- **Three possible billing parts** (any combination allowed)

1. **Usage Billing** (for long-term installed customers)
   - Last Meter Reading ← auto-filled from previous record
   - Current Meter Reading ← staff enters now
   - System calculates: usage = current – last
   - Subtotal = usage × rate

2. **Cylinder Delivery** (normal sales)
   - Dynamic rows (same style as distribution)

3. **Services** (installation, delivery fee, etc.)

**After [Save] → Critical Confirmation + Invoice Preview**

```
Transaction Confirmation
User: Account 001
Customer: Hock Soon Heng Factory
Transaction #: TRX-20260119-0042
Timestamp: 19 Jan 2026 08:45

Breakdown:
• Usage Billing:   1,200 units × $0.45 = $540.00
• Cylinder Delivery: 10 × CYL 12.7kg = $285.00
• Service: Installation × 1 = $50.00
──────────────────────────────
Total Amount:           $875.00
Payment received?       ○ Yes   ○ No

Invoice Actions:
• PDF will be generated
• Will be printed now (thermal printer)
• Will be emailed automatically to customer

                [ Back ]       [ Confirmed ]
```

**After Confirmed** (most important outcomes)
1. Transaction saved
2. **Invoice PDF auto-generated**
3. **Printed immediately** on mobile printer
4. **PDF emailed automatically** to customer
5. Inventory updated (cylinders now tracked at **client site**)
6. Audit log created
7. Invoice status tracked: **generated / printed / emailed / paid**

### 5. Very Important Tracking Requirements

The system must **always know where cylinders are**:

| Location Type     | How tracked                                      | Updated when                              |
|-------------------|--------------------------------------------------|-------------------------------------------|
| **Depot**         | Inventory per depot per equipment type           | After every confirmed Distribution        |
| **Client site**   | Per customer – cylinder items delivered          | After every confirmed Transaction         |
| **In transit**    | Implicit (truck load) – optional future feature  | During active distribution (not committed)|

**Invoice tracking** (critical for business & compliance)

| Status            | When set                                   | Visible to |
|-------------------|--------------------------------------------|------------|
| Generated         | After [Confirmed]                          | All        |
| Printed           | After successful mobile print              | Sales/Admin|
| Emailed           | After successful email send                | Sales/Admin|
| Paid              | Manual mark or payment integration         | All        |

### 6. Quick Architecture Reality Check

| Layer               | Main responsibilities                                                                 |
|---------------------|---------------------------------------------------------------------------------------|
| **Frontend (React)**| Dynamic tables, confirmation dialogs, mobile print trigger, offline queue             |
| **Backend (Django)**| Atomic transactions, unique number generation, PDF creation, email sending, audit    |
| **Database**        | Distribution header+items, Transaction+Invoice, Customer last meter, Depot & Client inventory |
| **Printing**        | Web Bluetooth + ESC/POS → thermal printer in field                                    |
| **Email**           | Backend sends PDF invoice automatically after confirmation                            |

### Summary – What this system really cares about (ranked)

1. **Prevent wrong cylinder movement or wrong billing** → mandatory confirmation + clear totals
2. **Fast & safe field data entry** → dropdowns + dynamic rows everywhere
3. **Generate & deliver correct invoice immediately** → print + auto-email
4. **Always know where every cylinder is** → depot & client-site tracking
5. **Track invoice lifecycle** → generated / printed / emailed / paid
6. **Clear traceability** → unique numbers + user + timestamp + audit

This is **classic Singapore LPG field operations software** —  
very focused, very pragmatic, safety-first, money-first, and built for real drivers on the road.

Bottom line:  
The system exists to let field staff **safely move cylinders**, **correctly bill customers**, **print invoices on the spot**, **email them automatically**, and **always know where every cylinder actually is**.

