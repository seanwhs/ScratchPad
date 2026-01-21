HSH LPG Sales System: Backend Architectural Blueprint

Introduction: A Deep Dive into the Django 6 + DRF Logic Powering Singapore Field Operations

The HSH LPG Sales System is a specialized backend solution engineered to manage the logistics, sale, and traceability of high-value physical assets in the challenging field environments of Singapore. It is designed from the ground up to handle unreliable network connectivity, enforce strict financial and inventory controls, and provide an immutable audit trail for every operational action. This document serves as a comprehensive blueprint of the backend architecture, its underlying philosophy, and its critical design patterns, intended for technical stakeholders, developers, and integration partners.

The system's design is guided by a single, non-negotiable mandate: "The backend is not just an API; it is the guardian of physical inventory and financial truth."


--------------------------------------------------------------------------------


1.0 Engineering Philosophy: The Triangle of Constraints

The system's architecture is not an arbitrary collection of technologies; it is a direct and deliberate response to a specific set of operational risks inherent in LPG distribution. Every major design decision can be traced back to a guiding philosophy we call the "Triangle of Constraints." This framework balances the competing demands of field usability, financial accuracy, and asset security, ensuring the system remains robust and reliable under real-world pressure.

Core Principle	Architectural Mandate & Rationale
Field-First Operation	Mandate: An offline-first design with a robust synchronization protocol. <br/><br/> Rationale: The primary operational constraint is that drivers and sales staff operate in zones with intermittent or non-existent connectivity, such as basements, industrial estates, and service corridors. The architecture must assume that offline is the default state and connectivity is an optimization. This requires a resilient on-device queue and a server-side protocol (client_temp_id) to deterministically reconcile data without loss or duplication.
Transactional Integrity	Mandate: Atomic database transactions with full rollback capabilities for all state-mutating operations. <br/><br/> Rationale: Financial data cannot be decoupled from inventory data. A delivered cylinder is simultaneously a reduction in truck stock and a billable item on an invoice. Any failure that allows one record to be created without the other results in "vanishing stock" or unbilled revenue. The system must enforce an "all or nothing" principle, wrapping inventory updates, invoice generation, and audit logging in a single, indivisible transaction.
Zero-Trust Traceability	Mandate: A strict, immutable, append-only audit log for every state change. <br/><br/> Rationale: LPG cylinders are high-value, reusable assets that are frequently misplaced or lost. To ensure accountability, every significant action—from a depot collection to a customer delivery—must be logged with a user, a timestamp, and a data snapshot. The system treats every action as a forensic event, creating a "black box" flight recorder that provides an undeniable record of truth, protected from tampering even by administrators.

This philosophical foundation directly informs the selection of the technology stack, ensuring that the tools chosen are capable of enforcing these critical mandates.

2.0 The 2026 Technical Stack

The technology stack for the HSH Sales System was chosen with strategic intent, prioritizing stability, transactional safety, and long-term maintainability for 2026 operations and beyond. Each component plays a specific role in upholding the engineering philosophy, forming a cohesive, layered, and defensible architecture.

* Data Layer
  * MySQL 8 (Production) / SQLite (Development): Provides enterprise-grade, ACID-compliant storage. Its support for row-level locking is critical for safely managing concurrent inventory updates without data corruption.
* Authentication
  * Simple JWT: Delivers industry-standard, stateless authentication perfectly suited for mobile and web clients. This allows the backend to remain scalable and decoupled from client-side session management.
* Core Logic
  * Python 3.12 + Django 6 (LTS): A mature, secure, and powerful foundation. The Long-Term Support (LTS) version of Django guarantees stability, while its native support for asynchronous operations provides a path for future performance enhancements.
* API Interface
  * Django Rest Framework (DRF) 3.15+ & drf-spectacular: The premier toolkit for building robust, scalable web APIs in the Django ecosystem. It provides the architectural patterns for serializers and viewsets, while drf-spectacular automates the generation of OpenAPI / Swagger documentation, ensuring a clear and self-documenting API contract.
* Output Engine
  * WeasyPrint + SMTP: A powerful combination for generating physical and digital proof of transactions. WeasyPrint reliably converts standard HTML/CSS into PDF invoices, while SMTP integration handles the automated delivery of these documents via email, completing the operational workflow.

This stack is not merely a list of tools but a set of integrated components designed to deliver a secure, robust, and auditable system. The organization of these components within a strictly layered architecture is key to its success.

3.0 System Architecture: A Strictly Layered Design

The system adheres to a strict, layered architecture to enforce a clean separation of concerns. This design is not an academic exercise; it is a critical mechanism for ensuring that business logic is centralized, data integrity is maintained, and the system is auditable and extensible. Each layer has a single, well-defined responsibility and is forbidden from performing the duties of another. Logic leakage between layers is treated as an architectural defect.

Data Layer (Models) At the base of the architecture, the Django Models are the authoritative representation of physical reality and financial state. They define entities like cylinders, depots, customers, and invoices. Critically, these models are designed to reflect operational truth, not UI convenience. Their structure is optimized for data integrity and relational consistency, forming the stable foundation upon which all logic is built.

Serializer Layer (Validation) Serializers act as the system's vigilant gatekeepers. Positioned between the raw data from API requests and the core business logic, their sole responsibilities are to validate the structure and data types of incoming payloads, normalize data from offline clients (including the essential client_temp_id), and ensure that every request conforms to the expected contract. This layer is explicitly forbidden from making any state mutations; it validates, but it does not act.

Service Layer (Business Authority) This is the heart of the backend and the only layer where state changes are permitted. The Service Layer encapsulates all business logic, acting as the final authority on operational reality. Its exclusive responsibilities include:

* Mutating inventory levels.
* Calculating billing totals and generating invoices.
* Guaranteeing atomicity by wrapping operations in database transactions.
* Creating immutable audit log entries for every action. By centralizing these critical functions, the architecture ensures that business rules are applied consistently and cannot be bypassed.

API Layer (ViewSets) The API Layer, implemented as DRF ViewSets, serves as the public-facing entry point to the system. It is designed to expose explicit, intent-based actions (e.g., confirm_distribution, create_transaction) rather than generic CRUD operations. This layer is responsible for handling HTTP requests, authenticating users, and delegating all business logic to the appropriate function in the Service Layer. It makes no business decisions itself.

This layered approach is fundamental to making the system's robust data integrity guarantees possible, providing a clear and secure structure for the underlying data model.

4.0 The Data Foundation: The Two Worlds of Logistics and Sales

The core data model is built on a fundamental operational distinction: the strict separation of internal logistics from customer-facing sales. This is not merely a categorical convenience; it is a critical design choice that prevents operational ambiguity, ensures accurate inventory tracking, and maintains a clean financial ledger. Conflating these two worlds would lead to stock discrepancies and billing errors.

Distribution (Logistics)

Moving stock from Depot to Truck. Internal inventory movements. No invoices.

* Key Models:
  * DepotInventory
  * DistributionHeader
  * DistributionItem

Transaction (Sales)

Billing the customer. Money flow & client asset assets. Triggers PDF.

* Key Models:
  * CustomerSiteInventory
  * MeterReading
  * Transaction
  * Invoice

Both of these worlds are built upon a shared foundation of Equipment Models (e.g., CYL 9kg, CYL 50kg), which represent the physical assets being managed. This clear separation in the data model allows for dynamic yet safe workflows that operate on this stable foundation.

5.0 Core Operational Workflows

The system's functionality is expressed through two architecturally distinct workflows, each designed for clarity and operational safety in the field. These workflows map directly to the "Two Worlds" data model, ensuring that logistical movements and customer sales are handled by separate, purpose-built engines.

5.1 Workflow A: Distribution (The Logistics Engine)

The purpose of this workflow is to safely and accurately record the movement of physical assets between a depot and a truck. It is a purely logistical process with no financial implications.

The process follows a deliberate three-step, intent-based sequence:

1. Input: The driver uses the frontend application to select the Depot, Equipment type (e.g., CYL 12.7), and the desired Quantity.
2. Submission: An initial API call is made (POST /distributions/) to create a draft record. This record is saved with a "Pending" status, indicating that the physical transfer has not yet been finalized.
3. Confirmation: After reviewing the draft, the driver makes a final, explicit API call to an intent-based endpoint (POST /distributions/{id}/confirm/). This action locks the record and signals to the backend that the physical movement is complete and authoritative.

Upon confirmation, the backend performs three critical actions:

* Atomically updates the DepotInventory table.
* Logs an immutable audit trail of the action.
* Generates a unique, sequential DIST- number (e.g., DIST-2026-884).

The valid movement types for this workflow are Collection (Full Out) and Empty Return (Empty In).

5.2 Workflow B: Transactions (The Sales Engine)

The purpose of this workflow is to calculate customer billing, update client-side inventory, and issue official invoices. This is the primary revenue-generating engine of the system.

A single Transaction Record is a composite of three potential components, forming the final billable amount:

Usage (Meter Math) + Delivery (Cylinders) + Services (Fixed Fees) = Transaction Record

* Usage: Calculated from meter readings (Current Meter - Last Meter) multiplied by the customer's rate.
* Delivery: The sale or return of physical cylinders (e.g., Full Deliver, Half Return).
* Services: Fixed fees for activities like installation or deposits.

Before creating a record, the backend enforces critical validation logic to prevent common data entry errors:

* Check: The current meter reading must be greater than or equal to the last recorded reading.
* Check: The system provides an optimistic warning if a transaction would result in negative inventory at the customer site.

This core business logic is strictly compartmentalized, with validation rules residing in transactions/serializers.py and the authoritative record creation process managed by transactions/services.py. These workflows are underpinned by a set of foundational technical mechanisms that ensure they are robust, secure, and reliable.

6.0 Critical Cross-Cutting Concerns

Several foundational technical pillars underpin the entire system, ensuring its integrity, security, and offline capability. These are not features of a single module but cross-cutting concerns that are enforced globally across the architecture.

6.1 Atomic Integrity & Concurrency

To prevent "vanishing stock" and corrupted financial records caused by partial data updates, the system enforces an "All or Nothing" principle for every state-changing operation. This is achieved through atomic database transactions.

The four-step atomic process for a transaction is as follows:

1. The transaction status is saved as Confirmed.
2. The relevant Inventory database row is locked using select_for_update to prevent concurrent modifications.
3. The inventory quantity is updated (e.g., inventory += item.qty).
4. The final Invoice record is generated.

This entire sequence is wrapped in a @transaction.atomic block. If any step fails for any reason (e.g., database error, validation failure), the block triggers an immediate ROLLBACK, reverting all changes and leaving the database in its original, consistent state.

6.2 Offline Strategy & Synchronization

The client_temp_id protocol is the core mechanism enabling reliable offline-first operations. It allows the frontend to create records while disconnected from the network and safely synchronize them later without duplication.

The synchronization process follows this flow:

1. OFFLINE: The frontend application generates a unique temporary ID (e.g., temp_123) for a new record created by the user.
2. ONLINE: When connectivity is restored, a POST payload is sent to the backend, which includes the client_temp_id.
3. BACKEND PROCESSING: The backend saves the record, ignores the temporary ID, and assigns an official, permanent, server-generated ID (e.g., TRX-999).
4. SYNC: The backend's response includes a mapping of the temporary ID to the new permanent ID and the authoritative server timestamp, allowing the frontend to update its local data and reconcile the offline queue.

For conflict resolution, the system uses a simple and deterministic Last-Write-Wins strategy based on the last_modified timestamp, which is always set authoritatively by the server.

6.3 Authentication & Security

The system employs a standard, robust JWT-based authentication flow to secure all endpoints.

1. A user submits their credentials via a POST request to /auth/login/.
2. Upon successful validation, the server issues two tokens: a short-lived Access Token, which is stored in the client's memory (RAM), and a long-lived Refresh Token, which is stored securely.
3. The client uses the /auth/refresh/ endpoint with the Refresh Token to obtain new Access Tokens, keeping the user's session alive without requiring them to log in repeatedly.

Access is further controlled by a Role-Based Access Control (RBAC) model:

* Driver/User: Permitted to create Distributions and Transactions.
* Admin: Possesses elevated privileges, including User Management, setting Customer Rates, and accessing Reports.

6.4 The Black Box: Audit Logging

The audit log system provides immutable accountability for every significant action, acting as the system's "flight recorder." It ensures that a complete, tamper-proof history of operations is always available for review.

Each AuditLog entry captures four critical pieces of information:

* Who: The user_id of the actor who performed the action.
* What: The action taken (e.g., Create, Confirm, Update).
* Where: The entity_type that was affected (e.g., Transaction, Distribution).
* Context: A payload containing a JSON snapshot of the relevant data before or after the change.

As a critical security measure, the Admin View for these audit logs is strictly Read-Only to prevent any possibility of tampering or deletion.

6.5 Invoicing & Output Logic

The system uses a trigger-based process to automatically generate invoices and other physical or digital outputs upon the confirmation of a financial transaction.

1. Trigger: A transaction's status is changed to Confirmed.
2. Render: The backend renders a standard Django Template (invoice.html) into a pure HTML document populated with the transaction data.
3. Convert: The WeasyPrint engine consumes the HTML and converts it into a professional, standards-compliant PDF file.

This generated PDF can then be sent through two final output channels:

* Email: Sent directly to the customer via SMTP integration.
* Print: A JSON response is sent to the frontend, containing data formatted for an ESC/POS thermal printer.

The invoice itself follows a clear lifecycle, tracked by status enums: generated → printed → emailed → paid.

7.0 API Design Strategy & Contract

The API is designed to be a predictable, robust, and stable contract for client applications. Its design philosophy prioritizes clarity and consistency, ensuring that developers can interact with the backend in a logical and resource-oriented manner.

* Resource-Oriented: The API models entities as resources, not procedures. Endpoints map to nouns representing the data they manage (e.g., /customers/, /transactions/).
* Predictable Naming: Resource endpoints consistently use plural nouns, creating a predictable structure across the entire API (e.g., /distributions/ and /distributions/{id}/confirm/ (Action)).
* Versioning: All API endpoints are prefixed with a version number in the URI (e.g., /api/v1/...). This practice provides a clear path for future evolution without breaking existing client integrations.
* Filtering: The API supports standardized filtering via URL query parameters, allowing clients to efficiently retrieve specific subsets of data (e.g., /transactions/?customer=3&date_from=2026-01-01).

Responses for list endpoints use a standard DRF pagination structure, providing clients with metadata to handle large datasets gracefully. A simplified example of this response format is shown below:

{
  "count": 102,
  "next": "http://api/.../page=2",
  "previous": null,
  "results": [
    {
      "id": 1,
      "transaction_number": "TRX-2026-001",
      "status": "confirmed"
    },
    ...
  ]
}


The full, interactive API contract is available for review via the auto-generated Swagger UI documentation provided by the backend.

8.0 Developer Quick-Start

This section provides the essential commands and links for developers to quickly set up a local development environment and begin interacting with the backend API.

Terminal Setup

Execute the following commands in order from the project's root directory to install dependencies, initialize the database, and start the local development server.

1.

pip install -r requirements.txt # (Comment: Installing Django 6 + dependencies...)


2.

python manage.py migrate # (Comment: Initializing MySQL schema...)


3.

python manage.py runserver # (Comment: Server running at http://127.0.0.1:8000/)


Documentation Access

Once the server is running, the complete, interactive Swagger UI documentation is available at the following local URL: http://127.0.0.1:8000/api/schema/swagger-ui/

9.0 Summary: Bridging Physical & Digital

The HSH Sales System is architected to be more than just a data-entry application; it is a robust bridge between the complex realities of physical logistics—trucks, cylinders, and paper receipts—and the exacting demands of digital finance—invoices, databases, and audit logs. The entire backend is engineered to enforce consistency, accountability, and reliability in an environment where mistakes have immediate financial and operational consequences.

The architecture delivers on three core qualities that are essential for success in field operations:

* Secure: JWT authentication and role-based access control protect system entry points, while atomic transactions safeguard the integrity of every record from data corruption.
* Robust: The offline-ready architecture, built with an understanding of real-world field conditions, ensures that operations can continue and data can be captured reliably, even with intermittent network connectivity.
* Scalable: The modular Django App Structure provides a clean separation of concerns, allowing for the independent development and future expansion of features without compromising the stability of the core system.

This blueprint outlines a system that is secure, resilient, and built for the future. Review the Swagger docs to begin integration.
