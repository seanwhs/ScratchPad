# **Understanding the Design of HSH Sales System**

This is a **practical, field-first system** designed for **real LPG operations in Singapore**.
It focuses on:

* **Fast, safe data entry in the field**
* **Correct invoicing on the spot**
* **Physical + digital proof**
* **Precise tracking of cylinders** at depots and client sites

---

## **1️⃣ Core Philosophy & Real Users**

**Primary users:**

* Delivery drivers and field sales staff (tablet + mobile thermal printer)

**Main goals (priority order):**

1. Record cylinder movements (depot ↔ truck) **quickly and safely**
2. Deliver to customers, read meters, perform services
3. **Generate correct invoice immediately**
4. **Print invoice on the spot** (thermal printer)
5. **Automatically email PDF invoice** to customer
6. Track exactly where every cylinder is (depot or client site)
7. Track invoice status (generated / printed / emailed / paid)

**Design mindset:**

* Mobile-first, minimal typing, mandatory human confirmation
* **Physical + email proof**
* Full traceability of inventory and invoices

---

## **2️⃣ Two Completely Different Workflows**

| Workflow         | Purpose                                      | Customer involved? | Creates Invoice? |      Prints?      | Emails PDF? | Tracks cylinders? |
| ---------------- | -------------------------------------------- | :----------------: | :--------------: | :---------------: | :---------: | :---------------: |
| **Distribution** | Move full/empty cylinders depot ↔ truck      |       **No**       |      **No**      |   Receipt (opt.)  |    **No**   |  **Yes** (depot)  |
| **Transaction**  | Sell, deliver, bill customer (usage + items) |       **Yes**      |      **Yes**     | **Yes** (invoice) |   **Yes**   |  **Yes** (client) |

---

## **3️⃣ Distribution – Pure Logistics (Depot ↔ Truck)**

**Main screen:**

```
[ Distribution ] [ Transaction ]
User: Account 001   (always shown – from login)
```

**Fast input table – dropdown + number fields:**

| Depots  | Equipment Name   | Quantity | Status       |
| ------- | ---------------- | -------- | ------------ |
| SINGGAS | CYL 9 kgs        | [number] | Collection   |
| UNION   | CYL 12.7 kgs     |          | Empty Return |
| HSH KB  | CYL 14 kgs       |          |              |
|         | CYL 50 kgs (POL) |          |              |
|         | CYL 50 kgs (L)   |          |              |

**Key UX patterns:**

* **[+] Add item** → dynamic rows (critical!)
* Almost all inputs = **dropdowns** → fast + low error
* Only two clear statuses: **Collection** (full out) vs **Empty Return** (empty in)

**Mandatory confirmation screen:**

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

**After [Confirmed]:**

* Atomic save + inventory update
* Audit log
* Optional small receipt print
* **Cylinders now tracked at correct depot**

---

## **4️⃣ Transaction – Customer Sales, Meter Billing & Invoicing**

The **core of the money flow**.

**Main screen elements:**

* Select **Customer** → auto-loads payment type & rates
* **Three possible billing parts** (any combination allowed):

1. **Usage Billing** – for long-term installed customers

   * Last Meter Reading ← auto-filled
   * Current Meter Reading ← entered now
   * System calculates usage = current – last
   * Subtotal = usage × rate

2. **Cylinder Delivery** – normal sales

   * Dynamic rows (same style as distribution)

3. **Services** – installation, delivery fees, etc.

**Critical confirmation + invoice preview:**

```
Transaction Confirmation
User: Account 001
Customer: Hock Soon Heng Factory
Transaction #: TRX-20260119-0042
Timestamp: 19 Jan 2026 08:45

Breakdown:
• Usage Billing:     1,200 units × $0.45 = $540.00
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

**After [Confirmed]:**

1. Transaction saved
2. **Invoice PDF auto-generated**
3. **Printed immediately**
4. **PDF emailed automatically**
5. Inventory updated (cylinders now tracked at **client site**)
6. Audit log created
7. Invoice status tracked: **generated / printed / emailed / paid**

---

## **5️⃣ Cylinder Tracking Requirements**

The system **must always know cylinder locations**:

| Location Type   | How tracked                        | Updated when                               |
| --------------- | ---------------------------------- | ------------------------------------------ |
| **Depot**       | Inventory per depot per equipment  | After every confirmed Distribution         |
| **Client site** | Per customer – delivered cylinders | After every confirmed Transaction          |
| **In transit**  | Implicit (truck load) – optional   | During active distribution (not committed) |

**Invoice tracking:**

| Status    | When set                          | Visible to  |
| --------- | --------------------------------- | ----------- |
| Generated | After [Confirmed]                 | All         |
| Printed   | After successful mobile print     | Sales/Admin |
| Emailed   | After email send                  | Sales/Admin |
| Paid      | Manual mark / payment integration | All         |

---

## **6️⃣ Quick Architecture Reality Check**

| Layer                | Responsibilities                                                                         |
| -------------------- | ---------------------------------------------------------------------------------------- |
| **Frontend (React)** | Dynamic tables, confirmation dialogs, mobile print trigger, offline queue                |
| **Backend (Django)** | Atomic transactions, unique number generation, PDF creation, email sending, audit        |
| **Database**         | Distribution header/items, Transaction/Invoice, Customer meter, Depot & Client inventory |
| **Printing**         | Web Bluetooth + ESC/POS → thermal printer in field                                       |
| **Email**            | Backend sends PDF invoice automatically after confirmation                               |

---

## **7️⃣ Summary – Key Priorities**

1. Prevent wrong cylinder movement or billing → **mandatory confirmation + clear totals**
2. Fast & safe field data entry → **dropdowns + dynamic rows everywhere**
3. Generate & deliver correct invoice immediately → **print + auto-email**
4. Always know cylinder locations → **depot & client tracking**
5. Track invoice lifecycle → **generated / printed / emailed / paid**
6. Clear traceability → **unique numbers + user + timestamp + audit**

**Bottom line:**
This system allows field staff to **safely move cylinders**, **bill customers correctly**, **print invoices on the spot**, **email them automatically**, and **always know the exact location of every cylinder**.

