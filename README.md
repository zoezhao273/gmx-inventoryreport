# GREE México — Inventory Report

A single-file web tool (`index.html`, also delivered as `GREE_Mexico_Inventory_Report.html`) that
turns the Yonyou NC Cloud exports into one screen the sales team can read: how many complete units
are actually sellable today, what is locked for customers, what is on the road between warehouses,
what is still on order — and, for every loose part, who has to follow it up.

Everything runs in the browser. A Google Apps Script + Google Sheet acts as the shared storage, so
finance uploads once and the whole team sees the same numbers.

---

## 1. Who does what

| Role | What they do |
|---|---|
| Sales | Open the page from the link (it carries the read key), read the two views, export to Excel if needed. |
| Finance | Open **Update data**, drop the exports, enter the upload PIN, publish. |

---

## 2. The two views

### Available stock
Component-level stock rolled up to sellable **kits** (indoor + outdoor + panel), per warehouse.

- **Sets per kit** = min(component on hand ÷ BOM qty), calculated **inside each warehouse**.
  Components are never paired across warehouses in the table.
- **TOTAL** = sets/pieces across the selected warehouses. Stock on Hand is the
  *Minus Reservation* export, so TOTAL is already net of locked stock.
- **Loose parts** (bottom of each product line) = on hand but not kittable in that warehouse.
  Hovering the code or the name shows which kits the part belongs to and its partner components.
  Below the loose rows, the **loose-part explanations** say why each piece is loose and who
  follows it up (section 4).
- When a shared component is short in a warehouse, the kits whose main body is **oldest**
  (Stock Aging, FIFO-trimmed to net on hand) are built first; no aging record comes last.
- **Negative on hand** is shown in red and never kitted; the product-line row and the
  product-line pill carry a flag.
- Rows and columns with nothing in them are hidden. Filters: product line, warehouse group
  (Main / DP / RE / Intransit / All), On order toggle, per-column filters for code and name.
- Warehouse order everywhere: GDL > MEX > MTY > CUL > MID, then ascending code; Z-Intransit last.

### Reservations
The Lock Stock Reservation Report grouped by demand document, rolled up to kit level.

- **Expiration = Date Required + 15 days.** Expired / today / ≤7 days are colour-coded; the tab
  shows a badge with the number of expired documents.
- Missing partners are flagged but not deducted. Customer Name is never uploaded or shown.

---

## 3. Column headers with a tooltip

Header tooltips open **instantly anywhere over the header cell** (tap on phones), not only on the `!`.

### Z-Intransit
Explained by the open transfer orders (Transfer Order Execution Query).

- `!` appears when an order needs Logistics, or when Z-Intransit does not reconcile with the
  open orders. The **Intransit** warehouse pill also gets an amber dot, so the alert is visible
  while only Main WH is selected.
- Tooltip format (orders within the normal transit time are not listed):
  ```
  调拨漏入库 / transfer partly received:
  TR-MX-2609-00008
  created 2026-09-28 (9 days)
  GDL-JD → CUL-ACX · 3 of 186 pcs not received
    MXL0007B ×1, MXL0009B ×1, MXL0013A ×1

  GDL-RC → CUL-ACX · 20 of 50 pcs not received
    MXR0002 ×10 sets
  ```
  One block per route (from → to); routes already fully received are left out; complete sets
  are shown as kits. A second section `调拨在途超过10天 / nothing received after 10 days` uses the
  same format.

### Purchase-order columns (On order toggle, off by default)
Each open PO becomes one column after TOTAL (sorted by ETA), plus **TOTAL + on order**.
Quantities are **Unreceived Qty**; each order is kitted on its own (never with stock or another
order). A `!` means **含未成套采购物料** (components not bought as complete sets) and/or
**部分到货** (Unreceived Qty ≠ Total Qty). Rows with TOTAL = 0 but stock on order are shown.

---

## 4. Loose-part attribution

One question per loose piece: **where is its matching part?** Searched in this order; each
step works on what the previous steps left. **Main warehouse** = warehouse code without the
`-DP` / `-RE` suffix (GDL-RC-AME is its own main warehouse). Nothing is ever paired across
main warehouses, and POs never pair across warehouses.

| # | Matching part is… | Explanation shown | Follow-up |
|---|---|---|---|
| 0 | (SMP sample — not matched at all) | Samples without their pair | Marketing / Sales |
| 1 | on an open transfer order to this warehouse | nothing received & ≤ 10 days → **no message**; partly received → *Transfer not fully received*; nothing received after 10 days → *Transfer not received after 10 days* | Logistics |
| 2 | in the same main warehouse's **-DP** | Damaged packaging | Logistics |
| 3 | in the same main warehouse's **-RE** | Product repair | Logistics |
| 4 | between **-DP and -RE** of the same main warehouse | Matching part within the same main warehouse | Logistics |
| 5 | on an open PO shipped to this same warehouse | Can be kitted with goods on order | no action (listed last) |
| 6 | nowhere | stock → *Short of the matching part* | Import and Logistics |
|   |   | Z-Intransit with no matching order → *Loose parts in transit* | Logistics |
|   |   | PO pieces → *Bought unpaired on purchase orders* | Import |

Plus **Partly received purchase orders** → Import (goods may be in without a receipt doc).

Transfer explanations use the same per-route format as the tooltip, two lines per route:
```
TR-MX-2609-00008 · created 2026-09-28 (9 days)
GDL-JD → CUL-ACX · 3 of 186 pcs not received: MXL0007B ×1, MXL0009B ×1, MXL0013A ×1
  pairs with loose stock in CUL-ACX: MXL0005B ×1, MXL0007A ×1, MXL0009A ×1, MXI0049 ×2
```
The "pairs with" line lists the destination-side sub-items (same rows as the Loose parts table).

Design decisions:
- Transfers **not sent in complete sets** are not checked separately: if a transfer really split a
  set, a warehouse ends up with a loose piece and the steps above attribute it.
- Kits compete for shared parts: the kit using the most parts already in stock goes first, then
  kits with more components, then kit code.
- The explanations never change the table; it always shows real stock.

---

## 5. Updating the data

Open **Update data**, enter the PIN, drop each file, check the status line, then **Publish to
cloud**. Each file is parsed and validated locally first; publishing is confirmed by reading the
cloud copy back (file name + row count). **Preview with local files** renders without touching
the cloud.

Upload cards (three per row):

| Row | 1 | 2 | 3 |
|---|---|---|---|
| 1 | Stock on Hand (Minus Reservation) — daily | Transfer Order Execution Query — daily, **export together with SOH** | Lock Stock Reservation Report — daily |
| 2 | Purchase Order list — daily, needs `Unreceived Qty` | Stock Aging Alert Analysis — daily | BOM Checking Report — when it changes |

The Transfer card shows a reconciliation line: in-transit qty of the open orders, summed by
material, must equal **Z-Intransit** in Stock on Hand. A difference usually means the two files
were exported at different times, or the transfer query's date range missed older orders.

### What is sent to the cloud
- **PO:** Order No., Order Date, Ship to, ETD, ETA, Note, Item No., Item Description,
  Unreceived Qty, Total Qty. No supplier, prices, amounts, FX, tax or payment terms.
- **Transfers:** only orders with something still in transit (all their lines): Doc No., Doc Date,
  Previous / New Warehouse, Material Code / Name, Transferred-out, Transferred-in, Intransit Qty.
  Voucher Created By / Approved By are never read.
- **Reservations:** no Customer Name.
- Cloud reads need the **read key** (from the page link `…/#k=KEY`, kept in the browser);
  writes need the **upload PIN**.

---

## 6. Export

**Export .xlsx** writes the current view as filtered (incl. PO columns, `!` markers on PO and
Z-Intransit headers), a **Loose parts notes** sheet (category, follow-up, warehouse / order no.,
detail) and a **Notes** sheet with upload times, rules and flagged orders.

---

## 7. Backend (`Inventory_Report_Code.gs`, maintained by finance)

- One Google Sheet, one tab per dataset: `soh`, `transfer`, `reserv`, `po`, `aging`, `bom` + `meta`.
- To add a dataset, add its name to the dataset list in the script. The **`transfer`** dataset
  must be accepted, and a publish with **0 rows** (no open transfers that day) must be accepted.

---

## 8. Known behaviours / limits

- **Kit materials in Stock on Hand are ignored** (any SKUCODE in column B of the BOM report).
- **Transfer days** are calendar days from Doc Date to today (browser local time); 10-day threshold.
- **Dates from Google Sheets** come back as ISO timestamps and are normalised to `YYYY-MM-DD`.
- **Ship to** must match a warehouse code for a PO column to follow the warehouse filter.
- **Warehouse groups** from the name: `*-DP` → DP, `*-RE` → RE, `Z-INTRANSIT*` → Intransit, else Main.
- Table rows have whole-pixel heights so the frozen SKUCODE / SKUNAME grid lines render evenly.
- The report refuses to render if kits × BOM + loose ≠ on hand; the banner names the first cell.

---

## 9. Quick troubleshooting

| Symptom | Cause / fix |
|---|---|
| Transfer card: "Z-Intransit … differ" | export Stock on Hand and Transfer Orders at the same time; widen the transfer query date range |
| Z-Intransit `!` / amber dot on Intransit | an open transfer is partly received or > 10 days old — hover the Z-Intransit header |
| "the item table has no Unreceived Qty column" | re-export the PO list with that column |
| "Cloud copy did not update" | wrong PIN, or the Apps Script rejected the dataset (check `transfer` is allowed) |
| Reconciliation banner | Stock on Hand and BOM are out of step; re-export both |
