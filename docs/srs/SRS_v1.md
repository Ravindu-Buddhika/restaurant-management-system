# Software Requirements Specification (SRS)
## Restaurant Management System (RMS) — POS, Kitchen, Inventory, Delivery & Payments Platform

**Version:** 0.1 (Draft — Sample)
**Prepared for:** Internal Development Reference
**Based on source notes:** POS-Financial-Management-Guide.md, Online_Delivery_Orders_Management_Architecture.md, Restaurant_Stock_Management_Architecture_EN.md, SVI-Restaurant-Order-Management-System.md, takeaway-counter-sale-module.md, Card-QR-Payments-Integration-Guide.md, dsa-and-memory-in-backend.md, dynamic-rbac-abac-architecture.md, fnb-inventory-costing-spec.md, Git_Release_Versioning_and_Tagging_Strategy.md, monolithic-vs-microservices.md

---

## 1. Introduction

### 1.1 Purpose
This document specifies the functional and non-functional requirements for a **Restaurant Management System (RMS)** covering Dine-In, Takeaway/Counter Sale, and Online Delivery order channels, along with supporting Inventory, Costing, Payments, and Access-Control subsystems. It is intended as the baseline requirements document from which detailed design, database schemas, and sprint-level tasks can be derived.

### 1.2 Scope
The system will be built in **phases**, consistent with the project's versioning roadmap:

| Phase | Version Tag | Scope |
|---|---|---|
| Phase 1 | `v1.0.0` | Core POS — Admin Panel, Inventory, Menu Management, Staff Access, Billing (Dine-In + Takeaway) |
| Phase 2 | `v2.0.0` | Self-Checkout Kiosk |
| Phase 3 | `v3.0.0` | Online Customer Platform (web ordering, delivery integration) |

The system will run as a **Modular Monolith**: a single Spring Boot backend and single central database serving multiple client applications — a Tauri-based POS desktop app, a Kiosk app, and a React web app — with a possible future migration path to microservices if scaling demands it.

### 1.3 Intended Audience
Backend/frontend developers, QA engineers, project stakeholders, and future maintainers of the system.

### 1.4 Definitions, Acronyms, and Abbreviations

| Term | Meaning |
|---|---|
| KOT | Kitchen Order Ticket |
| KDS | Kitchen Display System |
| BOM | Bill of Materials (recipe mapping) |
| GRN | Goods Received Note |
| UOM | Unit of Measure |
| FIFO | First-In-First-Out (stock rotation rule) |
| RBAC / ABAC | Role-Based / Attribute-Based Access Control |
| COGS | Cost of Goods Sold |
| SSE | Server-Sent Events |
| JWT | JSON Web Token |
| EDC | Electronic Data Capture (integrated card terminal) |

### 1.5 References
This SRS consolidates the architecture and workflow notes listed in the header above. Each functional area below traces back to one or more of those source documents.

### 1.6 Document Overview
Section 2 gives the overall product description. Section 3 lists functional requirements by module. Section 4 covers external interfaces. Section 5 covers non-functional requirements. Section 6 summarizes the technical architecture. Section 7 covers process/versioning requirements. Appendix A summarizes the core data model. **Appendix B lists gaps that are not yet covered by the source notes and should be resolved before this SRS is considered complete.**

---

## 2. Overall Description

### 2.1 Product Perspective
The RMS is a **new, self-contained product** composed of:
- One Spring Boot backend (monolithic core, modular internally by domain: Orders, Inventory, Payments, Access Control).
- One central relational database (single source of truth for all order, stock, and payment data).
- Multiple client front-ends talking to the same backend APIs:
  - **POS App** (Tauri desktop) — cashier/waiter facing.
  - **Kiosk App** — customer self-checkout (Phase 2).
  - **React Web App** — online ordering / customer platform (Phase 3).
  - **Kitchen Display System (KDS)** — kitchen-facing order queue.

### 2.2 Product Functions (Summary)
- Dine-in table/session management with multi-KOT billing.
- Takeaway/counter sale with token-based fulfillment.
- Online delivery order ingestion (direct API or aggregator middleware) with hybrid accept modes.
- Real-time raw-material inventory deduction, yield/wastage tracking, and FIFO batch costing.
- Multi-tender payments, equal/seat/item-based bill splitting.
- Card terminal and QR payment integration (integrated, non-integrated, dynamic QR, static QR).
- Dynamic, tenant-configurable role and permission management (RBAC/ABAC).
- Day-end stock and cash reconciliation with variance analysis.

### 2.3 User Classes and Characteristics

| User Class | Description |
|---|---|
| **Cashier** | Takes payments, opens/manages takeaway and dine-in bills, handles split billing. |
| **Waiter** | Enters dine-in orders via handheld device, manages table sessions. |
| **Kitchen Staff / Chef** | Uses the KDS to view, prepare, and dispatch KOTs. |
| **Delivery/Online Order Coordinator** | Manages the manual-review queue for incoming delivery orders. |
| **Store/Branch Manager** | Approves cancellations/wastage, runs reconciliation, manages stock audits. |
| **Admin** | Configures roles/permissions, menu, pricing, and system-wide settings. |
| **Customer** | Places orders via kiosk or online web app (Phase 2/3). |

### 2.4 Operating Environment
- Backend: Spring Boot (Java), single central relational database (e.g., PostgreSQL/MySQL).
- POS Client: Tauri desktop application.
- Caching layer: In-memory (application RAM) and/or Redis.
- Deployment: Cloud VM / managed DB (e.g., AWS RDS) or VPS, horizontally scaled behind a load balancer.

### 2.5 Design and Implementation Constraints
- Must start as a **Modular Monolith**; must not prematurely fragment into microservices (per architecture decision).
- Backend must be **stateless** (session state via JWT) to support horizontal scaling across multiple app server instances.
- All card-terminal integrations must go through a common `CardTerminalAdapter` interface (Adapter Pattern) — no hardcoding of a single terminal brand.
- All monetary calculations must avoid floating-point rounding errors (use fixed-point/decimal types).
- Financial records must be **immutable** — corrections happen via new Void/Refund entries, never via UPDATE/DELETE on completed bills.

### 2.6 Assumptions and Dependencies
- Restaurants may use varying card machine brands/connection types (Serial, LAN, HTTP).
- Delivery platforms (UberEats, PickMe, etc.) provide either direct webhook APIs or are accessed via aggregator middleware (e.g., Deliverect, UrbanPiper).
- Internet/network outages are expected occasionally and the system must degrade gracefully rather than block sales.

---

## 3. System Features (Functional Requirements)

### 3.1 Dine-In Order Management
**Description:** Table-based ordering with an open session that can receive multiple KOTs before a single consolidated bill is settled.

- FR-3.1.1: The system shall mark a table as `OCCUPIED` when a session starts and `AVAILABLE` when the session closes.
- FR-3.1.2: The system shall create a Master Order under an open `pos_session` per table.
- FR-3.1.3: The system shall route kitchen items and bar items to separate printers/queues (multi-routing).
- FR-3.1.4: The system shall support multiple sequential KOTs (KOT 1, 2, 3…) under one session, printing only newly added items each time.
- FR-3.1.5: At checkout, the system shall consolidate all KOTs under a session into a single bill.
- FR-3.1.6: On payment completion, the system shall set `orders.status = PAID`, `pos_sessions.status = CLOSED`, and `restaurant_tables.status = AVAILABLE`.
- FR-3.1.7: Waiter devices shall not cache order data locally; all entries save directly to the server.

### 3.2 Takeaway / Counter Sale
**Description:** Single-transaction, no-table order flow with upfront payment.

- FR-3.2.1: The system shall bypass `restaurant_tables` and `pos_sessions` entirely for takeaway orders.
- FR-3.2.2: The system shall require payment **before** KOT generation (`payment_status = PAID` precedes kitchen routing).
- FR-3.2.3: The system shall generate a unique **Token Number** per takeaway order for customer callout.
- FR-3.2.4: The system shall tag takeaway KOTs for **express/priority queue** processing on the KDS.
- FR-3.2.5: The system shall support a single order lifecycle: `PLACED → PAID → IN_PROGRESS → READY → COMPLETED`.

### 3.3 Online Delivery Order Management
**Description:** Multi-channel ingestion of delivery orders via direct platform APIs or aggregator middleware.

- FR-3.3.1: The system shall expose a webhook/API endpoint to receive incoming delivery orders (direct integration) and/or connect to a normalized aggregator feed.
- FR-3.3.2: The system shall support three order-acceptance modes: **Auto-Accept** (off-peak), **Manual Review** (rush hour, cashier accept/delay/reject), and **Fallback/Manual Entry** (network/API outage).
- FR-3.3.3: The system shall push prep-time estimates back to the delivery platform to synchronize rider dispatch.
- FR-3.3.4: The system shall support "86'ing" — marking an item unavailable in-store shall push a stock-sync update to all connected delivery platforms simultaneously.
- FR-3.3.5: The system shall tag delivery KOTs with `DELIVERY - [PLATFORM NAME]` and a driver/order reference.
- FR-3.3.6: The system shall record `delivery_platform`, `channel_order_id`, `delivery_status`, `rider_info_json`, and `platform_commission` per order for reconciliation.
- FR-3.3.7: The system shall verify Driver ID/Order Reference at handover and update status to `DISPATCHED`/`COMPLETED`.

### 3.4 Inventory & Stock Management
**Description:** Raw-material tracking with flexible units, breakdown/butchery, yield/wastage, and hybrid/prep-batch products.

- FR-3.4.1: The system shall support a Base UOM per material with Secondary UOM conversions (e.g., kg ↔ pcs ↔ portion).
- FR-3.4.2: The system shall support Material Breakdown, allocating a master item into sub-parts by standard percentage or manual entry, and reconciling sub-part totals against the master quantity.
- FR-3.4.3: The system shall record yield/wastage during trimming/cleaning, applying an **effective (yield-adjusted) unit cost** for recipe costing while deducting stock at raw-equivalent quantity.
- FR-3.4.4: The system shall post a `WASTE-PREP` ledger transaction at the time of prep for any yield loss, so that "Current Qty" reflects real usable stock.
- FR-3.4.5: The system shall support Hybrid/Semi-Finished items usable both as standalone menu items and as sub-recipes within other BOMs (nested/multi-level BOM).
- FR-3.4.6: The system shall support Prep Batches (broths, bases) as their own stock category, deducted directly when used.
- FR-3.4.7: The system shall support Dynamic Scrap Sale (no fixed price) recorded as Other Income.
- FR-3.4.8: On Goods Receiving, the system shall record only the actual quantity received (with a Credit Note for shortfalls) and shall store goods on a FIFO basis.
- FR-3.4.9: Stock shall be deducted in real time the moment a KOT is generated (not when food is completed).
- FR-3.4.10: A dish shall only be finally committed against inventory once the Chef marks it "Dispatched/Passed" on the KDS; cancellations before that point shall return the item to a "virtual shelf" with no raw-material loss.
- FR-3.4.11: Order cancellation shall support three resolutions: Full Restock, Full Waste, and Partial Waste (ingredient-level selection by a manager).
- FR-3.4.12: Batch-based (FIFO) allocation shall be used for stock deduction across multiple active batches of the same item, splitting the deduction across batches as needed.

### 3.5 Costing, Reconciliation & Variance Analysis
- FR-3.5.1: The system shall calculate COGS per order in real time using yield-adjusted effective unit costs.
- FR-3.5.2: The system shall compute Theoretical Stock as `Opening Stock + GRN Inflow − Sales Deductions − System Waste`.
- FR-3.5.3: Day-end variance shall be computed at the **item level**, aggregating across all active batches (not a single batch in isolation).
- FR-3.5.4: Variance shall be categorized as Spoilage, Theft, or Over-Portioning, only after confirming it is not unlogged prep wastage.
- FR-3.5.5: The system shall support weekly/monthly physical stock audits with a documented variance-investigation workflow (waste log review, spot-weighing checks).

### 3.6 Billing, Multi-Tender Payments & Bill Splitting
- FR-3.6.1: The system shall record every payment attempt against a bill, storing `amount_tendered`, `amount_applied`, and `change_given` separately (to support mixed-method payment and accurate cash-drawer reconciliation).
- FR-3.6.2: The system shall compute change as `max(0, Cash Handed Over − Remaining Due)`.
- FR-3.6.3: The system shall support **Equal Split**, allocating rounding remainders (leftover cents) to the last participant so totals reconcile exactly.
- FR-3.6.4: The system shall support **Split by Seat/Item**, including fractional allocation of shared items and proportional (not equal) discount distribution based on each seat's subtotal.
- FR-3.6.5: The system shall support partial settlement of a bill via `split_bills`, independently trackable as `UNPAID`/`PAID`.
- FR-3.6.6: All completed bills shall be immutable; corrections must be made via Void/Refund entries in a separate audit trail.
- FR-3.6.7: The system shall support end-of-shift cash drawer balancing: `Expected Cash = Opening Float + Cash Payments Applied − Petty Cash Payouts`.

### 3.7 Card & QR Payment Integration
- FR-3.7.1: The system shall support Integrated (automated amount-send, auto-confirm) and Non-Integrated (manual amount entry, manual approval-code entry) card terminal flows.
- FR-3.7.2: The system shall implement a `CardTerminalAdapter` interface with pluggable adapters (HTTP REST, Serial Port, TCP Socket) selected via a factory, so new terminal brands can be added without changing core billing logic.
- FR-3.7.3: The system shall support Dynamic QR (system-generated, amount pre-filled, auto-confirmed via bank webhook) and Static QR (fixed sticker, customer enters amount, cashier manually confirms).
- FR-3.7.4: Incoming bank/payment webhooks shall be verified via signature/fingerprint check before being trusted.
- FR-3.7.5: Incoming payment webhooks shall be idempotent — a duplicate webhook for an already-recorded Order ID shall be ignored, not double-recorded.
- FR-3.7.6: If an integrated card terminal loses connection, the system shall offer a "Switch to Manual Entry" fallback without blocking checkout.
- FR-3.7.7: While awaiting a card-machine response, the UI shall remain responsive (non-blocking, spinner-based wait).
- FR-3.7.8: Manually entered approval/reference codes shall be stored in `payments.transaction_reference` for end-of-day bank settlement cross-checking.

### 3.8 Access Control (Dynamic RBAC/ABAC)
- FR-3.8.1: The system shall decouple atomic **Permissions** (system-defined, fixed) from **Roles** (tenant-created, dynamic) via a junction table.
- FR-3.8.2: Admins shall be able to create custom roles combining permissions across modules (e.g., Production + HR) without a code deployment.
- FR-3.8.3: All backend authorization checks shall evaluate permission strings (e.g., `hasAuthority('production:batch:create')`), never hardcoded role names.
- FR-3.8.4: Active permissions shall be embedded in the user's JWT/session payload at login.
- FR-3.8.5: The frontend shall conditionally render UI elements based on the user's permission list.
- FR-3.8.6: The system shall support fine-grained ABAC constraints (e.g., max discount %, max discount amount, allowed locations) stored as JSON metadata against a role-permission pair.
- FR-3.8.7: The system shall support multi-tenant role isolation (`tenant_id` scoping on roles).

### 3.9 Kitchen Display System (KDS) & Order Queuing
- FR-3.9.1: Standard orders shall be processed FIFO (Queue).
- FR-3.9.2: Priority orders (VIP, express/takeaway) shall bypass the standard queue via a Priority Queue mechanism, without manual reordering by staff.
- FR-3.9.3: The KDS shall display order source (Dine-In/Takeaway/Delivery), priority tag, and a countdown/prep timer.
- FR-3.9.4: Kitchen staff shall be able to mark items "Ready for Packing"/"Dispatched" to trigger downstream inventory commit and packing workflows.

### 3.10 Search & Menu Performance
- FR-3.10.1: Menu/price data shall be served from an in-memory cache (HashMap-based) to avoid database load on every kiosk/web request; cache shall invalidate and reload only on menu/price changes.
- FR-3.10.2: Barcode/item lookups shall resolve in constant time regardless of catalog size (hash-table-backed lookup).
- FR-3.10.3: Menu search-as-you-type/autocomplete shall return results within a few keystrokes without a leading-wildcard database scan (Trie-backed or equivalent in-memory structure).

---

## 4. External Interface Requirements

### 4.1 User Interfaces
- POS App (cashier/waiter): order entry, table map, billing, split-bill UI, payment collection screens.
- KDS Screen: kitchen order queue with priority indicators and timers.
- Customer-facing Display: dynamic QR code display, "Payment Success" confirmation.
- Admin Panel: role/permission management, menu/inventory configuration, reconciliation reports.
- Kiosk / Web App (Phase 2/3): customer self-order and online ordering flows.

### 4.2 Hardware Interfaces
- Card Machines (EDC): Ingenico, Verifone, Pax or equivalent, connected via Serial Port (USB/RS-232), LAN/TCP, or Local HTTP API.
- Kitchen/Bar Receipt Printers, multi-routed by item category.
- Barcode Scanners for fast item lookup at counter.
- Customer-facing display screen for QR/payment confirmation.

### 4.3 Software Interfaces
- Delivery Platform APIs (UberEats, PickMe, DoorDash, Deliveroo) — direct webhook/REST integration.
- Aggregator Middleware (e.g., Deliverect, UrbanPiper) — normalized order feed as an alternative to direct integration.
- Bank/Payment Gateway Webhooks — for Dynamic QR and card settlement notifications.
- Central Relational Database (PostgreSQL/MySQL or equivalent), accessed via the Spring Boot backend only.

### 4.4 Communication Interfaces
- REST/HTTP APIs between all clients and the backend.
- WebSockets/SSE for real-time push updates (payment confirmation, KDS updates) — avoiding constant client polling.
- Signed HTTP webhooks (inbound) from banks and delivery platforms.

---

## 5. Non-Functional Requirements

### 5.1 Performance
- NFR-5.1.1: Cached menu/price lookups shall resolve in O(1) time, target ~1–2ms server-side.
- NFR-5.1.2: Barcode/item lookups shall be O(1) regardless of catalog size.
- NFR-5.1.3: Card-machine payment waits shall not block the POS UI thread.

### 5.2 Scalability
- NFR-5.2.1: The backend shall be stateless and horizontally scalable behind a load balancer.
- NFR-5.2.2: Database connection usage shall be pooled (e.g., HikariCP) rather than opening unbounded direct connections.
- NFR-5.2.3: Read-heavy queries (menu browsing, order search) shall be separable onto read replicas as load grows.

### 5.3 Security
- NFR-5.3.1: All authorization decisions shall be permission-based (RBAC/ABAC), not hardcoded roles.
- NFR-5.3.2: Inbound payment webhooks shall be signature-verified.
- NFR-5.3.3: Sensitive configuration (API keys, bank credentials) shall not be hardcoded in client apps.

### 5.4 Reliability & Availability
- NFR-5.4.1: Delivery order ingestion shall degrade to Manual Entry mode on network/API failure without stopping service.
- NFR-5.4.2: Card payment flow shall degrade to Manual Entry on terminal disconnection.
- NFR-5.4.3: Webhook processing shall be idempotent to survive network retries safely.

### 5.5 Data Integrity
- NFR-5.5.1: All multi-step financial operations (split payment, multi-tender settlement) shall be wrapped in ACID transactions with rollback on partial failure.
- NFR-5.5.2: Completed bills shall never be hard-deleted or mutated; corrections go through Void/Refund records.
- NFR-5.5.3: Stock ledger entries (GRN, deduction, waste, prep-loss) shall be append-only for auditability.

### 5.6 Maintainability
- NFR-5.6.1: New card-terminal brands shall be addable via a new Adapter implementation without modifying core billing logic.
- NFR-5.6.2: New delivery platforms shall be addable via a new connector or via the aggregator without core order-processing changes.

### 5.7 Auditability
- NFR-5.7.1: Manually entered payment reference codes shall be retained for bank settlement cross-checks.
- NFR-5.7.2: Stock variance investigations shall be traceable to waste logs, portion audits, and prep-wastage entries.

---

## 6. System Architecture Summary

### 6.1 Architecture Style
- **Modular Monolith** for Phase 1–3: one Spring Boot codebase, one central database, multiple independent client apps (Tauri POS, Kiosk, React Web) consuming the same backend APIs.
- Migration to microservices (per-service database, Saga pattern for cross-service consistency, event brokers such as Kafka/RabbitMQ) is a deferred future option, not a Phase 1–3 requirement.

### 6.2 Scaling Model
- Horizontal scaling: identical Spring Boot instances behind a load balancer; JWT-based stateless sessions so any instance can serve any request.
- Primary DB (writes) + Read Replicas (menu browsing, order search — ~90% of traffic) to protect the primary from overload.

### 6.3 In-Memory / DSA Strategy

| Use Case | Structure | Reason |
|---|---|---|
| Menu/price caching | HashMap (+ LRU eviction) | O(1) reads, bounded memory |
| Barcode lookup | Hash Table | O(1) regardless of catalog size |
| Standard KDS queue | FIFO Queue | Arrival-order fairness |
| Priority/express orders | Priority Queue (Heap) | O(log n) insert, automatic priority bubbling |
| Menu autocomplete | Trie | Avoids leading-wildcard DB scans |
| Primary key / indexed lookups | B-Tree / B+ Tree (DB engine) | O(log n) vs. full table scan |

---

## 7. Other Requirements — Release & Version Control

- The project shall follow **Semantic Versioning** (`vMAJOR.MINOR.PATCH`).
- Branching shall follow a simplified Git Flow: `main`/`master` (production-stable), `develop` (integration), `feature/<name>` (per-task).
- Version tags shall align to phase milestones: `v1.0.0` (Core POS), `v2.0.0` (Self-Checkout Kiosk), `v3.0.0` (Online Customer Platform).
- Annotated tags shall be used for releases (`git tag -a vX.Y.Z -m "..."`) to preserve author/date/message metadata.

---

## Appendix A — Core Data Model Summary

| Table | Purpose |
|---|---|
| `restaurant_tables` | Table identity and occupancy status (Dine-In only) |
| `pos_sessions` | Open/closed session per table (Dine-In only) |
| `orders` | Master order record; carries `order_type` (DINE_IN/TAKEAWAY/DELIVERY), delivery fields, payment status |
| `kots` | Kitchen ticket per routing batch; status lifecycle |
| `order_items` | Line items; carries `seat_number`, `is_shared`, `modifiers_json` |
| `bills` | Bill totals (subtotal, discount, tax, service charge, grand total) |
| `split_bills` | Per-person/per-seat sub-bills; `split_type` ENUM (EQUAL_SPLIT/BY_SEAT/BY_ITEM) |
| `payments` | Every payment attempt; tendered/applied/change amounts; transaction reference |
| `batch ledger` (per item) | Batch ID, qty received/current, unit cost, expiry — FIFO costing source |
| `roles` / `permissions` / `role_permissions` | RBAC/ABAC model, tenant-scoped |

---


*End of Draft SRS v0.1*
