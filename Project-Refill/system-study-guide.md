HSH Sales & Logistics System Study Guide

Short-Answer Quiz

Answer the following questions in 2-3 sentences each, based on the provided source materials.

1. What is the core engineering philosophy of the backend, often referred to as "The Triangle of Constraints"?
2. Explain the fundamental difference between a "Distribution" workflow and a "Transaction" workflow in this system.
3. What is the purpose of using atomic transactions, and what specific problem do they prevent in the context of LPG logistics?
4. Describe the client_temp_id protocol and its role in the offline synchronization strategy.
5. Which two frontend technologies are explicitly named as being responsible for the "Offline Queue + Background Sync" feature?
6. How are PDF invoices generated in the backend, and what action triggers their creation?
7. According to the architectural blueprint, what is the single, authoritative layer responsible for changing reality (e.g., mutating inventory, calculating billing)?
8. Describe the purpose and key characteristics of the Audit Logging system.
9. Based on the 4-month project plan, name two features that are explicitly "in scope" for the MVP and two features that are "out of scope" until Phase 2.
10. How does the system handle user authentication, and what are the two types of tokens involved?


--------------------------------------------------------------------------------


Answer Key

1. The core philosophy is the "Triangle of Constraints," which balances Field-First Operation, Transactional Integrity, and Zero-Trust Traceability. This philosophy addresses key constraints: drivers having intermittent connectivity, gas cylinders being high-value assets that go missing, and the need for financial data to be coupled with inventory data. The backend is positioned as the "guardian of physical inventory and financial truth."
2. A "Distribution" is a logistics workflow for internal inventory movements between a depot and a truck, involving no customers or invoices. A "Transaction" is a sales workflow that involves billing a customer for cylinder sales, meter usage, or services, which triggers the generation of an invoice and updates client asset records.
3. Atomic transactions ensure that a series of database operations (like updating inventory and creating an invoice) are treated as a single "all or nothing" unit. This prevents "vanishing stock" and other data inconsistencies, such as billing a customer without successfully updating the inventory record, which is a common risk in unstable network environments.
4. The client_temp_id protocol is central to the offline strategy. The frontend generates a temporary ID (temp_123) for a record created offline and sends it to the backend upon reconnection. The backend processes the record, assigns an official server ID (TRX-999), and sends a sync response mapping the temporary ID to the new real ID, allowing the frontend to reconcile its local queue.
5. The frontend uses localforage and Zustand for the offline queue. localforage provides reliable offline storage using IndexedDB, while Zustand is used as a lightweight global state management library for both the auth and offline stores.
6. PDF invoices are generated using the WeasyPrint library, which converts HTML templates into PDF files. This process is automatically triggered when a transaction is confirmed via the create_customer_transaction_and_invoice service function.
7. The "Service Layer" is the only place where reality changes. This layer is explicitly responsible for all business-critical logic, including inventory mutation, billing calculation, state transitions (e.g., confirming a transaction), document generation, and creating audit records.
8. The Audit Logging system acts as an immutable "flight recorder" for every significant action. It captures the "Who" (user), "What" (action), "Where" (entity type), and "Context" (JSON payload snapshot). The admin view for these logs is strictly read-only to prevent tampering.
9. In Scope for MVP: Core features include the Distribution Module (Depot ↔ Truck), Basic Transaction Module (Cylinder Sales First), Basic Offline Tolerance with auto-sync, and Basic Thermal Receipt Printing. Out of Scope for MVP (Phase 2): More advanced features like QuickBooks integration, full PDF invoice generation and email, advanced reporting, and a complete Admin console for user management are deferred.
10. The system uses JSON Web Tokens (JWT) for authentication via the Simple JWT library. The authentication flow involves a user logging in to receive a short-lived Access Token (stored in-memory/RAM) and a long-lived Refresh Token (kept in secure storage). The Refresh Token is used to obtain a new Access Token to keep the session alive.


--------------------------------------------------------------------------------


Essay Questions

Suggest detailed, evidence-based answers for the following questions. No answers are provided.

1. Discuss the "Backend as Physical Truth" philosophy outlined in the architectural documents. How does this principle directly influence specific technical decisions, such as the use of atomic transactions, intent-based APIs (e.g., /confirm/), and the immutable audit log?
2. Analyze the project's comprehensive strategy for "Field-First Operation." Detail how both the frontend stack (React, Zustand, localforage) and the backend architecture (offline sync protocol, Last-Write-Wins conflict resolution) work together to address the core constraint of intermittent network connectivity.
3. Evaluate the 4-Month Project Plan. Discuss the major risks identified in the timeline and the critical trade-offs made in the technical decisions for the MVP, such as choosing localStorage over IndexedDB and "Last-write-wins" over CRDTs. Are these trade-offs justified for an initial delivery?
4. Describe the complete, end-to-end lifecycle of a customer transaction, starting from a driver entering data on the frontend in an offline environment. Trace the data flow through the offline queue, synchronization with the backend, processing by the service layer, database commit, and the final generation of an invoice for both thermal printing and email.
5. Explain the system's multi-layered approach to security and data integrity. Cover authentication (the JWT flow), authorization (Role-Based Access Control for Admin vs. Driver), and the architectural patterns that make certain failure scenarios "impossible by construction" (e.g., inventory mismatch, duplicate transactions).


--------------------------------------------------------------------------------


Glossary of Key Terms

Term	Definition
ACID	An acronym for Atomicity, Consistency, Isolation, Durability. It describes a set of properties for database transactions that guarantee data validity despite errors or power failures. The system uses ACID-compliant databases (MySQL/SQLite).
Atomic Transaction	A database operation that ensures a sequence of actions are treated as a single, indivisible unit. The entire sequence must succeed, or it is fully rolled back, preventing partial updates and data inconsistency (e.g., "vanishing stock").
Audit Log	An immutable, append-only ledger that records every significant user action. It captures the user, action, entity, timestamp, and a JSON payload snapshot, serving as forensic evidence for compliance and traceability.
client_temp_id	A temporary identifier generated by the frontend for a record created while offline. This ID is used to track the record until it syncs with the backend, which then returns an official, permanent server ID.
CRDT	Conflict-free Replicated Data Type. A sophisticated data structure for handling offline conflicts, which was explicitly rejected for the MVP in favor of the simpler "Last-Write-Wins" strategy for audit clarity.
Distribution	A core logistics workflow for recording the internal movement of equipment (e.g., gas cylinders) between a depot and a truck. It does not involve a customer or generate a sales invoice.
DRF (Django REST Framework)	A powerful and flexible toolkit built on top of Django for creating Web APIs. It is a central component of the backend's API Interface layer.
drf-spectacular	A Python library used with DRF to automatically generate OpenAPI 3 schemas and user-friendly API documentation interfaces like Swagger UI.
ESC/POS	A command language developed by Epson for controlling receipt printers. The frontend uses react-thermal-printer to send ESC/POS commands for printing receipts and invoices in the field.
Field-First Operation	A core design philosophy that assumes intermittent or poor network connectivity is the normal operating condition. This dictates the need for robust offline capabilities and synchronization strategies.
Intent-Based API	An API design strategy where actions that cause significant state changes (like finalizing a sale) are handled by specific endpoints (e.g., /distributions/{id}/confirm/) rather than generic CRUD methods (like PATCH or PUT).
JWT (JSON Web Token)	A compact, URL-safe standard for creating access tokens that assert a number of claims. The system uses Simple JWT for secure, stateless authentication for its API endpoints.
Last-Write-Wins (LWW)	A conflict resolution strategy for data synchronization where, in the event of a conflict, the last version of the data to be written to the server is accepted as the authoritative version. The system uses server time to determine the "last write."
localforage	A JavaScript library used by the frontend to provide a simple, asynchronous storage system with a localStorage-like API but backed by more reliable technologies like IndexedDB or WebSQL. It is a key part of the offline queue.
MVP (Minimum Viable Product)	The initial version of the system to be delivered within 4 months, containing only the most essential features required to be functional for field operations, such as basic distribution and transaction modules.
RBAC (Role-Based Access Control)	A security mechanism that restricts system access based on user roles. The system defines roles like "Driver/User" (can create distributions/transactions) and "Admin" (can perform user management and view reports).
Service Layer	The architectural layer in the backend where all critical business logic resides. It is the only layer authorized to mutate inventory, calculate billing, and change the state of the system, ensuring consistency and atomicity.
Swagger UI	An interactive user interface that allows developers and testers to visualize and interact with the API's resources without having any of the implementation logic in place. Generated by drf-spectacular.
TanStack Query	A powerful asynchronous state management library for React, used by the frontend for best-in-class data fetching, caching, and synchronization with the server.
Transaction	A core sales workflow that represents a billing instance with a customer. It can include meter usage, cylinder sales, and other services, and always results in the generation of an invoice.
WeasyPrint	A Python library used in the backend's "Output Engine" to convert HTML and CSS into PDF documents. It is used to generate PDF invoices from Django templates.
Zod	A TypeScript-first schema declaration and validation library. It is used with React Hook Form on the frontend to create type-safe forms with minimal re-renders.
Zustand	A small, fast, and scalable state-management solution for React. The frontend uses it for managing global state, specifically for authentication (JWT tokens) and the offline queue.

