# Odoo 19 Community — Purchase Developer Training Guide

**Scope:** Odoo 19 Community `purchase` module (`addons/purchase/`) + the
purchase logistics extensions installed under `addons/` (`purchase_stock`,
`purchase_requisition`, `purchase_product_matrix`, `purchase_edi_ubl_bis3`).
This guide covers the default upstream Odoo Purchase app only — no custom
modules.

> **Every menu path in this guide is read from the actual XML in
> `addons/purchase/views/purchase_views.xml` and the menu XML of each module
> listed above.** Nothing here is from memory of older Odoo versions.

**4-part legend** (every menu/action gets all four):

- **Definition** — what it is, explained in plain terms first.
- **Usage** — what it is for / why you'd use it.
- **Config** — what must already exist before you can use it.
- **Workflow** — the ordered, imperative steps to use it.

Menu-path notation: `Top › Sub › Item`.

---

## Table of Contents

0. [Complete Configuration Checklist (do this first)](#0-complete-configuration-checklist-do-this-first)
1. [The Real Odoo 19 Menu Tree](#1-the-real-odoo-19-menu-tree)
2. [Core Models a Developer Must Know](#2-core-models-a-developer-must-know)
3. [Menu / Action Reference (4-part)](#3-menu--action-reference-4-part)

---

## 0. Complete Configuration Checklist (do this first)

Do these in order. Each step unblocks the next. When step 7 is done the whole
purchase app is ready to transact. Purchase depends on the **Accounting** app,
so its vendors, taxes, and payment terms are shared — configure those once and
both apps use them.

```
            PURCHASE SETUP (strict order)

  Company ─► Vendors ─► Products (purchasable)
     │          │              │
     │          │              ▼
     │          │       Control Policy
     │          │      (ordered / received)
     │          │              │
     │          ▼              ▼
     │    Payment Terms ◄── Vendor Pricelists
     │    & Purchase Taxes   (product.supplierinfo)
     │          │              │
     ▼          ▼              ▼
  Currency  Purchase Journal ─► READY TO ORDER
                            (RFQ → PO → Receipt → Bill)
```

### 0.1 Company, Currency & Multi-Company

- **Definition:** The company is the legal entity doing the buying. Every purchase order and vendor bill belongs to one company in one currency. A vendor pricelist, a bill, and a PO must all agree on company and currency or Odoo will convert between them.
- **Usage:** You set this up first because procurement, stock valuation, and accounting all hang off it. If you operate several companies, each gets its own purchase orders and its own vendors (enforced by multi-company rules).
- **Config:** Database + admin user exist; Accounting app installed (Purchase depends on it).
- **Workflow:**
  1. `Settings › Companies › Manage Companies` → open the company → set `currency_id`.
  2. Enable *Multi-Currencies* in Settings if you buy in more than one currency.
  3. `Purchase › Configuration › Settings` → confirm *Multi-Companies* rules are on (they are by default).

### 0.2 Vendors — `res.partner`

- **Definition:** A vendor is any supplier you buy from — a master-data record with address, contacts, and the commercial terms you've agreed (currency, payment terms, buyer responsible). Customers and vendors are the same `res.partner` model, just used differently.
- **Usage:** Every PO and every vendor bill points at a vendor. Setting each vendor up once with the right defaults means POs inherit the currency, payment terms, and deliver address automatically.
- **Config:** Company + currency set.
- **Workflow:**
  1. `Purchase › Orders › Vendors` → *New*.
  2. Set name, address, *Vendor Currency* (`property_purchase_currency_id`), *Payment Terms*.
  3. (Optional) set *Receipt Reminder* + *Days Before Receipt* so Odoo emails the vendor ahead of the promised delivery date.
  4. (Optional) assign a *Buyer* (`buyer_id`) — the user who manages this vendor.

### 0.3 Products (purchasable) & Control Policy

- **Definition:** A product is anything you buy — a storable item, a consumable, or a service. Each purchasable product carries its cost, its purchase taxes, and a **Control Policy** that decides whether you bill it *on ordered* or *on received* quantities.
- **Usage:** When you pick a product on a PO line, Odoo auto-fills the price (from the vendor pricelist), the taxes, and the description. The Control Policy drives the "to invoice" quantity — i.e. whether a bill can be raised before the goods arrive.
- **Config:** Product category + a purchase expense account and purchase tax (set up under Accounting).
- **Workflow:**
  1. `Purchase › Products › Products` → *New*.
  2. Set *Category*, *Cost*, and tick *Can be Purchased*.
  3. Under *Purchase* tab set *Vendor Taxes*, *Control Policy*:
     - **On ordered quantities** — bill for what you ordered (advances, services).
     - **On received quantities** — bill only for what actually arrived (goods).
  4. Add the *Vendors* rows (`seller_ids`) — one per supplier with their price, min qty, and lead time.

```
Control Policy drives "what can I bill right now?"

  On Ordered  →  qty_to_invoice = ordered qty  − already invoiced
  On Received →  qty_to_invoice = received qty − already invoiced
```

### 0.4 Vendor Pricelists — `product.supplierinfo`

- **Definition:** A vendor pricelist entry (one row per vendor per product) records: at what unit price, in what currency, for what minimum quantity, with what delivery lead time, this vendor sells this product. The same model also powers "which vendor do I buy this from?"
- **Usage:** When you add a product to a PO and pick its vendor, Odoo looks up the matching `seller_ids` row by partner + quantity + date + UoM and fills the price and the planned date. Without it, buyers type prices manually and lead times are guessed.
- **Config:** Vendor + product exist.
- **Workflow:**
  1. Either on the product form (*Purchase* tab → *Vendors* rows) **or** via `Purchase › Configuration › Vendor Pricelists`.
  2. Add a row per vendor: *Vendor*, *Price*, *Currency*, *Min Quantity*, *Delivery Lead Time*, *Validity* (start/end dates).

> Confirming a PO also writes the vendor back onto the product as a new
> `seller_ids` row (price + delivery lead time) — so the pricelist learns from
> what you actually agreed.

### 0.5 Payment Terms & Purchase Taxes — shared with Accounting

- **Definition:** A payment term is the agreed schedule for when a vendor gets paid (Net 30, 30% advance, etc.). Purchase taxes are the input taxes (VAT/AIT) you can reclaim on a bill. Both are defined once under Accounting and reused here.
- **Usage:** Set a default payment term on each vendor; POs and bills inherit it. Taxes flow from the product onto PO lines and onto bill lines, where fiscal positions can swap them (e.g. zero-rate an import).
- **Config:** Accounting app configured with payment terms and taxes.
- **Workflow:**
  1. `Accounting › Configuration › Invoicing › Payment Terms` → define schedules.
  2. `Accounting › Configuration › Accounting › Taxes` → define purchase taxes + repartition lines.
  3. `Accounting › Configuration › Accounting › Fiscal Positions` → map source→target tax by vendor location.
  4. Assign the default term on each vendor and the default tax on each product.

### 0.6 Security Groups

- **Definition:** Odoo splits purchase access into two visible roles plus two hidden feature groups. The two visible roles are **User** (raises RFQs, manages day-to-day orders) and **Administrator** (configures settings, sees reporting). The hidden groups switch on purchase warnings and receipt reminders.
- **Usage:** You assign each user the role that matches their responsibility and tick the feature groups you want company-wide. Odoo then shows/hides menus, the Approve button, and the Reports accordingly.
- **Config:** Users exist.
- **Workflow:** `Settings › Manage Users` → open a user → *Access Rights* → assign the Purchase group. Feature toggles live in `Purchase › Configuration › Settings` and flip these groups:

```
Purchase role hierarchy  (base.module_category_supply_chain)
  Purchase User       User            — raise RFQs, manage orders
  Purchase Manager    Administrator   — approve, configure, see reports

hidden feature groups:
  Purchase Warnings                    Partner/product-specific warnings on PO
  Send Receipt Reminders               email vendors before promised date
```

> The double-validation approval gate (§0.7) decides whether a User can
> confirm a PO outright or whether orders above a threshold bounce to a
> Manager for *Approve Order*.

### 0.7 Purchase Settings

- **Definition:** The master switches for the whole Purchase app, grouped under *Orders*, *Invoicing*, and *Logistics*. Each toggle changes how POs are locked, approved, billed, and linked to inventory.
- **Usage:** Come here once, early, to match the procurement policy. Each toggle unlocks a behaviour company-wide.
- **Workflow:** `Purchase › Configuration › Settings` → toggle → *Save*. Key toggles:
  - *Lock Confirmed Orders* — confirmed POs become read-only (the buyer can no longer edit lines after sending).
  - *Order Approval* → *Minimum Amount* — POs at or above this amount land in *To Approve* for a Manager instead of confirming outright (two-step validation).
  - *Warnings* — purchase-specific instructions per vendor/product.
  - *Receipt Reminders* — the daily cron that emails vendors before the promised date.
  - *3-way Matching* — installs `account_3way_match`: a vendor bill line can't be paid until its qty matches both the PO and the receipt.
  - *Purchase Agreements* — installs `purchase_requisition` (blanket orders / call for tenders).
  - *Grid Entry* — installs `purchase_product_matrix` (enter variant POs on a grid).

> Settings 0.1–0.7 done → you can raise RFQs, confirm POs, receive goods,
> receive vendor bills, and pay.

---

## 1. The Real Odoo 19 Menu Tree

Read directly from `purchase_views.xml` and each module's menu XML. The top
menu is `menu_purchase_root`, named **"Purchase"**.

```
Purchase  (menu_purchase_root)  ← top
│
├── Orders                              (menu_procurement_management)
│   ├── RFQ                             (menu_purchase_rfq)            purchase.order  [draft/sent/to approve]
│   ├── Purchase Orders                 (menu_purchase_form_action)    purchase.order  [purchase]
│   ├── Purchase Agreements             (menu_purchase_requisition_pro_mgt)  [purchase_requisition]
│   └── Vendors                         (menu_procurement_management_supplier_name)  res.partner
│
├── Products                            (menu_purchase_products)
│   ├── Products                        (menu_procurement_partner_contact_form)  product.template
│   └── Product Variants                (product_product_menu)                    [product variants]
│
├── Reporting                           (purchase_report_main)                   [Manager]
│   └── Purchase                        (purchase_report)  purchase.report — Purchase Analysis (graph/pivot)
│
└── Configuration                       (menu_purchase_config)                   [Manager]
    ├── Settings                        (menu_purchase_general_settings)         [system]
    ├── Vendor Pricelists               (menu_product_pricelist_action2_purchase)  product.supplierinfo
    └── Products                        (menu_product_in_config_purchase)
        ├── Attributes                  (menu_product_attribute_action)          [product variants]
        ├── Product Categories          (menu_product_category_config_purchase)
        └── Units & Packagings          (menu_purchase_uom_form_action)           [uom]
```

**Memorise this picture.** The common errors from pre-17 memory:

- **RFQ and Purchase Orders are the same model** (`purchase.order`), split into
  two menus by state. *RFQ* shows draft/sent/to-approve; *Purchase Orders*
  shows confirmed (`state='purchase'`). There is no separate "Quotation" model.
- There is **no** standalone "Vendor Bills" menu under Purchase — vendor bills
  live under **Accounting › Vendors › Bills** (they are `account.move` with
  `move_type='in_invoice'`). Purchase links them back to PO lines.
- *Purchase Agreements* only appears once the **Purchase Agreements** setting
  installs `purchase_requisition`.

---

## 2. Core Models a Developer Must Know

Think of Odoo Purchase as one **order engine** that plugs into Accounting for
the bill and into Inventory for the receipt. Behind every screen is a small set
of tables, and the extensions only add to these.

```
                    ┌────────────────────────┐
                    │    purchase.order       │  the order header — the RFQ/PO
                    │  state: draft|sent|to approve|
                    │         purchase|cancel
                    │  partner_id (Vendor), currency_id, company_id,
                    │  date_order, date_approve, date_planned,
                    │  amount_untaxed/tax/total, fiscal_position_id,
                    │  payment_term_id, user_id (Buyer), origin
                    └───────────┬────────────┘
                                │ One2many → order_line
                                ▼
                    ┌────────────────────────┐
                    │   purchase.order.line   │  one bought product line
                    │  product_id, product_qty, product_uom_id,
                    │  price_unit, discount, tax_ids,
                    │  date_planned, analytic_distribution,
                    │  qty_received / qty_invoiced / qty_to_invoice,
                    │  invoice_lines  ← back-link to bill lines
                    └───────────┬────────────┘
                       billed via (purchase_line_id)
                                ▼
                    ┌────────────────────────┐
                    │  account.move(line)     │  the vendor bill — in_invoice
                    │  move_type='in_invoice',
                    │  partner_id, invoice_date, amount_*,
                    │  line.purchase_line_id  ← which PO line it settles
                    └────────────────────────┘

       product.supplierinfo  → vendor pricelist row (price/lead time/min qty)
       res.partner           → vendor master + supplier currency/terms
       purchase.bill.union   → virtual union of draft bills + POs (the "add to PO" picker)
       purchase.bill.line.match → SQL view matching open PO lines to bill lines
```

**Rule:** a purchase order IS a `purchase.order`. Its lines are
`purchase.order.line`, which carry the three quantities that govern the whole
procure-to-pay loop: **ordered, received, invoiced**. A vendor bill is just an
`account.move` whose lines point back at PO lines through `purchase_line_id` —
that single field is what lets Odoo know "this bill settles that order". 90% of
purchase customizations inherit `purchase.order`, `purchase.order.line`, or the
`account.move` bridge.

---

## 3. Menu / Action Reference (4-part)

### 3.1 Orders

#### RFQ — Request for Quotation

- **Definition:** An RFQ is the document you send to a vendor to *ask* for a price and availability before you commit to buying. It is a `purchase.order` in `draft` or `sent` state — the negotiation phase of the same record that will later become the Purchase Order.
- **Usage:** This is where procurement starts. You raise an RFQ per vendor (or several, to compare), send it, receive their quote back, then either confirm it into a PO or discard it. Until it's confirmed, nothing is committed and no stock or accounting is affected.
- **Config:** Vendor, product(s) with vendor pricelist rows, a buyer.
- **Workflow:**
  1. `Purchase › Orders › RFQ` → *New*.
  2. Pick the vendor → currency, payment terms, delivery address auto-fill.
  3. Add lines (product / qty / planned date); price + lead time come from the vendor pricelist.
  4. *Send RFQ* (emails the vendor; state → `sent`).
  5. When the quote is agreed → *Confirm Order*.
     - If the total is below the approval threshold (or no threshold) → state `purchase` (a real PO).
     - If at/above the threshold → state `to approve`, waiting for a Manager.

```
   RFQ life-cycle (one purchase.order record, changing state)
   ═════════════════════════════════════════════════════════

   [draft]  ──Send RFQ──►  [sent]  ──Confirm──►  [to approve]*
                                                    │
                                                    │  Approve Order (Manager)
                                                    ▼
                                                 [purchase]
                                                    │
                              Set to Draft ◄─────────┘
                              Cancel (any state)
   * only when the total ≥ the Order Approval minimum
```

#### Purchase Orders

- **Definition:** A Purchase Order is a confirmed, binding `purchase.order` (`state='purchase'`). It's the contract: "we agree to buy these quantities at these prices, delivered by this date." This menu is pre-filtered to confirmed orders only.
- **Usage:** Once confirmed, the PO drives the rest of the cycle: it generates the receipt (with `purchase_stock`), it sets the quantities you can bill, and it locks against further edits when *Lock Confirmed Orders* is on. You track receipt and invoicing progress here through the `invoice_status` and on-hand stat buttons.
- **Config:** A confirmed RFQ (or raise and confirm directly).
- **Workflow:**
  1. `Purchase › Orders › Purchase Orders`.
  2. Open an order → *Receive Products* (or *Receipts* stat button) opens the incoming transfer.
  3. As goods arrive, validate the receipt → `qty_received` on each line climbs.
  4. *Create Bill* → generates a draft vendor bill (`account.move`, `in_invoice`) pre-filled from the PO lines; each bill line links back via `purchase_line_id`.
  5. `invoice_status` moves `no → to invoice → invoiced` as lines are billed.
  6. *Lock* (if enabled) freezes the order; a Manager can *Unlock* to correct a mistake.

#### Purchase Agreements — `purchase_requisition`

- **Definition:** A Purchase Agreement (or *blanket order* / *call for tenders*) is a negotiated container for a period: "for the next year, we'll buy up to X units of product Y at price Z from this vendor." You then draw down by raising ordinary POs against it.
- **Usage:** For recurring or high-volume buying where you've negotiated terms upfront but don't want one giant PO. Each call-off PO pulls from the agreement's quantity and inherits its price, so the negotiated rate is enforced automatically.
- **Config:** *Purchase Agreements* enabled in Settings; a vendor + product(s).
- **Workflow:**
  1. `Purchase › Orders › Purchase Agreements` → *New*.
  2. Choose *Agreement Type*: *Blanket Order* (quantity-based, with negotiated price) or *Call for Tenders* (select the winning bid among several vendors).
  3. Add lines (product / quantity / price). *Confirm*.
  4. *New Quotation* on the agreement raises a normal PO that draws down the agreed quantity.

#### Vendors (partners)

- **Definition:** Your supplier master list — name, address, contacts, supplier currency, payment terms, and buyer. Same `res.partner` model as customers.
- **Usage:** Set each vendor up once with the right defaults; every RFQ, PO, and bill afterwards just picks the vendor and inherits everything.
- **Config:** None.
- **Workflow:** `Purchase › Orders › Vendors` → *New* → fill supplier currency + payment terms + buyer → *Save*. The *Purchases* stat button shows the count of that vendor's POs.

### 3.2 Products

#### Products (purchasable)

- **Definition:** The catalogue of things you buy — each carrying its cost, purchase taxes, Control Policy, and vendor list (`seller_ids`).
- **Usage:** Pick a product on a PO line and the price, taxes, description, and planned date fill in automatically from the vendor pricelist. Maintaining the catalogue keeps buying fast, consistent, and auditable.
- **Config:** Product category + a purchase expense account and tax.
- **Workflow:** `Purchase › Products › Products` → *New* → set *Category*, *Cost*, *Can be Purchased*, *Vendor Taxes*, *Control Policy*, and add the *Vendors* rows. The *Purchased* stat button shows the quantity bought in the last year.

#### Product Variants

- **Definition:** Same product template, different attribute combinations (size, colour) — each variant is its own `product.product` and is bought and stocked separately.
- **Usage:** Only visible with *Product Variants* enabled. Use when you buy the same item from one vendor in several distinguishable forms.
- **Workflow:** `Purchase › Products › Product Variants`.

### 3.3 Reporting

#### Purchase (Purchase Analysis)

- **Definition:** A pivot/graph view over `purchase.report` — every confirmed PO line, with its quantities, prices, partner, company, and dates, slicable any way the business asks.
- **Usage:** Management reporting without Excel — spend by vendor, by product category, by month, by buyer; negotiation performance (negotiated price vs average); vendor delivery performance.
- **Config:** Confirmed purchase orders exist.
- **Workflow:** `Purchase › Reporting › Purchase` → pivot by *Vendor* / *Product Category* / *Order Month*, swap measures (*Count* / *Total* / *Untaxed Total*).

### 3.4 Configuration

#### Settings

- **Definition:** The master switches for the Purchase app — covered in full in §0.7.
- **Usage:** Come here to match the procurement policy company-wide: locking, approval thresholds, warnings, reminders, 3-way matching, agreements, grid entry.
- **Workflow:** `Purchase › Configuration › Settings` → toggle → *Save*.

#### Vendor Pricelists — `product.supplierinfo`

- **Definition:** The full list of vendor-per-product rows — unit price, currency, minimum quantity, delivery lead time, validity dates — that drive auto-filled PO lines.
- **Usage:** The procurement master-data screen. Maintain prices and lead times here in bulk (or per product on the product form) so buyers never type a price manually.
- **Config:** Vendors + products exist.
- **Workflow:** `Purchase › Configuration › Vendor Pricelists` → *New* → *Vendor* / *Product* / *Price* / *Currency* / *Min Quantity* / *Delivery Lead Time* / *Validity*.

#### Products (Attributes, Product Categories, Units & Packagings)

- **Definition:** The product master-data menus shared across apps. *Attributes* define variant axes (needs *Product Variants* on). *Product Categories* carry the purchase accounts and default purchase taxes. *Units & Packagings* defines the UoMs and their conversions (needs the UoM group).
- **Usage:** Set these up once; products inherit them. A product with the right category inherits the correct expense account on every PO line.
- **Workflow:** `Purchase › Configuration › Products › …`.

---

### 3.5 The Purchase ↔ Vendor Bill Bridge

This is the purchase-specific "deep" workflow — the equivalent of bank
reconciliation for procurement. It is what makes a PO not just an email but a
controlled, auditable transaction. Read this once and the rest of the app makes
sense.

```
   Procure-to-Pay loop (one PO, tracked end to end)
   ════════════════════════════════════════════════

   RFQ  ──Confirm──►  Purchase Order (purchase.order, state='purchase')
                            │
                            │  (purchase_stock) auto-generates a Receipt
                            ▼
                      stock.picking (incoming)  ──Validate──►  qty_received ↑
                            │
                            │  Create Bill  →  action_create_invoice
                            ▼
                      account.move (in_invoice)  ──draft bill
                            │   each bill line → purchase_line_id
                            ▼
                      qty_invoiced ↑   invoice_status: to invoice → invoiced
                            │
                            ▼
                      Confirm + Register Payment  (in Accounting › Vendors)
```

**The single field that ties it all together** is `purchase_line_id` on the
vendor-bill line (`account.move.line`). It points back at the `purchase.order.line`
the bill is settling. From it, Odoo computes:

- **qty_invoiced** on the PO line — how much has been billed so far.
- **qty_to_invoice** — what *can* be billed right now, per the product's
  Control Policy (ordered − invoiced, or received − invoiced).
- **invoice_status** on the PO — `no` / `to invoice` / `invoiced`.

#### Vendor Bill → PO auto-matching (OCR / EDI)

When a vendor's PDF or EDI message arrives, Odoo can match its lines to open PO
lines automatically so the buyer doesn't link them by hand:

```
   Vendor PDF / EDI bill
        │
        │  Send to OCR  /  Import EDI
        ▼
   draft account.move (in_invoice)
        │
        │  Find matching PO:
        │    • try a full total match against one PO
        │    • else try a subset total match (Knapsack over PO lines)
        │    • else match line-by-line (product + qty + price + name)
        ▼
   bill lines auto-linked via purchase_line_id
   invoice_origin  ← PO name(s)
   *Purchase Order* smart button appears on the bill
```

This is why *every* bill line that should settle a PO must carry a
`purchase_line_id` — it is the audit trail that answers "which order does this
charge belong to?".

#### Manual bill-to-PO matching

When auto-match can't decide (a vendor mixes products from several POs, or sends
a combined bill), the **Bill Matching** screen (`action_bill_matching` from a
bill, or the *Purchase* smart button) opens a side-by-side view of open PO lines
and unmatched bill lines:

- **Definition:** A read-only SQL view (`purchase.bill.line.match`) that shows,
  on one screen, every PO line still to be billed and every bill line not yet
  tied to a PO, per partner and company.
- **Usage:** The AP clerk's matching workbench. You pick the bill line, pick the
  PO line(s) it settles, adjust quantity/price if the vendor overcharged, and
  *Match* — the link is written and `qty_invoiced` updates.
- **Workflow:** Open a draft vendor bill → *Bill Matching* (or *Purchase* on the
  PO) → match products → *Add to PO* / *Create Bill* / *Match Lines*.

```
   Bill matching screen (purchase.bill.line.match — SQL view, UNION ALL)
   ═════════════════════════════════════════════════════════════════════
   Left side:  open PO lines   (state='purchase', qty_to_invoice > 0)
   Right side: open bill lines (in_invoice, draft/posted, no purchase_line_id)
                │
                ▼  Match by product (one PO line per product; leftover bill
                   lines deleted; remaining PO lines rolled into the bill)
```

---

## Appendix — Where to Look in Source

| Need                                                  | File (path under repo root)                                |
| ----------------------------------------------------- | ---------------------------------------------------------- |
| The core menu tree                                    | `addons/purchase/views/purchase_views.xml`                 |
| PO form, header buttons, state bar, server actions   | `addons/purchase/views/purchase_views.xml`                 |
| RFQ + Purchase Orders actions (split by state)        | `addons/purchase/views/purchase_views.xml`                 |
| Settings form                                         | `addons/purchase/views/res_config_settings_views.xml`      |
| How a PO confirms / approves / locks                  | `addons/purchase/models/purchase_order.py`                 |
| PO line quantities (received / invoiced / to invoice) | `addons/purchase/models/purchase_order_line.py`            |
| Vendor-bill ↔ PO bridge (`purchase_line_id`)          | `addons/purchase/models/account_invoice.py`                |
| Bill-to-PO auto-matching (Knapsack)                   | `addons/purchase/models/account_invoice.py`                |
| Bill-matching SQL view (open PO × bill lines)         | `addons/purchase/models/purchase_bill_line_match.py`       |
| Vendor pricelist (`product.supplierinfo`)             | `addons/purchase/models/product.py`                        |
| Vendor master fields (supplier currency, reminders)   | `addons/purchase/models/res_partner.py`                    |
| Settings fields (lock, double validation, toggles)    | `addons/purchase/models/res_config_settings.py`            |
| PO sequence + mail subtypes + reminder cron           | `addons/purchase/data/purchase_data.xml`, `data/ir_cron_data.xml` |
| Add-downpayment / add-products wizard                 | `addons/purchase/wizard/bill_to_po_wizard.py`              |
| Purchase Analysis report                              | `addons/purchase/report/purchase_report.py`                |
| RFQ / PO print templates                              | `addons/purchase/report/purchase_quotation_templates.xml`, `purchase_order_templates.xml` |
| PO → stock receipt linkage + drop-ship                | `addons/purchase_stock/models/purchase_order.py`           |
| Purchase Agreements (blanket orders)                  | `addons/purchase_requisition/models/purchase_requisition.py` |
| Variant grid PO entry                                 | `addons/purchase_product_matrix/`                          |
| EDI (BIS3) import/export                              | `addons/purchase_edi_ubl_bis3/`                            |
| Security groups & multi-company rules                 | `addons/purchase/security/purchase_security.xml`           |

---

*Menu paths in this guide are verified against the installed XML on branch
`om_mps_rebuild`. If the purchase stack is upgraded, re-read `purchase_views.xml`
and each module's menu XML — the path labels are the thing most likely to shift
between versions.*
