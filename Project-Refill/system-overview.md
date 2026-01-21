An Overview of the HSH Sales & Logistics System

Introduction: What is the HSH Sales System?

The HSH Sales System is a specialized, "field-first" platform designed to manage the complex logistics of Liquefied Petroleum Gas (LPG) distribution in Singapore. Its core purpose is to safely digitize the movement of physical assets like gas cylinders and manage customer billing in environments where internet connectivity is often unreliable. The system's architecture is a deliberate risk-containment strategy, built on the understanding that every digital record corresponds to a high-value physical asset. This ensures that the backend serves as the authoritative source of truth for all physical inventory and financial transactions.

1. The Core Problem: The Triangle of Constraints

We engineered the system to solve three primary business and operational constraints that define LPG field operations. These challenges form a "triangle of constraints" that the architecture must balance simultaneously.

* Field-First Operation: Drivers and sales staff frequently operate in industrial areas, basements, or service corridors with intermittent or non-existent internet connectivity, demanding an offline-first architecture with a reliable data synchronization queue.
* Transactional Integrity: In this industry, physical inventory (gas cylinders) and financial data (invoices) are inextricably linked. The system cannot allow these two records to be decoupled, demanding atomic, all-or-nothing database operations to prevent stock disputes and revenue loss.
* Zero-Trust Traceability: LPG cylinders are high-value, regulated assets that are often misplaced or lost in the field. The system must provide a clear, unambiguous, and tamper-proof trail of where every cylinder is at all times through strict, immutable audit logging.

"The backend is not just an API; it is the guardian of physical inventory and financial truth."

These operational problems are faced daily by the system's primary users.

2. System Users and Their Goals

The system is designed to serve field staff who are constantly on the move and require a tool that is fast, reliable, and straightforward.

Primary Users

* Delivery drivers
* Field sales staff

Key Operational Goals

1. Record the movement of gas cylinders between the central depot and their delivery truck.
2. Bill customers for services, cylinder deliveries, and gas usage based on meter readings.
3. Generate a correct and compliant invoice immediately upon completing a job at a customer's site.
4. Print a physical copy of the invoice on-site using a portable thermal printer.

To meet these goals, we built the system on a modern, dual-stack architecture.

3. High-Level Technical Architecture

The system is composed of a powerful backend that manages all business logic and a responsive frontend optimized for mobile, offline-first use.

Backend (Django)	Frontend (React)
Python/Django: Provides long-term support and stability for core logic.	React: Enables a modern, fast user interface with a robust component model.
Django Rest Framework (DRF): Creates a secure, modern, and well-documented API.	TanStack Query: Manages data fetching, caching, and synchronization with the backend.
WeasyPrint: Reliably converts HTML templates into compliant PDF invoices for email and archival.	Zustand: A lightweight solution for managing global application state, like user authentication.
Simple JWT: Secures the API using industry-standard JSON Web Tokens for mobile access.	localforage: Provides a reliable offline queue using IndexedDB, ensuring data persists even if the browser is closed.
MySQL: An ACID-compliant database with row-level locking, essential for preventing concurrency issues during inventory updates.	react-thermal-printer: Allows the app to connect directly to portable printers for on-site receipts.

This technology stack is organized around two fundamental business concepts that must remain distinct: logistical movements and customer sales.

4. The Data Foundation: The Two Worlds of Distribution vs. Transaction

The entire system is built on a critical distinction between two separate workflows: moving inventory for logistical purposes and selling inventory to a customer. Conflating these two "worlds" would lead to catastrophic accounting and stock-keeping errors.

Distribution (Logistics)

This workflow is purely for internal logistics—specifically, moving stock from a central depot to a driver's truck, or returning empty cylinders from the truck back to the depot. It is an inventory movement record, and no customer invoice is generated.

Transaction (Sales)

This workflow is for customer-facing sales activity. It involves billing a customer for gas usage, delivering new cylinders, or charging for services. This process directly involves money flow and client assets, and it always triggers the generation of a PDF invoice.

Let's examine how these two distinct workflows operate step-by-step in the field.

5. Core System Workflows Explained

The following workflows represent the day-to-day operations of a field user.

5.1. Workflow A: Distribution (The Logistics Engine)

This workflow dictates how a driver must safely record the movement of inventory between a depot and their truck.

1. Input: The driver selects the depot, the type of equipment (e.g., "CYL 12.7 kg"), the quantity, and the movement type (e.g., 'Collection' for full cylinders or 'Empty Return' for used ones).
2. Submission: The driver submits the form, which sends a POST request to the /distributions/ endpoint and creates a record with a 'Pending' status. This does not yet change official inventory counts.
3. Confirmation: After reviewing the details, the driver confirms the submission. This is a critical step, triggered by a POST request to the /distributions/{id}/confirm/ endpoint, that locks the record, updates the official DepotInventory, logs a permanent audit trail of the action, and generates a unique distribution number (e.g., DIST-2026-884) for tracking.

5.2. Workflow B: Transaction (The Sales Engine)

This is the process for making a sale and invoicing a customer at their location.

1. Trigger: The workflow begins when the driver is on-site with a customer and initiates a new transaction.
2. Calculation: The system calculates the total bill based on three potential components:
  * Meter Usage: Calculates the gas consumed since the last reading.
  * Cylinder Deliveries: Adds charges for any new cylinders delivered.
  * Service Fees: Includes any fixed fees for services like installation or deposits. Before calculating the bill, the system performs critical validation checks, such as ensuring the current meter reading is not lower than the previous one, to prevent billing errors.
3. Record Creation & Output: A successful submission creates both a Transaction Record and a corresponding Invoice Record. This action automatically triggers the generation of a formal PDF invoice, which is then emailed to the customer and permanently logged in the system's audit trail.

To ensure these complex, multi-step workflows are always reliable, the system incorporates several key features.

6. Key System Features for Reliability and Trust

Three core features work together to guarantee data integrity, operational reliability, and accountability, even in challenging field conditions.

6.1. Atomic Integrity: All or Nothing

The system uses atomic transactions to prevent data corruption and errors like "vanishing stock." When a driver confirms a sale, multiple database operations must occur in a single, unbreakable sequence—such as locking the inventory row, updating the stock quantity, and generating the invoice record. The @transaction.atomic process ensures that this group of operations either completes entirely or fails completely. If any single step fails, the entire process is rolled back, and no partial or incorrect data is saved.

6.2. Offline First: The client_temp_id Protocol

The system is designed to function without a stable internet connection. When a user creates a transaction offline, the application generates a temporary ID (client_temp_id) and saves the data securely on the device. Once an internet connection is restored, the application automatically syncs with the backend, sending the queued data. The backend processes the record, assigns it a permanent official ID (e.g., TRX-999), and sends that ID back to the app to complete the loop.

6.3. The Black Box: Immutable Audit Logging

We built an immutable audit logging system that functions like a plane's flight recorder, ensuring complete and tamper-proof accountability. It records every significant action with four key pieces of information:

* Who: The user who performed the action.
* What: The type of action (e.g., Create, Confirm, Update).
* Where: The entity that was affected (e.g., Transaction, Distribution).
* Context: A snapshot of the data involved in the action.

This log is strictly read-only, even for system administrators, creating a tamper-proof, chronological record that ensures complete accountability.

7. Summary: Bridging the Physical and Digital Worlds

Ultimately, the HSH Sales System acts as a secure bridge between the tangible world of physical logistics (trucks, paper receipts, and gas cylinders) and the precise world of digital finance (databases, invoices, and audit logs). It ensures that every action in the physical world is perfectly and reliably reflected in the digital record. This is achieved by building the entire system on three core attributes:

* Secure: Built with industry-standard JWT for authentication and guaranteed atomic transactions to protect every financial operation.
* Robust: An offline-first architecture using the client_temp_id protocol ensures uninterrupted operation in any field environment.
* Scalable: A domain-aligned Django application structure allows for future growth, such as QuickBooks integration or advanced BI reporting, without compromising core logic.
