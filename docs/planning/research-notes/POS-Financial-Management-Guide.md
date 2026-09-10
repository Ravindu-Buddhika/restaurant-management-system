# Restaurant POS System — Financial Management & Billing Module
### A Beginner-Friendly Guide to 3 Real-World Billing Scenarios

This guide explains three tricky billing situations that every restaurant POS (Point of Sale) system must handle correctly. Each scenario is broken down into:

1. **What the problem actually is** (in plain English)
2. **A real-world example** you can picture happening at an actual restaurant
3. **The step-by-step system flow** (what the software does, in order)
4. **The database tables involved** and why each field exists
5. **The math/logic formula** behind it

No prior database or coding knowledge is assumed — every term is explained the first time it appears, inline, as it comes up.

---

## Overview Table

| # | Scenario | The Core Problem | How the System Solves It |
|---|---|---|---|
| 1 | **Multi-Tender Overpayment** | Customer pays using more than one method (e.g., part card, part cash), and the cash given is *more* than needed | A payment ledger tracks how much was tendered, how much was actually applied to the bill, and how much change was returned |
| 2 | **Equal Split (Split by Amount)** | Dividing a bill evenly causes rounding errors in cents | A "cent adjustment" rule dumps the leftover cents into the last person's share |
| 3 | **Split by Items / Shared Items** | People want to pay only for what they ordered, but some items (like a shared pizza) belong to nobody in particular | The system splits shared items fractionally and applies discounts proportionally to each person's share |

---

## Scenario 1: Multi-Tender Overpayment (Cash + Card Mix)

### The Problem in Plain English
Sometimes a customer doesn't pay with just one method. They might swipe a card for part of the bill, then hand over cash for the rest — but the cash they hand over is more than what's actually owed. The system needs to know exactly how much came from each payment method, and correctly calculate the change to give back, **without losing track of the real total revenue.**

### Real-World Example
Imagine you run a small restaurant in Colombo. A customer's bill comes to **Rs. 5,000**.

- They tap their card for **Rs. 3,000** (partial payment).
- Now Rs. 2,000 is still owed.
- They only have a **Rs. 5,000 note** in cash — so they hand that over instead of asking for smaller notes.
- The cashier needs to:
  - Apply Rs. 2,000 of that cash toward the remaining bill
  - Give back **Rs. 3,000 as change**
  - Record that **only Rs. 2,000 cash actually stayed in the drawer**, not Rs. 5,000 — otherwise, when the cashier counts the drawer at the end of the day, the numbers won't match.

This exact situation happens constantly at cafés, food courts, and restaurants — someone pays with a mix of card + cash, or two different cards, or cash + a loyalty voucher.

### System Flow (Step by Step)

```
[Start Billing] ──> Select Order (Rs. 5,000)
       │
       ├── Payment 1: CARD ──> Customer taps card for Rs. 3,000
       │                       System applies Rs. 3,000 to the bill
       │                       Remaining Due = Rs. 5,000 − Rs. 3,000 = Rs. 2,000
       │
       ├── Payment 2: CASH ──> Customer hands over Rs. 5,000 note
       │                       System applies only Rs. 2,000 (what's still owed)
       │                       System calculates Change = Rs. 5,000 − Rs. 2,000 = Rs. 3,000
       │                       Cashier hands back Rs. 3,000 in change
       │
       └── [Finalize Bill] ──> Bill Status changes to PAID
                                Receipt is printed
                                Cash drawer opens automatically
```

### The Formulas (explained simply)

**1. How much is still owed, after each payment:**
```
Remaining Due = Grand Total − (sum of everything applied so far)
```
*Example:* Rs. 5,000 − Rs. 3,000 (card) = Rs. 2,000 still owed.

**2. How much change to give back:**
```
Change Given = max(0, Cash Handed Over − Remaining Due)
```
*"max(0, ...)"* just means "never go below zero" — you can't give negative change.
*Example:* max(0, Rs. 5,000 − Rs. 2,000) = Rs. 3,000.

**3. How much cash actually stays in the drawer:**
```
Actual Cash Added to Drawer = Cash Handed Over − Change Given
```
*Example:* Rs. 5,000 − Rs. 3,000 = Rs. 2,000 (this is the real cash income, not Rs. 5,000).

### Database Tables Involved

The key table here is **`payments`** — think of it as a diary that records *every single payment attempt*, even if a bill is paid using 2 or 3 different methods.

```sql
CREATE TABLE payments (
    payment_id VARCHAR(36) PRIMARY KEY,   -- unique ID for this one payment attempt
    bill_id VARCHAR(36),                  -- which bill this payment belongs to
    split_bill_id VARCHAR(36) NULL,       -- (used only in Scenario 2/3, leave NULL here)
    payment_method ENUM('CASH', 'CARD', 'QR', 'LOYALTY'),
    amount_tendered DECIMAL(10, 2),       -- what the customer physically handed over
    amount_applied DECIMAL(10, 2),        -- what actually counted toward the bill
    change_given DECIMAL(10, 2),          -- what was handed back to the customer
    transaction_ref VARCHAR(100) NULL,    -- card machine reference number, if any
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (bill_id) REFERENCES bills(bill_id)
);
```

**Why 3 separate money fields instead of just one "amount"?**
Because each one answers a different question:
- `amount_tendered` → "What did the customer physically give me?" (needed for the cash drawer count)
- `amount_applied` → "How much actually paid off the bill?" (needed for accounting/revenue)
- `change_given` → "How much did I give back?" (needed to prove the drawer balances)

If you only stored one number, you'd never be able to explain *why* the cash drawer has less money in it than the total cash customers handed over.

**How this row would look for our example (2 rows, since 2 payment methods were used):**

| payment_id | bill_id | payment_method | amount_tendered | amount_applied | change_given |
|---|---|---|---|---|---|
| PMT-001 | BILL-100 | CARD | 3000.00 | 3000.00 | 0.00 |
| PMT-002 | BILL-100 | CASH | 5000.00 | 2000.00 | 3000.00 |

---

## Scenario 2: Split by Amount (Equal Split with Cent Rounding)

### The Problem in Plain English
When a group of friends says "just split it evenly between us," simple division can create a rounding problem. Money only exists in whole cents (or whole rupees, in some currencies) — you can't hand someone a fraction of a cent. So when a bill doesn't divide evenly, *someone* has to absorb the tiny leftover amount.

### Real-World Example
Three friends have dinner together. The bill is **Rs. 1,000**, and they want to split it equally 3 ways.

```
Rs. 1,000 ÷ 3 = Rs. 333.3333...
```

If you round each person's share to Rs. 333.33 (2 decimal places, since that's the smallest unit of currency):

```
Rs. 333.33 × 3 = Rs. 999.99
```

That's **1 cent (Rs. 0.01) short** of the actual Rs. 1,000 bill! If the restaurant did this for every group bill, over thousands of transactions, they would slowly lose money. The system needs a rule for where that leftover cent goes.

**The chosen rule:** give the base amount (Rs. 333.33) to everyone *except the last person*, and give the last person the base amount *plus* whatever is left over — so the numbers always add up exactly.

### System Flow

```
[Start Split] ──> Total Bill = Rs. 1,000, Split Between = 3 people
       │
       ├── Step 1: Calculate base share per person
       │           1000 ÷ 3 = 333.333... → round DOWN to 333.33
       │
       ├── Step 2: Calculate the leftover (remainder)
       │           1000 − (333.33 × 3) = 1000 − 999.99 = 0.01
       │
       ├── Step 3: Assign shares
       │           Person 1 → Rs. 333.33
       │           Person 2 → Rs. 333.33
       │           Person 3 (last person) → Rs. 333.33 + Rs. 0.01 = Rs. 333.34
       │
       └── [Create Sub-Bills] ──> 3 separate sub-bills created, each marked "UNPAID"
                                   Total of all sub-bills = Rs. 1,000.00 (matches exactly)
```

### The Logic (in code form, explained line by line)

```javascript
function calculateEqualSplit(totalAmount, numPeople) {
    // Step 1: Work out each person's basic share, rounded DOWN to the nearest cent
    let baseAmount = Math.floor((totalAmount / numPeople) * 100) / 100; // → 333.33

    // Step 2: Find out how much money is "missing" if everyone paid the base amount
    let remainder = Math.round((totalAmount - (baseAmount * numPeople)) * 100) / 100; // → 0.01

    let splitBills = [];
    for (let i = 0; i < numPeople; i++) {
        // Step 3: Give the leftover cents to the LAST person only
        let billAmount = (i === numPeople - 1) ? (baseAmount + remainder) : baseAmount;
        splitBills.push({ sub_bill_no: i + 1, amount: billAmount.toFixed(2) });
    }
    return splitBills; // → [333.33, 333.33, 333.34]
}
```

**In plain English, this function does 3 things:**
1. Round each person's share *down* (never up — so we never accidentally overcharge someone before we know how much is left over)
2. Work out exactly how many cents were "lost" by rounding down
3. Add those lost cents onto the very last sub-bill, so the total always matches the original bill exactly

### Database Tables Involved

The **`split_bills`** table stores each person's individual share:

```sql
CREATE TABLE split_bills (
    split_bill_id VARCHAR(36) PRIMARY KEY,
    bill_id VARCHAR(36),                  -- links back to the original full bill
    split_type ENUM('EQUAL_SPLIT', 'BY_SEAT', 'BY_ITEM'),
    sub_total DECIMAL(10, 2),
    tax_share DECIMAL(10, 2),
    discount_share DECIMAL(10, 2),
    payable_amount DECIMAL(10, 2),        -- the final amount this person owes
    status ENUM('UNPAID', 'PAID') DEFAULT 'UNPAID',
    FOREIGN KEY (bill_id) REFERENCES bills(bill_id)
);
```

**Why is this a separate table instead of just a note on the receipt?**
Because each split needs to be tracked and paid *independently*. Person 1 might pay by card right away, while Person 2 pays 10 minutes later by cash. The system needs a way to know which parts of the bill are already paid and which are still pending — this table's `status` column does exactly that.

**How our example would look in this table:**

| split_bill_id | bill_id | split_type | payable_amount | status |
|---|---|---|---|---|
| SPLIT-01 | BILL-200 | EQUAL_SPLIT | 333.33 | UNPAID |
| SPLIT-02 | BILL-200 | EQUAL_SPLIT | 333.33 | UNPAID |
| SPLIT-03 | BILL-200 | EQUAL_SPLIT | 333.34 | UNPAID |

Once each person pays, a row gets added to the `payments` table (from Scenario 1) with `split_bill_id` filled in, pointing to which specific split this payment covers — this is *why* `payments.split_bill_id` exists as a nullable foreign key.

---

## Scenario 3: Split by Items / Shared Items (Proportional Discounts)

### The Problem in Plain English
Sometimes people don't want an *equal* split — they want to pay only for what they personally ordered. This gets complicated when:
- One item (like a large pizza) is shared between two or more people
- A discount applies to the *whole* bill, and needs to be fairly divided between people based on how much each person actually ordered

### Real-World Example
Two friends sit down at a table:

- **Seat 1** orders a **Burger** — Rs. 1,200
- **Seat 2** orders a **Pasta** — Rs. 1,500
- They **share a Large Pizza** together — Rs. 3,000
- The restaurant is running a **10% discount** on the whole order

**Step 1 — Total before discount:**
```
Rs. 1,200 (Burger) + Rs. 1,500 (Pasta) + Rs. 3,000 (Pizza) = Rs. 5,700
```

**Step 2 — Apply the 10% discount to the whole bill:**
```
Discount = Rs. 5,700 × 10% = Rs. 570
Net Payable = Rs. 5,700 − Rs. 570 = Rs. 5,130
```

**Step 3 — Split the shared pizza in half between the two seats:**
```
Pizza per seat = Rs. 3,000 ÷ 2 = Rs. 1,500 each
```

**Step 4 — Work out each seat's subtotal (before discount):**
```
Seat 1: Burger (1,200) + half Pizza (1,500) = Rs. 2,700
Seat 2: Pasta (1,500) + half Pizza (1,500) = Rs. 3,000
```

**Step 5 — Apply the 10% discount *proportionally* to each seat, based on their own subtotal (not equally!):**
```
Seat 1 discount: Rs. 2,700 × 10% = Rs. 270 → Net payable = Rs. 2,430
Seat 2 discount: Rs. 3,000 × 10% = Rs. 300 → Net payable = Rs. 2,700
```

**Check the math:** Rs. 2,430 + Rs. 2,700 = Rs. 5,130 ✅ (matches the total net payable exactly)

Notice that the discount isn't split 50/50 (Rs. 285 each) — it's split based on *how much each person actually ordered*, which is the fair way to do it.

### System Flow

```
                    [Original Order: Rs. 5,700]
                    [Discount 10%:  −Rs.   570]
                    [Net Payable:    Rs. 5,130]
                              │
           ┌──────────────────┴──────────────────┐
           ▼                                      ▼
   [Seat 1 Sub-Bill]                      [Seat 2 Sub-Bill]
   • Burger:        Rs. 1,200             • Pasta:         Rs. 1,500
   • Pizza (½):      Rs. 1,500             • Pizza (½):      Rs. 1,500
   • Subtotal:       Rs. 2,700             • Subtotal:       Rs. 3,000
   • Discount (10%): −Rs. 270              • Discount (10%): −Rs. 300
   ───────────────────────────             ───────────────────────────
   • Net Payable:    Rs. 2,430             • Net Payable:    Rs. 2,700
```

### How the System Tracks "Shared" Items

The **`order_items`** table has two special columns that make this possible:

```sql
CREATE TABLE order_items (
    item_id VARCHAR(36) PRIMARY KEY,
    order_id VARCHAR(36),
    item_name VARCHAR(100),
    unit_price DECIMAL(10, 2),
    quantity INT,
    seat_number INT NULL,          -- which seat ordered this? (NULL if shared)
    is_shared BOOLEAN DEFAULT FALSE, -- TRUE if multiple seats are splitting this item
    FOREIGN KEY (order_id) REFERENCES orders(order_id)
);
```

- `seat_number` → tells the system *whose* item this is. A Burger ordered by Seat 1 has `seat_number = 1`.
- `is_shared` → a flag (true/false switch) telling the system "don't assign this to one seat — divide it between everyone marked as sharing it."

**Our example in this table:**

| item_id | item_name | unit_price | seat_number | is_shared |
|---|---|---|---|---|
| ITM-01 | Burger | 1200.00 | 1 | FALSE |
| ITM-02 | Pasta | 1500.00 | 2 | FALSE |
| ITM-03 | Large Pizza | 3000.00 | NULL | TRUE |

When the system sees `is_shared = TRUE`, it looks up how many seats are sharing that item and divides the price accordingly (in this case, ÷ 2, since 2 seats are sharing).

The resulting per-seat totals then get saved into the same **`split_bills`** table used in Scenario 2 — except this time `split_type = 'BY_SEAT'` or `'BY_ITEM'` instead of `'EQUAL_SPLIT'`. This is exactly why the `split_type` column exists as an ENUM (a fixed list of allowed values) — the same table structure supports all 3 splitting methods.

---

## Putting It All Together: The Full Database Schema

Here is how all the tables connect to each other, and why:

```sql
-- 1. Orders Master Table — the "parent" record for a table's order
CREATE TABLE orders (
    order_id VARCHAR(36) PRIMARY KEY,
    table_number VARCHAR(10),
    status ENUM('OPEN', 'IN_BILLING', 'COMPLETED', 'VOIDED') DEFAULT 'OPEN',
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- 2. Order Line Items — every individual dish ordered, with seat/sharing info
CREATE TABLE order_items (
    item_id VARCHAR(36) PRIMARY KEY,
    order_id VARCHAR(36),
    item_name VARCHAR(100),
    unit_price DECIMAL(10, 2),
    quantity INT,
    seat_number INT NULL,
    is_shared BOOLEAN DEFAULT FALSE,
    FOREIGN KEY (order_id) REFERENCES orders(order_id)
);

-- 3. Master Bills Table — the overall bill for the whole table/order
CREATE TABLE bills (
    bill_id VARCHAR(36) PRIMARY KEY,
    order_id VARCHAR(36),
    sub_total DECIMAL(10, 2),
    discount_amount DECIMAL(10, 2) DEFAULT 0.00,
    tax_amount DECIMAL(10, 2) DEFAULT 0.00,
    service_charge DECIMAL(10, 2) DEFAULT 0.00,
    grand_total DECIMAL(10, 2),
    payment_status ENUM('PENDING', 'PARTIALLY_PAID', 'PAID') DEFAULT 'PENDING',
    FOREIGN KEY (order_id) REFERENCES orders(order_id)
);

-- 4. Sub-Bills / Split Bills Table — used for Scenario 2 and Scenario 3
CREATE TABLE split_bills (
    split_bill_id VARCHAR(36) PRIMARY KEY,
    bill_id VARCHAR(36),
    split_type ENUM('EQUAL_SPLIT', 'BY_SEAT', 'BY_ITEM'),
    sub_total DECIMAL(10, 2),
    tax_share DECIMAL(10, 2),
    discount_share DECIMAL(10, 2),
    payable_amount DECIMAL(10, 2),
    status ENUM('UNPAID', 'PAID') DEFAULT 'UNPAID',
    FOREIGN KEY (bill_id) REFERENCES bills(bill_id)
);

-- 5. Payments Ledger — used for Scenario 1 (and to settle Scenario 2/3 splits)
CREATE TABLE payments (
    payment_id VARCHAR(36) PRIMARY KEY,
    bill_id VARCHAR(36),
    split_bill_id VARCHAR(36) NULL,
    payment_method ENUM('CASH', 'CARD', 'QR', 'LOYALTY'),
    amount_tendered DECIMAL(10, 2),
    amount_applied DECIMAL(10, 2),
    change_given DECIMAL(10, 2),
    transaction_ref VARCHAR(100) NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (bill_id) REFERENCES bills(bill_id),
    FOREIGN KEY (split_bill_id) REFERENCES split_bills(split_bill_id)
);
```

### How the Tables Relate to Each Other (Relationship Map)

```
orders (1 table's order)
   │
   ├──< order_items (many dishes, each tagged with seat_number / is_shared)
   │
   └──< bills (the overall bill for that order)
              │
              ├──< split_bills (0 or more — only exists if the bill was split)
              │         │
              │         └──< payments (a split bill can be paid separately)
              │
              └──< payments (or the whole bill can be paid directly, no split)
```

*The `<` symbol means "one-to-many" — for example, one order can have many order_items, but each order_item belongs to only one order.*

---

## Core Best Practices (Why These Rules Matter)

| Rule | What It Means | Why It Matters |
|---|---|---|
| **ACID Transactions** | Use `START TRANSACTION ... COMMIT` at the database level | If a multi-tender or split payment fails halfway through (e.g., power cut, network drop), the database rolls back completely — so you never end up with a bill that's "half-paid" in a way that doesn't make sense |
| **Immutable Financial Logs** | Never `DELETE` or `UPDATE` a completed bill | Financial records must be tamper-proof for audits and legal compliance. If a mistake needs fixing, create a new "Void" or "Refund" entry in a separate audit table instead of erasing history |
| **Cash Drawer Balancing** | `Expected Cash = Opening Float + Cash Payments Applied − Petty Cash Payouts` | This formula lets a manager check, at the end of a shift, whether the actual cash in the drawer matches what the system expects — catching theft, mistakes, or missed transactions early |

---

## Summary Cheat Sheet

| Scenario | Key Table(s) | Key Formula | Real-World Trigger |
|---|---|---|---|
| 1. Multi-Tender Overpayment | `payments` | `Change = max(0, Tendered − Remaining Due)` | Customer pays with 2+ methods, one overpays |
| 2. Equal Split | `split_bills` | Base share rounded down; leftover cents go to the last person | Group wants to "just split it evenly" |
| 3. Split by Items | `order_items`, `split_bills` | Shared item price ÷ number of sharers; discount applied proportionally to each seat's subtotal | Group wants to pay only for what they personally ordered, including shared dishes |
