**HSH Sales System – Realistic 4-Month Project Plan**  
**Customer:** Hock Soon Heng LPG Pte Ltd  
**Project Title:** HSH Sales System  
**Budget reference:** Stand-alone ≈ S$16,000 (one-time development)  
**Current date:** January 2026  
**Maximum realistic duration:** 4 months (~16 weeks)

### Final Scope Decision – What is **actually achievable** in 4 months

**Core MVP focus (must have – delivered end of month 4):**

1. Secure login + very basic role distinction (Sales vs Admin)
2. **Distribution** module (main business flow)
   - Select depot, cylinder type, quantity, collection/empty return
   - Add multiple items, preview, confirmation screen
   - Save → unique distribution number
3. **Basic Transaction** module (cylinder-focused first)
   - Select customer
   - Sell cylinders (quantities per type)
   - Very simple services (delivery/installation fee)
   - Basic total calculation (client preview only)
   - Save transaction → server validation
4. **Very basic offline tolerance**
   - Queue transactions & distributions when offline
   - Auto-sync when connection returns (simple last-write-wins)
5. **Basic thermal receipt printing** (browser → ESC/POS)
6. Simple list views
   - My recent distributions
   - My recent transactions
7. Minimal audit trail (server logs who did what, when)
8. Very basic inventory deduction (optimistic – warning only on negative)

**Explicitly out of scope for the 4-month budget & timeline (phase 2):**

- QuickBooks integration
- Full PDF invoice generation + email
- Advanced reporting & exports
- Meter reading with history/validation
- Complete Admin console (user mgmt, customer CRUD, rate changes)
- Sophisticated offline conflict resolution
- Multi-depot advanced visibility
- Full RBAC granularity
- Aging report, payment tracking, outstanding invoices

### 4-Month Aggressive Timeline (16 weeks)

| Phase                          | Weeks       | Main Deliverables                                                                                     | Owner / Focus                               | Risk Level |
|-------------------------------|-------------|-------------------------------------------------------------------------------------------------------|---------------------------------------------|------------|
| 1. Kick-off & Critical Decisions | 1–1.5     | • Final scope agreement<br>• Data model core tables<br>• API contract first draft (Swagger)<br>• Auth & security decisions<br>• Database choice (MySQL/PostgreSQL) | Project lead + senior dev                   | ★★★★ |
| 2. Backend Foundation           | 2–5.5      | • Users + JWT<br>• Basic Customer model (read-only for sales)<br>• Depot + Equipment types<br>• Inventory (simple counter)<br>• Distribution endpoint + service<br>• Basic Audit service | Backend developer                           | ★★★★★ |
| 3. Core Frontend + Distribution | 4–8.5      | • Login + protected routes<br>• Distribution form + multi-item<br>• Confirmation screen<br>• Basic offline queue (localStorage)<br>• Simple list of distributions | Frontend developer + backend integration    | ★★★★ |
| 4. Transaction Flow + Printing  | 7–11.5     | • Transaction form (cylinder priority)<br>• Basic service fees<br>• Save transaction + inventory touch<br>• Basic thermal receipt printing<br>• Transaction list | Frontend + backend + hardware testing       | ★★★★★ |
| 5. Polish, Offline Sync & Testing | 10–14     | • Offline → online sync (simple)<br>• Basic error handling & UX feedback<br>• Manual testing main flows<br>• Bug fixing sprint<br>• Basic security checklist | Full team + client early feedback           | ★★★★ |
| 6. Final Delivery & Handover     | 14–16     | • Final bug fixing<br>• Staging deployment (Docker + cloud)<br>• Basic developer handover doc<br>• Client UAT (2 weeks parallel)<br>• Small fixes after UAT | All hands + deployment engineer             | ★★★ |

### Most Realistic Team Configurations (S$16k budget reality – Jan 2026)

**Option A – Tightest (possible but high stress)**  
• 1 very strong full-stack developer (Django + React good experience)  
• Works ~90–100% time  
→ Risk: burnout, quality corners cut

**Option B – Most balanced & recommended**  
• Backend developer (Django focus) ≈ 60–70% time  
• Frontend developer (React + Tailwind) ≈ 70–85% time  
• Client side provides testing + printer hardware + feedback loop  
→ Best quality/speed compromise

**Option C – Very difficult to fit budget**  
Full-time pair + part-time PM/QA → usually exceeds S$16k in Singapore rates

### Critical Technical Decisions – Must lock-in **Week 1–2**

Decision                                    | Recommended choice (4-month version)         | Comment / Trade-off
--------------------------------------------|-----------------------------------------------|----------------------------------------------
Database                                   | MySQL 8 (as originally proposed)              | Faster setup, client familiarity
Authentication                             | JWT short-lived + localStorage                | Simplest realistic option
Offline storage                            | localStorage + simple queue                   | IndexedDB too time-consuming for MVP
Sync conflict strategy                     | Last-write-wins + manual check                | Sophisticated CRDT impossible in 4 months
Inventory safety                           | Optimistic + warning (not hard block)         | Full SELECT FOR UPDATE + rollback too slow
Printing                                   | Web Bluetooth (ESC/POS)                       | Most common & cheapest path
API documentation                          | drf-yasg (Swagger) minimal                    | Still very valuable even simplified
Frontend state                             | React Router loaders + actions + Zustand      | Good balance speed vs maintainability

### Final 4-Month Success Definition (realistic & honest)

At the end of 16 weeks the system should allow a sales person to:

1. Log in securely
2. Record cylinder collections & empty returns (distribution)
3. Sell cylinders to customers (basic transaction)
4. See a simple receipt on thermal printer
5. Continue working when internet drops (queue → sync later)
6. Have basic traceability (who did what, when)

Everything else (admin console, meter detail, advanced reports, QuickBooks, perfect offline conflict resolution, polished UX) → **Phase 2** (additional 4–8 months depending on priorities).

