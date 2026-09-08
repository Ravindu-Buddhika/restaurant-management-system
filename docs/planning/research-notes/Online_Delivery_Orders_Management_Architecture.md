# Online Delivery Orders Management & Architecture

*A Comprehensive Guide to Integrating, Processing, and Managing Third-Party Delivery Orders in a Restaurant POS System*

---

## Table of Contents

1. [Core Architectural Concepts](#1-core-architectural-concepts)
2. [Order Processing Modes: The Hybrid Model](#2-order-processing-modes-the-hybrid-model)
3. [End-to-End Online Delivery Workflow](#3-end-to-end-online-delivery-workflow-step-by-step)
4. [Database Schema Adjustments for Online Delivery](#4-database-schema-adjustments-for-online-delivery)

---

## 1. Core Architectural Concepts

### A. Third-Party API Integration & Aggregator Architecture

In modern restaurant systems, handling online delivery platforms (e.g., **UberEats, PickMe, DoorDash, Deliveroo**) can be done via two main software integration patterns:

#### 🔹 Direct Platform Integration
- Each delivery platform exposes its own **Developer APIs** and **Webhook endpoints**.
- Your POS system acts as an **HTTP endpoint receiver**.
- When a customer places an order on UberEats/PickMe, the platform sends a real-time **HTTP POST payload (JSON)** directly into your POS system's API.

#### 🔹 Aggregator Middleware Integration (e.g., Deliverect, UrbanPiper)
- Instead of writing and maintaining separate API connectors for every delivery partner, your POS connects to a **single unified middleware hub** (Aggregator).
- The Aggregator receives incoming orders from all platforms, **normalizes the JSON data** into one standard structure, and pushes it into your POS backend.

> **Comparison at a Glance**

| Aspect | Direct Integration | Aggregator Middleware |
|---|---|---|
| Development effort | High (per-platform connector) | Low (one integration point) |
| Maintenance | Complex — each API changes independently | Simplified — aggregator handles updates |
| Data format | Platform-specific JSON | Normalized, standardized JSON |
| Scalability to new platforms | Slow, requires new dev work | Fast, usually plug-and-play |
| Cost | No middleware fee, but higher dev cost | Subscription/middleware fee |

---

### B. Dynamic Menu & 86'ing (Out-of-Stock Syncing)

- **Real-time Inventory Linkage:** When an ingredient or menu item runs out in-store, marking it as **"Unavailable" / "86'd"** on the POS triggers an outward API call (`PATCH` / `PUT`) to **all connected platforms simultaneously**.
- **Automation Benefit:** Prevents customers from ordering out-of-stock items, which reduces order cancellations and negative reviews.

### C. Automated Preparation Time (Prep Time) Engine

- **Dynamic Time Calculation:** Based on current kitchen load (number of active KOTs on the Kitchen Display System), the POS automatically calculates an **Estimated Preparation Time** — e.g., 15 minutes during off-peak vs. 35 minutes during rush hour.
- **Rider Synchronization:** This time estimate is pushed back to the delivery platform via API so that a driver/rider is dispatched to **arrive precisely when the food is packed**, avoiding cold food or driver congestion.

---

## 2. Order Processing Modes: The Hybrid Model

To handle real-world operational challenges (kitchen rush hours, internet outages, etc.), a robust POS system must implement a **Hybrid Order Acceptance Mechanism**.

```
                              [ Incoming Delivery Order (API / Webhook) ]
                                                   │
                                     ┌─────────────┴─────────────┐
                                     ▼                           ▼
                        [ Kitchen Status Normal ]     [ Kitchen Rush / Busy ]
                                     │                           │
                                     ▼                           ▼
                           (Mode 1: Auto-Accept)       (Mode 2: Manual Review)
                                     │                           │
                                     └─────────────┬─────────────┘
                                                   │
                                                   ▼
                                  [ Mode 3: Fallback / Manual Entry ]
                                     (Used if API / Network drops)
```

### 🟢 Mode 1: Fully Automated Accept (Auto-Accept Mode)

| | |
|---|---|
| **Trigger** | During off-peak hours or when active kitchen load is below a set threshold |
| **Execution** | 1. The API receives the payload and parses order details.<br>2. The POS automatically approves the order and responds to the platform API with a `200 OK` status and estimated prep time.<br>3. A **Kitchen Order Ticket (KOT)** is generated instantly — no manual cashier intervention needed. |

### 🟡 Mode 2: Managed Accept (Manual Review Mode)

| | |
|---|---|
| **Trigger** | During rush hours or high kitchen capacity |
| **Execution** | 1. Incoming orders appear as a **high-priority modal/pop-up** on the Cashier's screen with an alert chime.<br>2. The Cashier can **Accept**, **Delay (+10 mins)**, or **Reject** (e.g., if kitchen capacity is exceeded).<br>3. Once approved, the order routes to the kitchen and inventory is deducted. |

### 🔴 Mode 3: Fallback / Manual Entry Mode

| | |
|---|---|
| **Trigger** | Network outages, API downtime, or un-integrated platforms |
| **Execution** | 1. Orders arrive on dedicated platform tablets (e.g., an UberEats tablet).<br>2. The Cashier manually inputs the order into a dedicated **"Delivery Order Form"** on the POS.<br>3. Ensures inventory tracking and KOT generation remain consistent across all operational channels. |

---

## 3. End-to-End Online Delivery Workflow (Step-by-Step)

```
[1. Order Ingestion (API / Webhook)] → [2. Acceptance & Prep Time Sync] → [3. KOT Routing & Inventory Deduction]
                                                                                              │
 [6. Handover to Rider & Closure]  ←  [5. Packing & Rider Verification]  ←  [4. Kitchen Preparation (KDS)]
```

### Step 1 — Order Ingestion
- The customer places an order via **UberEats, PickMe**, or the restaurant's **direct web/mobile application**.
- Payment is processed online (Credit/Debit Card) or marked as **Cash on Delivery (COD)**.
- The platform transmits the payload (items, modifiers, customer notes, order ID) to the **POS API**.

### Step 2 — Order Acceptance & Prep Time Sync
- Based on the active mode (**Auto-Accept** or **Manual Review**), the order is accepted.
- The estimated prep time is transmitted back to the delivery platform to schedule rider dispatch.

### Step 3 — KOT Routing & Real-Time Inventory Deduction
- The order is converted into a **KOT** with a clear `"DELIVERY - [PLATFORM NAME]"` identifier and driver token/order reference.
- Ingredients are **deducted immediately** from real-time stock based on pre-defined recipe mappings.

### Step 4 — Kitchen Preparation & KDS Display
- The order appears on the **Kitchen Display System (KDS)** tagged with priority and countdown timer.
- Chefs prepare and plate the meal, marking status as **"READY FOR PACKING"** upon completion.

### Step 5 — Packaging & Rider Verification
- Packing staff verify the order against the KOT receipt, seal the delivery bag, and attach the **dispatch sticker**.
- The platform assigns a delivery rider who arrives at the restaurant counter.

### Step 6 — Handover & Order Closure
- Staff verify the **Driver ID / Order Reference** on the driver's smartphone app.
- The parcel is handed over, and staff update the POS order status to **DISPATCHED / COMPLETED**.

---

## 4. Database Schema Adjustments for Online Delivery

To support multi-channel online orders, the database design extends the core ordering tables as follows.

### 📋 Table: `orders` (Key Fields)

| Field Name | Type | Description |
|---|---|---|
| `order_id` | UUID / BIGINT | Primary identifier for internal system |
| `order_type` | ENUM | `DINE_IN`, `TAKEAWAY`, `DELIVERY` |
| `delivery_platform` | VARCHAR | `UBEREATS`, `PICKME`, `DIRECT_WEB`, etc. |
| `channel_order_id` | VARCHAR | External Order ID provided by UberEats/PickMe API |
| `delivery_status` | ENUM | `RECEIVED`, `ACCEPTED`, `PREPARING`, `READY`, `DISPATCHED`, `DELIVERED`, `CANCELLED` |
| `rider_info_json` | JSONB | Stores Driver Name, Phone Number, Vehicle Reg No, and Pickup ETA |
| `platform_commission` | DECIMAL | Commission fee charged by the delivery platform for financial reconciliation |

### 📋 Table: `kots` (Key Fields)

| Field Name | Type | Description |
|---|---|---|
| `kot_id` | UUID / BIGINT | Identifier for kitchen ticket |
| `order_id` | UUID / BIGINT | Foreign Key referencing `orders.order_id` |
| `status` | ENUM | `SENT`, `PREPARING`, `READY`, `DISPATCHED` |
| `order_type` | ENUM | Identifies ticket destination and priority tag |

### 📋 Table: `order_items` (Key Fields)

| Field Name | Type | Description |
|---|---|---|
| `item_id` | UUID / BIGINT | Primary Key |
| `kot_id` | UUID / BIGINT | Foreign Key referencing `kots.kot_id` |
| `menu_item_id` | UUID / BIGINT | Foreign Key for the menu item |
| `quantity` | INT | Quantity ordered |
| `unit_price` | DECIMAL | Price per unit (accounting for platform-specific markup prices) |
| `modifiers_json` | JSONB | Structured JSON containing paid add-ons, instructions, and exclusions |

---

## Summary

This architecture enables a restaurant POS to handle **multiple online delivery channels simultaneously**, with:

- ✅ Flexible integration (direct API or aggregator middleware)
- ✅ Real-time menu and stock synchronization
- ✅ Smart, load-aware order acceptance (Auto / Manual / Fallback)
- ✅ A clear, traceable order lifecycle from ingestion to handover
- ✅ A database schema designed to support multi-channel reconciliation and reporting

