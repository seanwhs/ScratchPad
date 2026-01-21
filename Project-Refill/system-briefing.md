HSH LPG Sales & Logistics System: A Comprehensive Briefing

Executive Summary

The HSH Sales & Logistics System is a production-ready, field-first digital platform engineered for Hock Soon Heng LPG Pte Ltd to manage its LPG sales and logistics for Singapore field operations. The system is architecturally centered on the non-negotiable mandate that the backend is the authoritative source of truth for all physical and financial states. It is designed as a risk-containment strategy to address the unique challenges of managing high-value, reusable physical assets (LPG cylinders) in environments with intermittent network connectivity.

The system's core design is governed by three principles: Field-First Operation, ensuring robust offline capabilities; Transactional Integrity, guaranteeing all-or-nothing database operations to prevent data corruption; and Zero-Trust Traceability, providing an immutable audit log for every significant action. This philosophy is manifested through a modern, dual-stack architecture: a robust Django 6 backend responsible for business logic and data integrity, and a responsive React 19 frontend optimized for on-site usability.

Key capabilities include a strict separation between logistics (Distribution) and sales (Transaction) workflows, atomic commits that prevent "vanishing stock," a sophisticated offline queue with a client_temp_id protocol for seamless data synchronization, and a comprehensive output engine for generating PDF invoices, email notifications, and ESC/POS thermal receipts. The entire system is designed for a 4-month MVP delivery with a budget of S$16,000, demonstrating a highly focused and aggressively scheduled project plan.

Core Engineering Philosophy: The Triangle of Constraints

The system's architecture is built upon a foundational philosophy that directly addresses the operational realities of LPG distribution. The central tenet is that "the backend is not just an API; it is the guardian of physical inventory and financial truth." This principle is a risk-containment strategy designed to prevent revenue leakage, stock disputes, and regulatory exposure by ensuring the digital record perfectly mirrors physical reality. This philosophy is upheld by balancing three critical constraints.

1. Field-First Operation:

* Constraint: Drivers operate in zones with intermittent connectivity (industrial estates, basements, service corridors).
* Solution: The system assumes offline operation is the baseline and connectivity is an optimization. The architecture features a robust offline queue on the frontend (localforage) and a backend reconciliation protocol (client_temp_id) to ensure data captured at the point of action is safely synchronized later.

2. Transactional Integrity:

* Constraint: Financial data cannot be decoupled from inventory data. Every transaction has a direct physical consequence.
* Solution: The system enforces atomic transactions (@transaction.atomic) with database-level row locking (select_for_update) for all critical operations. This "all or nothing" approach ensures that inventory updates, financial records, and invoice generation either succeed together or fail together, preventing partial commits and data inconsistencies.

3. Zero-Trust Traceability:

* Constraint: Gas cylinders are high-value, regulated assets that often go missing, requiring strict accountability.
* Solution: An immutable, append-only "Black Box" audit log records every state-changing action. It captures the user, action, entity, timestamp, and a JSON payload snapshot. The admin view of these logs is read-only to prevent tampering, providing a definitive forensic trail for every cylinder movement and financial transaction.

System Architecture and Design

The HSH system employs a modern, layered architecture with a clear separation of concerns between the backend, frontend, and data models. This structure is designed for scalability, maintainability, and security.

Backend: A Service-Driven, Layered Model

The backend is built with a strict, service-driven layered architecture where each component has a single, explicit responsibility.

* ViewSets (API Layer): Thin entry points that handle HTTP requests and responses. They delegate all business logic to the service layer and use serializers for data validation. They provide custom actions like /confirm/ for explicit state transitions.
* Services (Business Logic Layer): The authoritative core where all reality-changing operations occur. This layer is responsible for inventory mutation, billing calculations, lifecycle transitions, document generation, and audit logging. All service functions are wrapped in atomic transactions to guarantee consistency.
* Serializers (Validation Layer): Act as gatekeepers for incoming data. They validate structure, handle payload normalization (including client_temp_id for offline sync), but are explicitly forbidden from mutating inventory or finalizing state.
* Models (Data Layer): Represent physical reality, not UI convenience. The models are organized into domain-aligned Django applications (accounts, inventory, transactions, distribution, audit) to enforce clear boundaries and control.

Frontend: Offline-First and User-Centric

The frontend is a production-ready React application built for field usability on tablets.

* Key Features: It leverages React 19's new compiler and Router v7's Data APIs for performance and efficient data handling. Forms are type-safe using React Hook Form and Zod. Global state for authentication and the offline queue is managed by the lightweight library Zustand.
* Offline Capability: A central feature is the offline queue built with localforage (which uses IndexedDB), ensuring that distributions and transactions created without a network connection are stored safely and synchronized automatically when connectivity is restored.
* Hardware Integration: The frontend directly manages hardware interactions, specifically connecting to 80mm thermal printers via Web Bluetooth for on-site ESC/POS receipt printing.

Data Model Foundation: The Two Worlds

A critical architectural decision is the deliberate separation of the system into two distinct, non-conflatable workflows and data models.

Workflow	Purpose	Core Function	Invoice Generation
Distribution (Logistics)	Manages internal inventory movements between depots and trucks.	Records the collection of full cylinders and the return of empty ones.	No invoice generated.
Transaction (Sales)	Manages customer billing for products and services.	Calculates billing, updates client-site inventory, and triggers the sales process.	Invoice is always generated.

This separation prevents ambiguity between internal logistics and external sales, ensuring accurate inventory tracking and financial reporting.

Key Technical Capabilities and Workflows

The system's architecture enables several critical capabilities designed to ensure data integrity and operational reliability in the field.

Atomic Operations and Concurrency

To prevent "vanishing stock" and data corruption in high-concurrency scenarios, the system uses an atomic commit process for all state-changing operations.

1. Start Atomic Block: The operation begins within a @transaction.atomic decorator.
2. Lock Inventory Row: The system places a database lock on the relevant inventory row using select_for_update to prevent other processes from modifying it.
3. Update Quantity: The inventory quantity is updated.
4. Generate Records: The associated transaction and invoice records are generated.
5. Commit: All changes are committed to the database.

If any step fails, the entire transaction is rolled back, leaving the database in its original state and preventing partial updates.

Offline-First Strategy and Synchronization

The client_temp_id protocol is the cornerstone of the offline strategy.

1. Offline Creation: When offline, the frontend app generates a temporary ID (client_temp_id) for a new record (e.g., a transaction).
2. Queueing: The record is stored in the local offline queue.
3. Synchronization: Once online, the app sends a batch POST request to the backend, including the payload with the client_temp_id.
4. Backend Processing: The backend saves the record, assigns an official, permanent server ID (e.g., 'TRX-999'), and logs both the temporary and real IDs.
5. Reconciliation: The backend returns a map of temporary IDs to their new server IDs, allowing the frontend to update its state and clear the queue.
6. Conflict Resolution: The system uses a "Last-Write-Wins" strategy based on the last_modified server-side timestamp, a conscious trade-off to prioritize simplicity and audit clarity over more complex CRDTs for the MVP.

Invoicing and Output Generation

The system features a multi-channel output engine triggered upon transaction confirmation.

1. Trigger: A confirmed transaction initiates the flow.
2. Render: A Django template (invoice.html) is rendered into HTML.
3. Convert: The WeasyPrint engine converts the HTML into a PDF file.
4. Distribute: The output is delivered via two channels:
  * Email: Sent asynchronously via Gmail SMTP with the PDF attached.
  * Print: A JSON response containing an ESC/POS payload is sent to the frontend for thermal printing.
5. Lifecycle Tracking: The invoice status is tracked through enums: generated → printed → emailed → paid.

Authentication and Security

Security is managed through JWT and role-based access control.

* JWT Flow: Upon login, the server issues a short-lived Access Token (stored in memory) and a long-lived Refresh Token (kept in secure storage). The refresh token is used to seamlessly acquire new access tokens, keeping the session alive.
* Role-Based Access Control (RBAC): All critical API endpoints are protected. The system defines distinct roles with specific permissions:
  * Driver / User: Can create Distributions and Transactions.
  * Admin: Can perform user management, update customer rates, and generate reports.
  * Supervisor: Can oversee depots or approve transactions.

Technology Stacks (2026)

The project utilizes modern, stable, and production-ready technologies chosen for longevity, performance, and transactional safety.

Backend Technology Stack

Component	Choice	Rationale
Python	3.12	Latest stable version for performance and features.
Django	6	Long-Term Support (LTS) version with robust async support and stability.
Django REST Framework	3.15+	Industry-standard for building modern, scalable APIs.
Database	MySQL 8 (Prod) / SQLite (Dev)	ACID-compliant storage with reliable row-level locking for data integrity.
Authentication	Simple JWT	Stateless, industry-standard JSON Web Token implementation for secure mobile access.
OpenAPI Docs	drf-spectacular	Automatic generation of OpenAPI/Swagger documentation for the API.
PDF Generation	WeasyPrint	Reliable conversion of HTML templates to PDF for invoicing.
Email	Gmail SMTP	Field-ready and reliable SMTP service for delivering invoices.
Logging	Custom Middleware	Provides full API request logging for performance monitoring and auditing.

Frontend Technology Stack

Library / Tool	Version	Rationale
React	19.x	Leverages the new compiler, improved hooks, and suspense for a modern UI.
React Router	7.x	Full Data APIs (loaders, actions, defer) for efficient data fetching and state management.
Vite	6.x	Extremely fast development server and build tool.
TailwindCSS	4.x	High-performance, Just-In-Time (JIT) utility-first CSS framework for rapid UI development.
TanStack Query	5.x	Best-in-class library for data fetching, caching, and synchronization.
Zustand	5.x	Lightweight global state management for auth and offline stores.
React Hook Form + Zod	latest	Provides type-safe forms with minimal re-renders for a great user experience.
react-thermal-printer	latest	Enables direct ESC/POS thermal printing from the browser via Web Bluetooth.
localforage	latest	Provides a simple but reliable API over IndexedDB for the offline queue.

Project Scope and Delivery

The project is defined by a clear scope, an aggressive timeline, and a fixed budget, targeting an MVP delivery for Hock Soon Heng LPG Pte Ltd.

Project Mandate and Budget

* Customer: Hock Soon Heng LPG Pte Ltd
* Project Title: HSH Sales System
* Budget: S$16,000 for one-time development of a stand-alone system.
* Optional Integration: An additional S$13,800 for future QuickBooks integration.

4-Month MVP Scope

The initial 4-month delivery focuses on core field operations.

In Scope for MVP:

* Secure login with basic Admin vs. Sales role distinction.
* Distribution Module: For managing cylinder movements between depots and trucks.
* Basic Transaction Module: For cylinder sales and simple service fees.
* Basic Offline Tolerance: Queuing of distributions and transactions with auto-sync.
* Basic Thermal Receipt Printing: Browser-based ESC/POS printing.
* Minimal audit trail and optimistic inventory deduction.

Explicitly Out of Scope for MVP (Phase 2):

* QuickBooks integration.
* Full PDF invoice generation and emailing.
* Advanced reporting and BI exports.
* Meter reading with history and validation.
* A complete admin console for CRUD operations on users and customers.
* Sophisticated offline conflict resolution (e.g., CRDTs).

Aggressive 16-Week Timeline

The project follows a fast-paced 16-week plan with intentional overlaps to accelerate delivery.

* Weeks 1-5.5: Focus on backend foundation, including data models, APIs, and authentication.
* Weeks 4-8.5: Frontend development begins, focusing on the core distribution workflow and offline queue.
* Weeks 7-11.5: Development of the transaction flow and integration with thermal printing hardware.
* Weeks 10-16: Final polish, offline sync hardening, testing, bug-fixing, deployment, and client User Acceptance Testing (UAT).

The critical path is the stabilization of the backend foundation, which must be completed by Week 6 to avoid delaying the entire project.

API Design and Testing

The system's API serves as a formal contract between the frontend and backend, designed for clarity, predictability, and robustness.

API Design Strategy

* Resource-Oriented: Entities are exposed as resources (e.g., /customers/, /transactions/).
* Predictable Naming: Uses plural nouns for collections.
* Explicit Actions: Critical state changes are handled by dedicated action endpoints (e.g., /distributions/{id}/confirm/) rather than generic PUT/PATCH methods.
* Versioning: URI-based versioning (/api/v1/...) is used to allow for future evolution.
* Standardization: Utilizes standardized query parameters for filtering and standard DRF pagination responses.

Postman Testing Plan

A comprehensive Postman test plan is established to ensure API correctness and reliability.

* Workflow Simulation: The plan uses chained requests and environment variables ({{CUSTOMER_ID}}, {{TRANSACTION_ID}}) to simulate real-world user workflows.
* Comprehensive Coverage: An endpoint matrix details every API endpoint, its method, and its requirement for authentication.
* Key Areas Tested: The plan covers JWT authentication, CRUD operations for customers, inventory updates, distribution creation and confirmation, transaction creation, and invoice retrieval (PDF and email).
* Negative Testing: The plan recommends testing invalid inputs, missing fields, and expired tokens to ensure robust error handling.
