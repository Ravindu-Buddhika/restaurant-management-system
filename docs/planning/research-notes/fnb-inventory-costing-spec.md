# F&B Inventory, Costing & Reconciliation System — Corrected Spec

> **Note on corrections:** The original draft was arithmetically correct throughout, but had three
> logic gaps — a missing "prep wastage" ledger entry, an inconsistent scope in the Phase 5 variance
> example (single batch vs. total item stock), and an undocumented convention linking recipe quantity
> to batch deduction quantity. All three are fixed and called out below.

---

## Phase 1 — Procurement & Stock Inflow (GRN & Batch Creation)

As soon as raw materials arrive from a vendor, they are entered into the system as a new batch.

- **Batch Generation:** Every inbound stock lot gets a unique Batch ID.
- **Metadata Attachment:** Expiry Date, Purchase Unit Cost, Vendor, and Received Date are attached directly to the batch.

**Batch Storage Ledger**

| Batch ID | Item | Qty Received | Current Qty | Unit Cost | Expiry Date |
|---|---|---|---|---|---|
| BAT-CHK-001 | Chicken Patty (Raw) | 5.0 kg | 5.0 kg | Rs. 1,000 / kg | 2026-09-10 |
| BAT-CHK-002 | Chicken Patty (Raw) | 5.0 kg | 5.0 kg | Rs. 1,100 / kg | 2026-09-15 |

---

## Phase 2 — Recipe (BOM) & Yield Adjustment

When a raw ingredient is prepped (trimming/boning), the resulting wastage is factored into recipe costing.

**Yield % & Effective (True) Cost**

If 1 kg of chicken (Rs. 1,000) yields 800 g of usable product after cleaning (Yield 80%):

$$\text{Effective Unit Cost} = \frac{\text{Purchase Cost}}{\text{Yield \%}} = \frac{\text{Rs. 1,000}}{0.80} = \text{Rs. 1,250 / kg}$$

**Bill of Materials — 1 Chicken Burger**

| Ingredient | Quantity | Cost |
|---|---|---|
| Chicken Patty (yield-adjusted) | 150 g (0.15 kg) | Rs. 1,250/kg effective |
| Burger Bun | 1 unit | Rs. 40 |
| Cheese Slice | 1 unit | Rs. 30 |

**⚠ Fix — missing prep-wastage transaction:** Applying the 1,250/kg *effective* rate assumes 20% of
every kg purchased is lost during trimming. That loss must be posted to the batch ledger as its own
transaction at the time of prep (not silently absorbed into the costing rate only):

| Batch ID | Transaction | Qty | Reason |
|---|---|---|---|
| BAT-CHK-001 | `WASTE-PREP` | −X kg | Trimming/boning loss (20% of prepped raw qty) |

Without this entry, "Current Qty" in the ledger stays overstated relative to real usable stock, and
the eventual shortfall gets wrongly flagged as theft/spoilage in Phase 5 instead of being recognized
as expected yield loss.

**Convention note:** The BOM's 150 g is treated as a raw-equivalent quantity for *stock deduction*,
while the 1,250/kg effective rate is used for *costing*. This nets out correctly for COGS purposes
(0.30 kg × Rs. 1,250 = 0.375 kg × Rs. 1,000 = Rs. 375) — but it is a deliberate convention that should
be documented, not an accident of the numbers.

---

## Phase 3 — POS Order Execution & Batch Deduction (FIFO Rule)

When a customer orders 2 Chicken Burgers, the POS deducts stock from the batch with the **nearest
expiry date first** (FIFO).

**1. Total Demand**

- Chicken Patty required: 0.15 kg × 2 = **0.30 kg**
- Burger Buns required: 1 × 2 = **2 units**

**2. Batch Allocation Logic (FIFO)**

The system sorts active batches by expiry date ascending, then deducts:

- BAT-CHK-001 (expiry Sept 10) has 5.0 kg remaining — enough to cover the full 0.30 kg demand.
  $$\text{Remaining Qty in BAT-CHK-001} = 5.0 - 0.30 = 4.70\text{ kg}$$

**Split allocation example:** if BAT-CHK-001 had only 0.10 kg left, the system deducts 0.10 kg from
BAT-CHK-001 and the remaining 0.20 kg from BAT-CHK-002 (Split Batch Allocation).

---

## Phase 4 — Financial & Accounting Integration (COGS)

COGS is calculated in real time the moment an order completes.

$$\text{Chicken Cost} = 0.30\text{ kg} \times \text{Rs. 1,250 (effective)} = \text{Rs. 375}$$
$$\text{Buns Cost} = 2 \times \text{Rs. 40} = \text{Rs. 80}$$
$$\text{Cheese Cost} = 2 \times \text{Rs. 30} = \text{Rs. 60}$$
$$\text{Total COGS (2 Burgers)} = 375 + 80 + 60 = \text{Rs. 515}$$

---

## Phase 5 — Day-End Reconciliation & Variance Analysis (Corrected)

At day-end, a physical count is compared against the system's theoretical stock.

$$\text{Theoretical Stock} = (\text{Opening Stock} + \text{GRN Inflow}) - \sum(\text{Sales Deductions}) - \sum(\text{System Waste})$$

**⚠ Fix — scope of comparison:** Variance must be computed at the **item level**, across *all* active
batches of Chicken Patty — not against a single batch in isolation. The original example compared the
physical count only to BAT-CHK-001's remaining 4.70 kg, silently ignoring BAT-CHK-002's untouched
5.0 kg. Batch-level detail is still useful for *attributing* which lot the loss likely came from
(typically the one FIFO was drawing from), but the headline variance figure should use total stock.

**Corrected worked example (item-level):**

| | Chicken Patty |
|---|---|
| BAT-CHK-001 theoretical remaining | 4.70 kg |
| BAT-CHK-002 theoretical remaining | 5.00 kg |
| **Total theoretical stock** | **9.70 kg** |
| Actual physical count | 9.50 kg |
| **Unexplained variance** | **−0.20 kg (200 g loss)** |

$$\text{Financial Loss Value} = 0.20\text{ kg} \times \text{Rs. 1,250 (effective cost)} = \text{Rs. 250}$$

**Variance Categorization:** A 200 g shortfall is logged as one of:
- **Spoilage** (expired/discarded stock)
- **Theft**
- **Over-portioning** (staff using more than the recipe-specified quantity)

— but only *after* confirming it isn't simply unlogged prep wastage (see Phase 2 fix above). Any
trim/prep loss that was properly posted as a `WASTE-PREP` transaction should already be subtracted out
by the theoretical stock formula and must not be double-counted here.
