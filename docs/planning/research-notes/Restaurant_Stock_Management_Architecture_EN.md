# Restaurant Stock & Inventory Management System Architecture

## 1. Core Architecture & Concepts

| Concept | Description & Real-World Logic | System Implementation |
| :--- | :--- | :--- |
| **Flexible UOM System** | The same material may be used in different units — grams, pieces, or full units (e.g., whole chicken, full roast, portion, drumstick). | Each material has a **Base UOM** (e.g., kg/g), with **Secondary UOM conversions** or alternative-unit links defined against it. |
| **Material Breakdown** | A single master raw material is cleaned or cut into sub-parts (butcher test / disassembly). | When the master item is broken down, sub-parts are added to stock either by a **standard percentage** or via **manual entry**. |
| **Yield & Wastage** | Weight loss that occurs during cleaning/trimming of raw materials (yield %), and the leftover waste (trimmings). | Conversion rates are set at the recipe/BOM level, and actual wastage is recorded in a **waste log** or allocated to cost. |
| **Hybrid Products** | The same item can be used directly as a menu item, or as an add-on/sub-recipe within another menu item. | Items are marked as **Hybrid / Semi-Finished**, enabling **multi-level BOMs** (nested recipes). |
| **Prep Batches** | By-products (bones, trimmings) are combined to make base stocks/broths for later use. | Raw stock is transferred into **Base Prep Stock**, which is then deducted directly when used in main orders. |
| **Scrap Sales** | Unused parts (skin, offal, bones) are sold externally without a fixed price. | Sold via a **Dynamic Scrap Sale** option (no fixed price) and recorded as "Other Income." |

---

## 2. End-to-End Inventory Workflows

### A. Goods Receiving
1. **Invoice vs. Physical Check:** Verify quantity, weight, and price against the invoice.
2. **Discrepancy / Shortfall:** If there is a shortfall between the invoice and the goods received, obtain a **Credit Note**, and record **only the quantity actually received** in the GRN (Goods Received Note).
3. **Storage & FIFO:** Send goods to store on a **First-In, First-Out** basis so that earlier stock is used first.

---

### B. Raw Material Processing & Conversion Workflow

```
Whole Chicken (Master Stock: 10 kg)
          │
          ├── [Butcher Test / Auto-Breakdown]
          │
          ├──► Chicken Breast (3.0 kg) ───► Linked to Burger BOM
          ├──► Drumsticks/Wings (5.0 kg) ──► Linked to Plate BOM (Unit/Pcs)
          ├──► Bones/Neck (1.0 kg) ───────► Converted to Chicken Broth (Base Prep)
          └──► Offal/Skin (1.0 kg) ───────► Scrap Sale (Dynamic Price) OR Waste Log
```

*(Note: 3.0 + 5.0 + 1.0 + 1.0 = 10 kg — the breakdown correctly reconciles with the 10 kg master stock.)*

---

### C. Kitchen Consumption & POS Deductions

```
               ┌──► Direct Sale Order ───► Deducts Finished BOM (e.g., Chop Suey Portion)
[POS Order] ──┤
               └──► Hybrid Add-on Order ─► Deducts Main BOM + Sub-Recipe Portion
```

* **Weight-Based (g/kg):** For items like rice and curries, the relevant gram quantity is deducted directly.
* **Piece-Based (Count):** Items like drumsticks/wings are deducted by piece count, with an average weight applied in the background to cap the deduction.

---

### D. Audit, Reconciliation & Variance Analysis

When performing a physical stock audit (weekly or monthly):

**Theoretical Stock Calculation:**
$$\text{Theoretical Stock} = \text{Opening Stock} + \text{GRN Purchases} - \text{POS Deductions}$$

**Variance Calculation:**
$$\text{Variance} = \text{Theoretical Stock} - \text{Actual Stock}$$

#### How to Investigate the Cause of Variance:
1. **Wastage:** Check the waste log book.
2. **Over-Portioning:** Catch chef errors through weekly **spot weighing checks** (randomly weighing portions).
3. **Theft:** Any unexplained shortage remaining after accounting for waste logs and portion audits is identified as theft.

---

## 3. System Data Architecture & Logic Structure

```
[Master Material: Whole Chicken]
   │
   ├── Primary UOM: Kg
   ├── Alternative UOMs: Pcs, Portions
   │
   ├── [Sub-Parts Mapping]
   │      ├── Part A: Breast (UOM: g)
   │      ├── Part B: Drumstick (UOM: Pcs)
   │      └── Part C: Bones (UOM: Kg)
   │
   └── [Recipe / BOM Levels]
          ├── Recipe 1: Burger (Uses Part A)
          ├── Recipe 2: Fried Rice + Chop Suey (Uses Hybrid Sub-Recipe)
          └── Base Prep: Stock Broth (Uses Part C)
```
