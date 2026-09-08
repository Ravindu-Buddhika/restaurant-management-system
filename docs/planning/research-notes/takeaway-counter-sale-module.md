# Takeaway / Counter Sale Module

## 1. Core Concepts

### 1.1 No Table / Session Management
Unlike Dine-In, Takeaway orders do not require table reservation, table occupancy tracking, or long-running table sessions. The `restaurant_tables` and `pos_sessions` entities are bypassed entirely for this order type.

### 1.2 Upfront & Direct Settlement
In Dine-In, payment is collected at the end of the meal. In Takeaway / Counter Sale, payment is collected **upfront**, at the time of order entry, before the KOT is fired.

### 1.3 Immediate Stock Deduction
As with Dine-In, raw material inventory is deducted automatically the moment the order is placed and the KOT is generated — based on recipe/BOM mapping.

### 1.4 Single-KOT Execution
Takeaway orders are typically fulfilled as a single batch. There is no "add more items later" loop and therefore no multi-KOT cycle per order (unlike Dine-In, where a table session can generate several KOTs over time).

### 1.5 Express Queue Routing
Takeaway KOTs are tagged for **express processing** on the Kitchen Display System (KDS) so kitchen staff can prioritize fast counter turnaround over longer-prep Dine-In tickets.

---

## 2. End-to-End Workflow

```
[1. Order Entry & Customization]
              │
              ▼
[2. Upfront Payment & Bill Generation]
              │
              ▼
[3. KOT & Token Generation]
              │
              ▼
[4. Real-Time Inventory Deduction]
              │
              ▼
[5. Kitchen Prep & Callout / Display]
              │
              ▼
[6. Order Handover & Order Close]
```

### Step 1 — Order Entry & Customization
- Customer places an order at the counter.
- Cashier selects menu items and applies modifiers (Paid Add-ons, Free Instructions, Exclusion Modifiers) via the POS interface.

### Step 2 — Upfront Payment & Bill Generation
- Cashier collects payment (Cash, Card, QR, or Credit) immediately on order confirmation.
- Order status changes to `PAID`.
- A payment receipt is printed for the customer.

### Step 3 — KOT & Token Generation
- System generates a unique **Takeaway Token Number** for the order.
- Multi-routing logic sends order details to the relevant Kitchen or Bar printer(s).
- The printed KOT clearly shows a **"TAKEAWAY"** tag along with the Token Number.

### Step 4 — Real-Time Inventory Deduction
- On KOT generation, raw materials are deducted from inventory immediately, based on recipe mapping rules.

### Step 5 — Kitchen Preparation & Callout / Display
- Kitchen prepares the order via the KDS.
- On completion, kitchen staff mark the KOT status as `READY`.
- The Token Number appears on the Customer Display Screen as **"READY"** (or is called out by counter staff).

### Step 6 — Order Handover & Order Close
- Customer presents their token and collects the packaged food.
- Order status updates to `COMPLETED`.

---

## 3. Database Schema (Takeaway Adjustments)

### 3.1 `orders`
| Field | Notes |
|---|---|
| `order_id` | Primary key |
| `order_type` | `TAKEAWAY` |
| `token_number` | Unique per-day/per-shift token |
| `total_amount` | Final billed amount |
| `payment_status` | `PAID` (set at Step 2, before KOT) |
| `status` | `PLACED → PAID → IN_PROGRESS → READY → COMPLETED` |

### 3.2 `kots`
| Field | Notes |
|---|---|
| `kot_id` | Primary key |
| `order_id` | FK → `orders.order_id` |
| `status` | `SENT → PREPARING → READY → DISPATCHED` |
| `order_type` | `TAKEAWAY` (denormalized for fast KDS filtering/routing; source of truth remains `orders.order_type`) |

### 3.3 `order_items`
| Field | Notes |
|---|---|
| `item_id` | Primary key |
| `kot_id` | FK → `kots.kot_id` |
| `menu_item_id` | FK → menu master |
| `quantity` | — |
| `unit_price` | — |
| `modifiers_json` | Paid add-ons / free instructions / exclusions |

> **Note:** `restaurant_tables` and long-running `pos_sessions` are intentionally bypassed for Takeaway orders — there is no table state to manage.

---

## 4. Corrections Made From the Original Draft
1. Unified the Step 2 heading — original text used two different names ("Upfront Payment & Bill Generation" in the flow diagram vs. "Upfront Payment & Final Settlement" in the step detail). Standardized to **"Upfront Payment & Bill Generation"**.
2. Added the missing `READY` status to `kots.status` — Step 5 references marking the order "READY," but the original enum (`SENT, PREPARING, DISPATCHED`) didn't include it.
3. Added an explicit `orders.status` lifecycle (`PLACED → PAID → IN_PROGRESS → READY → COMPLETED`), since the original schema only listed `payment_status` and never defined the order's overall status progression referenced in Steps 2 and 6.
4. Flagged `kots.order_type` as a denormalized field (duplicated from `orders.order_type`) — kept intentionally for fast KDS express-queue filtering, but noted `orders.order_type` as the source of truth to avoid data-consistency bugs.
5. Cleaned up the workflow diagram into a single top-to-bottom flow (the original had a confusing two-row/backwards-arrow layout).
