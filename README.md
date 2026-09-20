# GREE México — Inventory Report

A single-file web tool (`GREE_Mexico_Inventory_Report.html`) that turns four Yonyou NC Cloud
exports into one screen the sales team can read: how many complete units are actually
sellable today, what is locked for customers, and what is still on the water.

Everything runs in the browser. A small Google Apps Script + Google Sheet acts as the shared
storage, so finance uploads once and the whole team sees the same numbers.

---

## 1. Who does what

| Role | What they do |
|---|---|
| Sales | Open the page, read the two views, export to Excel if needed. No login. |
| Finance | Open **Update data**, drop the exports, enter the upload PIN, publish. |

---

## 2. The two views

### Available stock
Component-level stock rolled up to sellable **kits** (indoor + outdoor + panel), per warehouse.

- **Sets per kit** = min(component on hand ÷ BOM qty), calculated **inside each warehouse**.
  Components are never paired across warehouses.
- **TOTAL** = sets/pieces across the selected warehouses. Stock on Hand is the
  *Minus Reservation* export, so TOTAL is already net of locked stock.
- **Loose parts** (bottom of each product line) = on hand but not kittable in that warehouse —
  indoor units without outdoor units, spare panels, and so on. Hovering the code or the name
  shows which kits the part belongs to and which partner components are missing.
- When a shared component is short in a warehouse, the kits whose main body is **oldest**
  (Stock Aging, FIFO-trimmed to net on hand) are built first; main bodies with no aging record
  come last.
- Rows and columns with nothing in them are hidden. Filters: product line, warehouse group,
  free-text search, plus per-column filters for code and name.

### Reservations
The Lock Stock Reservation Report grouped by demand document, rolled up to kit level.

- **Expiration = Date Required + 15 days.** Expired / today / ≤7 days are colour-coded, and the
  tab shows a badge with the number of expired documents.
- Each document expands into kit rows (sets) and loose reserved pieces; missing partners are
  flagged but **not** deducted — a reserved indoor unit still counts as a reserved kit.

---

## 3. On order (purchase orders)

The **On order (PO)** button sits under the warehouse filter and is **off by default**. When on,
each open purchase order becomes one column after TOTAL, plus a final **TOTAL + on order**
column.

- Columns are sorted by ETA (no ETA last). The second header line shows `ETA MM-DD · Ship to`.
- Quantities are **Unreceived Qty**, so goods already received are never double counted.
  Orders that are fully received are not published at all.
- PO columns follow the warehouse filter, matched on the order's **Ship to** warehouse.
- Each order is kitted **on its own** — never paired with stock, never with another order.
  Kits are filled biggest-first (most distinct components) so shared outdoor units and panels
  leave as few unmatched pieces as possible.
  - kit row → sets on that order
  - component row (expand a kit) → pieces purchased on that order
  - loose row → pieces bought without their partner
- Rows with **TOTAL = 0 but something on order** are still shown — that is exactly the
  "no stock now, 20 arriving 11 Oct" case sales needs.

### The "!" marker
A `!` after the order number means at least one of:

1. **含未成套采购物料 / components not purchased as complete sets** — after kitting the order,
   BOM components are left over (e.g. 30 outdoor units with no indoor unit).
2. **部分到货 / Unreceived Qty ≠ Total Qty** — part of the order is already in stock.

Hovering the column header shows one tooltip with the order details (ETA, ETD and transit days,
supplier, unreceived / ordered quantity, note) and, below a blank line, the details of whichever
of the two conditions applies.

---

## 4. Updating the data

Open **Update data**, enter the PIN, drop each file, check the green status line, then
**Publish to cloud**. Each file is parsed and validated locally first; publishing is confirmed by
reading the cloud copy back and comparing file name and row count.

| # | Export (Yonyou NC Cloud) | How often | Notes |
|---|---|---|---|
| 1 | Stock on Hand (Minus Reservation) | daily | needs `CATECODE` / `SKUCODE` columns |
| 2 | Purchase Order list | weekly | **must include the `Unreceived Qty` column** |
| 3 | Lock Stock Reservation Report | daily | needs `Demand Doc No.` / `SKUCODE` / `Reserved Qty` |
| 4 | Stock Aging Alert Analysis | daily | needs `Material Code` / `Days Exceeded` |
| 5 | BOM Checking Report | when it changes | kit → sub-item structure |

**Preview with local files** renders the report from the files in your browser without touching
the cloud copy — always use it before publishing something new.

The header shows the upload time of each dataset; Stock on Hand and Reservation are marked
in amber when they are more than 36 hours old.

### What the PO upload sends to the cloud
Only order identity and quantities:

`Order No. · Order Date · Supplier · Ship to · ETD · ETA · Note · Item No. · Item Description · Unreceived Qty · Total Qty`

Unit price, Amount (OC), Amount (FC), FX rate, currency, tax rate, payment terms, subtotal/VAT/total
and the approval block are read for the local checks and then dropped — **no money data ever
leaves the browser**.

---

## 5. Export

**Export .xlsx** writes the current view exactly as filtered, including the PO columns and the
`TOTAL + on order` column. Component rows are grouped as an outline level, and a **Notes** sheet
records the upload times, the rules used, and why each order was flagged with `!`.

---

## 6. Backend

- Storage: one Google Sheet, one tab per dataset, written by a Google Apps Script web app
  (`Inventory_Report_Code.gs`). The script URL is at the top of the app section of the HTML.
- Adding a dataset means adding its name to the dataset list in `Code.gs` — the tool itself
  handles parsing, validation and publishing.
- Writes are protected by the upload PIN; reads are public to anyone with the page.

---

## 7. Known behaviours / limits

- **No cross-PO kitting.** An indoor unit on one order and its outdoor unit on another are each
  reported as unmatched; the `!` tooltip names the codes so it can be checked manually.
- **Kit qty vs pieces.** A kit row counts sets, a loose or spare-part row counts pieces; the
  product-line subtotal mixes the two, exactly as the on-hand columns always have.
- **Dates from Google Sheets.** Sheets converts `2026-07-23` into a real date and returns it as an
  ISO timestamp; the tool normalises every PO and reservation date back to `YYYY-MM-DD`.
- **Ship to must match a warehouse code** (GDL-JD, GDL-RC, MTY-OFFICE, MEX-VIRTUAL, …) for the
  warehouse filter to place a PO column correctly.
- **Warehouse groups** are derived from the name: `*-DP` → DP, `*-RE` → RE, `Z-INTRANSIT*` →
  Intransit, everything else → Main.
- The report refuses to render if component quantities do not reconcile
  (kits × BOM + loose ≠ on hand); the banner names the first failing component/warehouse.
- Browser support: any current Chrome / Edge / Safari. Nothing is stored on the device.

---

## 8. Quick troubleshooting

| Symptom | Cause / fix |
|---|---|
| "the item table has no Unreceived Qty column" | re-export the PO list with that column |
| "every order is fully received" | nothing is open — no PO data is published |
| "Cloud copy did not update" | wrong PIN, or the Apps Script rejected the file |
| PO column missing for an order | its Ship to is outside the selected warehouses, or nothing on it belongs to the selected product lines |
| A column shows `!` but everything looks fine | hover the header — it may be a partial receipt, not a kitting problem |
| Reconciliation banner | Stock on Hand and BOM are out of step; re-export both |
