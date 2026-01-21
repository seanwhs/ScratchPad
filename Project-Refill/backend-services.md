Understanding the HSH Backend: A Service-Driven Architecture

Introduction: The Guardian of Truth

When software manages physical goods, its role transcends simply responding to data requests. For a system managing high-value, regulated assets like LPG cylinders, this responsibility is absolute. The HSH backend is guided by a central philosophy that shapes every architectural decision:

"The backend is not just an API; it is the guardian of physical inventory and financial truth."

This philosophy is a risk-containment strategy designed to bridge the gap between a cylinder's physical reality—sitting in a depot, on a truck, or at a customer's site—and its digital state in the database. Any deviation is a direct threat that can manifest as lost revenue, stock disputes, or compliance failures. To uphold this standard, the system is built upon a strict, layered architecture that enforces discipline and ensures every action is deliberate, atomic, and auditable.

1. The Four Layers of Responsibility

The backend is built with a strict, service-driven layered architecture where each component has a single, explicit responsibility. This design ensures that business logic doesn't leak between layers, making the system robust, predictable, and maintainable.

Layer	Core Responsibility	Strictly Forbidden From
ViewSets (API Layer)	Handling HTTP requests and responses. Authenticating users and delegating immediately to the Service Layer.	Containing any business logic. Directly mutating the database state or inventory.
Serializers (Validation)	Acting as gatekeepers for all incoming data. Validating data structure and normalizing offline payloads (e.g., client_temp_id).	Mutating inventory or finalizing the state of any record. Making business decisions.
Services (Business Logic)	Acting as the authoritative core where all "reality-changing" operations occur: inventory mutation, billing, state changes.	Handling raw HTTP requests or responses. Interacting directly with the transport layer.
Models (Data Layer)	Representing physical reality, not UI convenience. Defining the structure and relationships of core business entities.	Containing complex business logic, calculations, or state-transition rules.

1.1. Models (The Data Layer): Representing Physical Reality

In the HSH system, Models are designed to be a direct digital representation of physical reality. They are not tailored for the convenience of a user interface but are structured to reflect the actual business domain. This is enforced by organizing them into domain-aligned Django applications with clear boundaries:

* accounts/ — User identities, roles, and authentication.
* inventory/ — Records of depot, truck, and client-site stock.
* transactions/, invoices/ — The billing and invoicing engine for customer sales.
* distribution/ — The internal logistics engine for moving stock between depots and trucks.
* depots/, equipment/ — The registry for physical locations and assets like cylinders.
* audit/ — The immutable, append-only ledger for all system activity.

1.2. Serializers (The Validation Layer): The Gatekeepers

Serializers act as the strict gatekeepers for any data entering the system. They are the first line of defense, ensuring that every payload is well-formed and valid before it can be processed further. Their responsibilities are tightly controlled:

* Validate Data Structure: They check that incoming data conforms to the expected format, types, and constraints (e.g., enums, required fields).
* Normalize Payloads: They prepare data for the services, most critically by handling the client_temp_id protocol. This allows the system to accept data created offline and reconcile it once a network connection is restored.

1.3. ViewSets (The API Layer): The Thin Entry Points

The ViewSets are the system's public-facing API endpoints, but they are intentionally designed to be "thin." They handle the mechanics of HTTP—parsing requests, authenticating users via JWT, and formatting responses—but contain no actual business logic.

Their primary role is to immediately delegate all processing to the appropriate function in the service layer. This strictness is a critical risk mitigation strategy; it prevents developers from accidentally bypassing the service layer's atomic guarantees, a severe risk in a system managing financial and physical assets. To ensure state changes are explicit, the API follows an "Intent-Based API" design strategy, using custom actions like /distributions/{id}/confirm/ to trigger specific, high-consequence operations rather than relying on generic PUT or PATCH requests.

1.4. Services (The Business Logic Layer): The Authoritative Core

The Service Layer is the heart of the backend. It is the authoritative core where all "reality-changing" operations occur. This is the only place where the system's state—whether financial or physical inventory—is mutated. Every function in this layer is designed to be a complete, self-contained business operation. Let's explore the specific functions that form this critical core.

2. A Deep Dive into the Service Layer

All critical business logic—billing calculations, inventory updates, and forensic auditing—is encapsulated in dedicated service functions. This ensures that every operation is consistent, atomic, and auditable. A key architectural guarantee is that every service function that mutates state is wrapped in the @transaction.atomic decorator, which ensures that all steps within the function either succeed together or the entire operation is rolled back. Furthermore, to prevent concurrency issues like race conditions, critical records like inventory are locked using select_for_update within the transaction, ensuring that only one process can modify a record at a time.

2.1. Transaction & Invoice Services (transactions/services.py)

This service manages the entire customer sales and billing workflow.

* Function: create_customer_transaction_and_invoice(user, data)
  * What it does: This is the primary function for processing a customer sale. Its responsibilities include:
    * Creating the core Transaction record.
    * Calculating billing for meter usage (current_meter - last_meter).
    * Generating a formal PDF invoice using the WeasyPrint library.
    * Optionally sending the generated PDF to the customer via email.
    * Creating a comprehensive entry in the AuditLog to record the entire process.
  * How it works: Wrapped in @transaction.atomic, this function executes its steps as a single, indivisible unit. If any part of the process fails—for example, if the PDF generation encounters an error—the entire operation is rolled back, and the database remains in its original, consistent state. Crucially, this process also locks the specific inventory rows using select_for_update, preventing race conditions where multiple transactions might attempt to claim the same stock simultaneously.

2.2. Distribution Services (distribution/services.py)

These services handle the internal logistics of moving cylinders between depots and trucks.

* Function: create_distribution(user, depot, items, remarks)
  * What it does: This function creates the initial distribution record, including a header and all associated line items. Crucially, it does not update any inventory counts at this stage; it merely records the intent to move stock.
* Function: confirm_distribution(distribution_id, user)
  * What it does: This function finalizes the stock movement. Its responsibilities include:
    * Marking the distribution as "confirmed" to prevent it from being processed again.
    * Atomically updating inventory quantities based on the movement direction (OUT for deliveries, IN for returns).
    * Logging the final inventory movement in the AuditLog.
  * How it works: This is the "intent-based" action that makes the physical movement official in the digital world. By separating creation from confirmation, the system ensures inventory is only changed when a user explicitly commits the action.

2.3. Inventory Services (inventory/services.py)

This service provides a controlled way to manipulate inventory records directly.

* Function: update_inventory(entity, entity_id, equipment_id, quantity, user)
  * What it does: This service directly updates or creates a CustomerSiteInventory record, which tracks the equipment a specific customer possesses.
  * How it works: Its core logic is built to maintain inventory integrity. It prevents stock from ever going into a negative value and ensures that any manual adjustment is captured in the AuditLog, providing a clear trail for any changes made outside the standard transaction or distribution workflows.

2.4. Audit Services (audit/services.py)

This service provides the system's "black box" flight recorder, ensuring complete and immutable traceability for every action.

* Function: log_action(user, action, entity_type, entity_id, payload)
  * What it does: This centralized function is called from within other services to record who did what, on which entity, and when.
  * How it works: The payload field is a critical feature that stores a JSON snapshot of key data related to the action (e.g., items, quantities, totals). This provides a complete forensic trail that can be used for compliance, debugging, and resolving disputes, making accountability non-negotiable.

2.5. Utility Services (core/utils/numbering.py)

This is a supporting service that guarantees operational uniqueness for key records.

* Function: generate_number(prefix)
  * What it does: This function generates unique, human-readable, and timestamp-based identifiers for core records, such as TRX for transactions, INV for invoices, and DST for distributions.
  * How it works: By using a timestamp-based approach, it ensures that no two core records can ever have the same number, which is critical for unique identification and audit traceability.

3. Conclusion: How the Architecture Upholds the Philosophy

The strict, service-driven architecture of the HSH backend is a direct implementation of its core philosophy. By enforcing a clear separation of concerns and centralizing all critical logic, the system ensures that it can reliably act as the guardian of truth. For any developer joining the project, understanding these three architectural pillars is key:

1. Services are the single source of truth. All business logic and state changes—without exception—must be executed through a service function. This prevents logic from being scattered and ensures every action is consistent and auditable.
2. Atomicity is non-negotiable. The pervasive use of @transaction.atomic combined with row-level locking (select_for_update) is the system's primary defense against data corruption. It prevents scenarios like "vanishing stock" or race conditions by guaranteeing that complex operations are treated as a single, all-or-nothing unit.
3. Views are thin, on purpose. ViewSets exist only to handle HTTP traffic and delegate to services. This discipline keeps the architecture clean, separates concerns, and prevents developers from bypassing the system's core integrity guarantees.

This layered and service-driven architecture is not an academic exercise; it is the technical enforcement of the system's mandate. The design makes catastrophic failure modes like "inventory mismatch between depot and audit log" or "duplicate confirmed transactions from retries" not just unlikely, but impossible by construction. It is how the backend successfully fulfills its role as the "guardian of physical inventory and financial truth."
