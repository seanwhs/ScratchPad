# **HSH Sales System – Realistic 4-Month Project Plan**

**Customer:** Hock Soon Heng LPG Pte Ltd
**Project Title:** HSH Sales System
**Budget Reference:** Stand-alone ≈ S$16,000 (one-time development)
**Current Date:** January 2026
**Maximum Duration:** 4 months (~16 weeks)

---

## **1. Final Scope – Achievable in 4 Months**

**Core MVP (must-have by end of month 4):**

1. Secure login + basic role distinction (Sales vs Admin)
2. **Distribution Module (Depot ↔ Truck)**

   * Select depot, cylinder type, quantity, collection/empty return
   * Multi-item rows, preview, confirmation screen
   * Save → unique distribution number
3. **Basic Transaction Module (Cylinder Sales First)**

   * Select customer
   * Sell cylinders (quantities per type)
   * Simple service fees (delivery/installation)
   * Basic total calculation (client preview only)
   * Save transaction → server validation
4. **Basic Offline Tolerance**

   * Queue transactions & distributions when offline
   * Auto-sync on reconnection (last-write-wins)
5. **Basic Thermal Receipt Printing** (browser → ESC/POS)
6. Simple list views:

   * My recent distributions
   * My recent transactions
7. Minimal audit trail (server logs who did what, when)
8. Very basic inventory deduction (optimistic; warns if negative)

**Explicitly out of scope for 4-month MVP (Phase 2):**

* QuickBooks integration
* Full PDF invoice generation + email
* Advanced reporting & exports
* Meter reading with history/validation
* Complete Admin console (user mgmt, customer CRUD, rate changes)
* Sophisticated offline conflict resolution
* Multi-depot advanced visibility
* Full RBAC granularity
* Aging reports, payment tracking, outstanding invoices

---

## **2. 4-Month Aggressive Timeline (16 Weeks)**

| Phase                                 | Weeks  | Main Deliverables                                                                                                                          | Owner / Focus                         | Risk  |
| ------------------------------------- | ------ | ------------------------------------------------------------------------------------------------------------------------------------------ | ------------------------------------- | ----- |
| **1. Kick-off & Critical Decisions**  | 1–1.5  | Final scope, core tables & data model, API contract (Swagger), Auth & security, DB choice (MySQL)                                          | Project lead + senior dev             | ★★★★  |
| **2. Backend Foundation**             | 2–5.5  | Users + JWT, Customer model (read-only), Depot & Equipment types, Inventory counters, Distribution endpoint & service, Basic audit service | Backend dev                           | ★★★★★ |
| **3. Core Frontend + Distribution**   | 4–8.5  | Login + protected routes, Distribution form + multi-item rows, Confirmation screen, Offline queue (localStorage), Simple distribution list | Frontend + backend integration        | ★★★★  |
| **4. Transaction Flow + Printing**    | 7–11.5 | Transaction form (cylinder-first), Service fees, Save transaction + inventory update, Thermal receipt printing, Transaction list           | Frontend + backend + hardware testing | ★★★★★ |
| **5. Polish, Offline Sync & Testing** | 10–14  | Offline → online sync (simple), Error handling & UX feedback, Manual testing, Bug-fix sprint, Basic security checklist                     | Full team + client feedback           | ★★★★  |
| **6. Final Delivery & Handover**      | 14–16  | Final bug fixes, Staging deployment (Docker + cloud), Developer handover doc, Client UAT (2-week parallel), Small fixes after UAT          | All hands + deployment engineer       | ★★★   |

---

## **3. Team Configurations (S$16k Budget)**

**Option A – Tightest**

* 1 full-stack dev (Django + React) ~100% time
* Risk: burnout, corners cut

**Option B – Balanced & Recommended**

* Backend (Django) 60–70% time
* Frontend (React + Tailwind) 70–85% time
* Client provides testing, printer hardware, feedback loop
* Best balance of speed, quality, and stress

**Option C – Difficult to Fit Budget**

* Full-time pair + part-time PM/QA → exceeds S$16k

---

## **4. Critical Technical Decisions (Week 1–2)**

| Decision               | Recommended Choice                       | Comment / Trade-off               |
| ---------------------- | ---------------------------------------- | --------------------------------- |
| Database               | MySQL 8                                  | Fast setup, familiar to client    |
| Authentication         | JWT (short-lived) + localStorage         | Simple & secure for MVP           |
| Offline Storage        | localStorage + simple queue              | IndexedDB too time-consuming      |
| Sync Conflict Strategy | Last-write-wins + manual check           | Full CRDT impossible in 4 months  |
| Inventory Safety       | Optimistic + warning                     | Full row-lock + rollback too slow |
| Printing               | Web Bluetooth (ESC/POS)                  | Cheap & field-ready               |
| API Documentation      | drf-yasg (Swagger) minimal               | Valuable even simplified          |
| Frontend State         | React Router loaders + actions + Zustand | Good speed vs maintainability     |

---

## **5. 4-Month Success Definition**

By week 16, the system should allow a salesperson to:

1. Log in securely
2. Record cylinder collections & empty returns (Distribution)
3. Sell cylinders to customers (basic Transaction)
4. See a simple receipt on a thermal printer
5. Continue working offline → auto-sync later
6. Maintain basic traceability (who did what, when)

**Everything else** → **Phase 2** (additional 4–8 months depending on priorities)

---

## **6. ASCII Gantt-Style 4-Month Plan**

```
HSH SALES SYSTEM – 4-MONTH PLAN (JAN 2026)
Weeks:  1    2    3    4    5    6    7    8    9   10   11   12   13   14   15   16

Phase 1: Kick-off & Critical Decisions
       ██████
       (Week 1–1.5)

Phase 2: Backend Foundation
            █████████████
            (Week 2–5.5)

Phase 3: Core Frontend + Distribution
                     ███████████████
                     (Week 4–8.5)

Phase 4: Transaction Flow + Printing
                                ███████████████
                                (Week 7–11.5)

Phase 5: Polish, Offline Sync & Testing
                                        █████████████
                                        (Week 10–14)

Phase 6: Final Delivery & Handover
                                                    ██████
                                                    (Week 14–16)
```

---

### **Legend & Notes**

* Each `█` ≈ 1 week of work
* Overlaps are **intentional**, representing parallel work:

  * Frontend starts while backend stabilizes
  * Transaction module overlaps Distribution frontend polish
  * Testing overlaps final feature development
* Phases 1–4 = feature implementation
* Phases 5–6 = polish, offline sync, QA, UAT, deployment

---

✅ **Highlights**

* Backend Foundation (Phase 2) is critical – all other modules depend on APIs & inventory logic
* Overlaps accelerate delivery without compromising MVP
* Offline queue + confirmation screens maintain **safety and traceability**
* Thermal printing + auto PDF/email are **Phase 2** enhancements

---

