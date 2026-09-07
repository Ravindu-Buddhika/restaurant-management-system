# SVI / Restaurant Order Management System – Dine-In Workflow & Architecture Summary

## 1. Key Technical Concepts & Architectural Decisions

### A. Real-Time Inventory & Material Deductions

**Trigger Point:** The moment a customer places an order and the Waiter/POS submits a **KOT (Kitchen Order Ticket)**, the stock of the relevant ingredients/raw materials is deducted **automatically and immediately**. The system does **not** wait until the food is actually ready before deducting stock.

**Reason:** Because the kitchen starts pulling raw materials as soon as the KOT is issued (not when the dish is finished), deducting stock at that exact moment allows out-of-stock situations to be detected in real time.

### B. Kitchen Prep Batches & Dispatch Logic (Virtual Shelf)

- **Prep Batches:** Frequently prepared components — such as fried rice bases, gravies, and broths — don't need to be deducted directly from raw materials each time. Instead, they can exist as a separate **"Prep Stock"** category.
- **Dispatch Logic:** A dish is only finally linked to a specific customer order once the **Chef marks it as "Dispatched / Passed"** through the KDS (Kitchen Display System).
- **Re-usability:** If an order is cancelled *before* the food has been plated and sent out, there is **no raw material loss** — the item can simply be returned to the kitchen's internal "virtual shelf" and re-assigned to the next order that needs it.

### C. Cancellation & Waste Management (Item-Level / Ingredient-Level)

When an order is cancelled, the system offers **three options**:

1. **Full Restock** – The food was never prepared, so 100% of the raw materials are returned to stock.
2. **Full Waste** – The food is fully prepared/finished, so 100% goes into the Waste Log.
3. **Partial Waste (Custom Ingredient Exclusion)** – For example, if the rice portion is wasted but the chicken portion is still usable, the manager can use the Manager UI to select *only* the wasted ingredients to send to the Waste Log, while the remaining (unused/unaffected) ingredients are restocked.

### D. Modifiers Logic (Add-ons vs. Instructions)

- **Paid Modifiers (Add-ons):** These increase the price, and the corresponding ingredient is deducted from stock (e.g., Extra Cheese).
- **Free Instructions:** These don't change price or inventory — they are simply printed on the KOT as a note (e.g., "Less Spicy").
- **Exclusion Modifiers (No / Without):** Price stays the same, but a backend recipe-deduction rule is triggered to skip deducting that specific ingredient (e.g., "No Chop Suey").

---

## 2. End-to-End Dine-In Workflow (Step-by-Step)

```
[1. Table Selection] → [2. Session & Master Order Start] → [3. Order Entry & KOT Creation]
                                                                    │
[6. Session Close]  ←  [5. Payment & Settlement]  ←  [4. Dispatch & Multi-KOT Loop]
```

### Step 1 – Table Selection & Session Start
- Once a table is selected, its status in `restaurant_tables` changes to **OCCUPIED**.
- A new open session record is created in the `pos_sessions` table, and a corresponding **Master Order** record is created under that session.

### Step 2 – Order Entry & Partial KOT Generation
- The waiter enters the order using their handheld mobile device.
- All data is saved directly to the **server** — the local device does not retain/cache the data.
- Based on multi-routing logic, kitchen items are printed to the **Kitchen Printer**, and drink items are printed to the **Bar Printer** (this becomes **KOT 1**).

### Step 3 – Running Order Loop (Multi-KOT Handling)
- If the customer later requests an additional drink or dessert, new KOTs (**KOT 2, KOT 3**, etc.) are added under the *same active session*.
- Only the newly added items are printed to the relevant kitchen/bar printer — not the entire order again.

### Step 4 – Checkout & Bill Printing
- When the customer requests the bill, the database combines **all KOTs under that session** (KOT 1 + KOT 2 + KOT 3 …) into a **single, consolidated bill**.

### Step 5 – Payment & Session Closure
- Once payment is completed:
  - `orders` status → **PAID**
  - `pos_sessions` status → **CLOSED**
  - `restaurant_tables` status → **AVAILABLE** (the table becomes free again)

---

## 3. Database Schema Structure (Dine-In Session Design)

| Table | Fields |
|---|---|
| **restaurant_tables** | `id`, `table_number`, `status` (AVAILABLE, OCCUPIED) |
| **pos_sessions** | `session_id`, `table_id`, `status` (OPEN, CLOSED), `start_time`, `end_time` |
| **orders** | `order_id`, `session_id`, `total_amount`, `payment_status` (PENDING, PAID) |
| **kots** | `kot_id`, `order_id`, `status` (SENT, PREPARING, DISPATCHED), `created_at` |
| **order_items** | `item_id`, `kot_id`, `menu_item_id`, `quantity`, `unit_price`, `modifiers_json` |

---

## 4. DB Performance & Hosting Cost Optimization

- With **fixed cloud servers** (e.g., AWS RDS, managed PostgreSQL) or a **dedicated VPS**, the monthly cost stays fixed regardless of how many queries are run.
- If using **serverless databases** (e.g., Firebase, Supabase), it's important to avoid constant second-by-second polling of the database. Instead, using **event-based requests** along with **WebSockets/SSE** can keep both the query count and the associated cost at a minimal level.

---

## 📌 Next Agenda

In the next session, the discussion will cover the two remaining major order methods for a restaurant system:

1. **Takeaway / Counter Sale** – The method where a customer pays directly at the cashier and takes the order.
2. **Online Delivery Orders** – Managing orders coming through platforms like PickMe, UberEats, or a company's own delivery app.
