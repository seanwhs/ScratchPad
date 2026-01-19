HSH Sales System Overview

This document provides a comprehensive overview of the HSH Sales System, a field-first LPG (Liquefied Petroleum Gas) sales and logistics solution designed for Singapore operations. It synthesizes system design, backend implementation, and frontend requirements into a technical reference for developers and architects.


--------------------------------------------------------------------------------


1. System Design and Architecture

The HSH Sales System is designed to solve the challenges of real-time inventory tracking and immediate billing in the field. It emphasizes fast data entry, physical and digital proof of sale, and precise cylinder tracking.

Core Philosophy and Workflows

The system distinguishes between two primary operational workflows to ensure data integrity across the supply chain:

Feature	Distribution Workflow	Transaction Workflow
Purpose	Logistics: Depot \leftrightarrow Truck movement.	Sales: Delivery and billing to customers.
Customer Involvement	No.	Yes.
Invoice Generation	No.	Yes (Includes Usage, Delivery, and Services).
Cylinder Tracking	Depot inventory levels.	Client site inventory levels.
Output	Optional receipt.	Thermal printed invoice + Automated PDF email.

Architectural Principles

* Mobile-First Design: Optimized for tablet and mobile use with minimal typing and mandatory human confirmation steps.
* Atomic Integrity: Critical operations (e.g., confirming a delivery) are wrapped in atomic transactions to ensure that inventory updates, invoice generation, and audit logging all succeed or fail together.
* Offline Tolerance: Supports field operations in areas with poor connectivity using a "last-write-wins" synchronization strategy and client-generated temporary IDs.
* Traceability: Every action is linked to a user and timestamp, assigned a unique number (e.g., TRX-..., DIST-..., INV-...), and recorded in an audit log.


--------------------------------------------------------------------------------


2. Backend Architecture and Implementation

The backend is built using Django 5.1 and Django REST Framework (DRF), utilizing a resource-oriented API design.

Technical Stack

* Framework: Django 5.1.4 (LTS) with DRF 3.15+.
* Database: MySQL 8 for production; SQLite for development.
* Authentication: JWT (JSON Web Tokens) via djangorestframework-simplejwt.
* Documentation: OpenAPI/Swagger via drf-spectacular.
* Utilities: WeasyPrint for HTML-to-PDF generation and Django SMTP for automated emailing.

API Design Principles

1. Versioning: URI versioning (e.g., /api/v1/) ensures backward compatibility.
2. Resource-Oriented: Every entity (customers, transactions, inventory) is a resource accessible via standard REST conventions.
3. Idempotency: Unique transaction and distribution numbers prevent duplicate processing.
4. Offline Sync Support: Creation endpoints accept client_temp_id. The server returns an official ID and timestamp for reconciliation.
5. Filtering & Pagination: Standardized list views include date ranges and entity filtering (e.g., ?customer=3&date_from=2026-01-01).

Core Data Models

* Accounts/Users: Managed via JWT; distinguishes between Sales and Admin roles.
* Depots & Equipment: Defines storage locations and cylinder types (e.g., 9kg, 12.7kg, 14kg, 50kg).
* Inventory: Tracks stock levels at both Depots and Client sites.
* Transactions/Invoices: Captures sales data, meter readings, and service fees, linked to an invoice tracking lifecycle (Generated \rightarrow Printed \rightarrow Emailed \rightarrow Paid).


--------------------------------------------------------------------------------


3. Frontend Architecture and Field Implementation

The frontend is a React 19 application designed for responsive, field-ready performance.

Technical Stack

* Build Tool: Vite 6.x.
* State Management: Zustand 5.x for lightweight global state (Auth, Offline Queue).
* Data Fetching: TanStack Query 5.x for caching and synchronization.
* Form Management: React Hook Form + Zod for type-safe validation.
* Styling: TailwindCSS 4.x for mobile-first utility styling.

Field-Specific Features

* Dynamic Entry Tables: Implementation of dynamic rows for Distributions and Transactions, allowing drivers to add multiple items (Equipment + Quantity + Status) quickly.
* Thermal Printing: Integration with Web Bluetooth and ESC/POS protocols to trigger immediate printing on mobile thermal printers.
* Offline Queue: Utilizes localforage (IndexedDB) to store transactions locally when the device is offline, with automated background synchronization upon reconnection.
* Confirmation Dialogs: Mandatory popups summarizing totals (e.g., Total Collection vs. Total Empty Return) before data is committed.


--------------------------------------------------------------------------------


4. Comprehensive Glossary

Term	Definition
Atomic Transaction	A database operation that ensures all steps (inventory update, log creation, invoice save) complete successfully or none at all.
Collection	In the Distribution workflow, refers to taking full cylinders out of a depot.
Empty Return	Returning empty cylinders back to a depot or from a customer site.
client_temp_id	A unique ID generated by the mobile app while offline to prevent duplicate records during later synchronization.
Usage Billing	Calculating costs based on "Current Meter Reading" minus "Last Meter Reading."
Service Items	Non-cylinder billables such as hose clips, regulators, or repair fees.
ESC/POS	A standard printer command language used by thermal receipt printers.
JWT	JSON Web Token; a secure way to handle user authentication between frontend and backend.
WeasyPrint	A library used by the backend to convert HTML/CSS templates into PDF invoices.
Last-Write-Wins	A simple conflict resolution strategy for offline sync where the most recent update is preserved.


--------------------------------------------------------------------------------


5. Study Quiz

1. Which backend technology is responsible for converting invoice templates into downloadable PDF files? A) Django SMTP B) WeasyPrint C) TanStack Query D) Zod

2. In the HSH Sales System, what is the primary difference between a "Distribution" and a "Transaction"? A) Distribution is for admins; Transaction is for sales staff. B) Distribution involves customers; Transaction does not. C) Transaction creates an invoice and tracks client inventory; Distribution tracks depot movements. D) Distribution requires a meter reading; Transaction does not.

3. How does the system handle data entry when a driver is in a location with no internet access? A) The app prevents any data entry until a signal is found. B) Data is stored in an offline queue via localforage and synced when back online. C) The driver must call the admin to enter the data manually into the backend. D) The app uses SMS to send data to the server.

4. What is the purpose of the @transaction.atomic decorator in the Django backend code? A) To speed up API response times. B) To ensure inventory integrity by rolling back changes if any part of the process fails. C) To automatically generate Swagger documentation. D) To encrypt JWT tokens before they are sent to the frontend.

5. Which frontend library is used for global state management and the offline queue? A) React Hook Form B) TailwindCSS C) Zustand D) Axios

6. Which status lifecycle is correct for an invoice in the HSH system? A) Pending \rightarrow Approved \rightarrow Paid B) Generated \rightarrow Printed \rightarrow Emailed \rightarrow Paid C) Draft \rightarrow Sent \rightarrow Collected D) Collection \rightarrow Delivery \rightarrow Return


--------------------------------------------------------------------------------


6. Answer Key

1. B (WeasyPrint)
2. C (Transaction creates an invoice and tracks client inventory; Distribution tracks depot movements.)
3. B (Data is stored in an offline queue via localforage and synced when back online.)
4. B (To ensure inventory integrity by rolling back changes if any part of the process fails.)
5. C (Zustand)
6. B (Generated \rightarrow Printed \rightarrow Emailed \rightarrow Paid)
