# Odoo 19 Community — Inventory Developer Training Guide

**Scope:** Odoo 19 Community `stock` module (`addons/stock/`) + the inventory
valuation / logistics stack installed under `addons/` (`stock_account`,
`stock_landed_costs`, `stock_picking_batch`, `stock_dropshipping`,
`stock_delivery`). This guide covers the default upstream Odoo inventory app
only — no custom modules.

> **Every menu path in this guide is read from the actual XML in
> `addons/stock/views/stock_menu_views.xml` and the menu XML of each module
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

Do these in order. Each step unblocks the next. When step 10 is done the whole
inventory module is ready to transact.

```
            INVENTORY SETUP (strict order)

  Company ─► Warehouse ─► Locations ─► Operation Types
     │           │             │              │
     │           │             │              ▼
     │           │             │      Routes + Rules
     │           ▼             │       (push/pull/MTO)
     │    Reception/Delivery   │              │
     │       Steps (1/2/3)     │              ▼
     │           │             │       Putaway / Storage
     ▼           ▼             ▼              Categories
  Products ──────────────► Product Categories
  (is_storable,             (cost method if
   tracking, UoM)           stock_account)
     │
     ├──► Lots / Serial Numbers  (if tracking != none)
     ├──► Packages / Package Types
     └──► Reordering Rules (min/max)
                              │
                              ▼
                   READY TO TRANSFER (Receipts / Deliveries / Internal)
```

### 0.1 Company & Warehouse

- **Definition:** The company is the legal entity whose stock you own. A warehouse is a physical place where that company stores goods — each warehouse gets its own location tree, operation types, and sequences, and (with `stock_account`) its own valuation.
- **Usage:** You set these up first because nothing else has a "where". Every product, every unit of stock, every move belongs to one warehouse's location tree inside one company.
- **Config:** Database + admin user exist.
- **Workflow:**
  1. `Inventory › Configuration › Warehouse Management › Warehouses` → New.
  2. Set `name` + `code` (≤5 chars, unique per company). Odoo auto-creates the view location, the Stock location, and all operation types (Receipts, Deliveries, Pick, Pack, Quality Control, Storage, Internal, Cross-Dock) plus their sequences.
  3. Set *Receipt Steps* (1/2/3-step incoming) and *Delivery Steps* (1/2/3-step outgoing) — these switch the extra Input / QC / Output / Packing locations on or off and rebuild the routes.
  4. Creating a 2nd warehouse auto-enables *Manage Multiple Warehouses* + *Manage Multiple Stock Locations* on every internal user.

### 0.2 Locations — `stock.location`

- **Definition:** A location is any place stock can be — a whole warehouse (view), a bin/rack (internal), a customer site, a vendor's yard, the scrap pile, a transit zone, or a production line. Locations nest in a tree, so "child of" queries are a single fast lookup.
- **Usage:** Every move has a source and a destination location. The `usage` field is the rule-maker: `internal` is on-hand, `customer`/`supplier` are outside the books, `transit` is the inter-company in-between, `inventory` is the adjustment bucket, `view` is a non-physical folder.
- **Config:** Warehouse exists.
- **Workflow:**
  1. Enable `Inventory › Configuration › Settings › Warehouse › Store products in specific locations`.
  2. `Inventory › Configuration › Warehouse Management › Locations` → New.
  3. Set `name`, `location_id` (parent), `usage`. Set `barcode` for scanning.
  4. (Optional) set a *Removal Strategy* (FIFO/LIFO/FEFO/Closest/Least Packages) and a *Storage Category* per location.

```
stock.location usage values and what they mean:
  supplier   vendor's premises (stock enters here)
  view       non-physical parent (warehouse root, zone folder)
  internal   real, owned stock — counted as on-hand
  customer   customer's site (stock leaves here)
  inventory  the "Inventory Loss" adjustment bucket
  production a manufacturing work centre
  transit    inter-company transfer holding area
```

### 0.3 Operation Types — `stock.picking.type`

- **Definition:** An operation type is a template for one kind of transfer — "Receipts", "Delivery Orders", "Internal Transfers", "Pick", "Pack", "Quality Control", "Storage", "Cross-Dock". It carries the source/destination locations, the sequence that numbers every transfer of this type, the lot-creation policy, the reservation method, and the backorder policy.
- **Usage:** Every transfer points at one operation type. The type's *Code* (`incoming`/`outgoing`/`internal`) drives the form layout, the lot fields, the default locations, and which menu the transfer lands in. They are auto-created per warehouse but you can add custom ones.
- **Config:** Warehouse + locations exist.
- **Workflow:** `Inventory › Configuration › Warehouse Management › Operations Types` → open one and adjust:
  - *Default Source / Destination Location*.
  - *Reservation Method* (`at_confirm` / `manual` / `by_date`) + *Reservation Days Before*.
  - *Create Backorder* (`ask` / `always` / `never`).
  - *Create New Lots* / *Use Existing Lots* (whether to type new lot names or pick existing ones).
  - *Sequence Prefix* (e.g. `IN`, `OUT`, `WH/INT`) — changing it rewrites the prefix.

### 0.4 Routes & Rules — `stock.route` + `stock.rule`

- **Definition:** A **route** is a named collection of **rules** that moves a product through the warehouse. A **rule** is one step — "pull from Stock to Output (Pick)", "push from Input to Stock (Store)", "pull from Vendor (Buy)", "pull & push". Each rule has an action (pull/push/pull_push), a procure method (make_to_stock / make_to_order), a source location, a destination location, and the picking type it should spawn.
- **Usage:** When you confirm a sales order, procurement asks each product's routes for a rule that can satisfy the demand; the rule spawns the right moves chained through the warehouse. Routes are selectable on products, product categories, package types, or whole warehouses.
- **Config:** Locations + operation types exist (the warehouse already created default routes for you).
- **Workflow:**
  1. Enable `Inventory › Configuration › Settings › Warehouse › Use your own routes`.
  2. `Inventory › Configuration › Warehouse Management › Routes` → open a route, inspect its rules.
  3. `Inventory › Configuration › Warehouse Management › Rules` to add/edit — set *Action*, *Operation Type*, *Source / Destination Location*, *Procure Method*, *Propagate Cancel / Carrier*.

```
rule action        meaning
  pull         "Pull From"  — take from src, spawn a picking to dest
  push         "Push To"    — after arrival, auto-chain forward to dest
  pull_push    both         — pull then push (a full corridor)
  buy          (purchase_stock) — generate a purchase order
  manufacture  (mrp)        — generate a manufacturing order

rule procure method
  make_to_stock      Take From Stock (reserve if available)
  make_to_order      Trigger Another Rule upstream
  mts_else_mto       Try stock first, fall back to MTO
```

### 0.5 Products — `product.template` / `product.product`

- **Definition:** A product is anything you stock or move. In inventory only *Storable* products are quantity-tracked — consumables and services flow through moves but don't keep an on-hand balance.
- **Usage:** Every move points at a product; the product's *Tracking*, *Type*, *Routes*, and *Unit of Measure* drive how the move is reserved, valued, and numbered. Set these correctly once and day-to-day moves just inherit them.
- **Config:** Product category + UoM exist; (with `stock_account`) the category has a cost method and valuation accounts.
- **Workflow:**
  1. `Inventory › Products › Products` → New.
  2. Tick *Storable Product*, set *Category*, *Sale Price*, *Cost*.
  3. Set *Tracking* (`none` / `By Lots` / `By Unique Serial Number`) — only visible once *Traceability › Lots & Serial Numbers* is enabled.
  4. *Inventory* tab → set *Routes* (e.g. "Buy" or "Manufacture") and the *Description on Receptions / Deliveries*.
  5. (Optional) set weight/volume (used for putaway and shipping connectors).

### 0.6 Lots / Serial Numbers — `stock.lot`

- **Definition:** A lot (or serial number — same model, one-per-unit when tracking is "By Unique Serial Number") is a unique batch tag stuck on a physical quantity of a tracked product so you can prove where it came from and where it went.
- **Usage:** Required whenever a product has tracking on. Every executed move line for that product points at a lot; the lot's *On Hand* quantity shows you where those units currently sit; the traceability report walks the full upstream/downstream chain.
- **Config:** *Traceability* enabled in settings; product tracking set.
- **Workflow:** `Inventory › Products › Lots/Serial Numbers` → New → `name`, `product_id`, `ref` (optional). Lots can also be created inline on a receipt.

### 0.7 Packages & Package Types — `stock.package` / `stock.package.type`

- **Definition:** A **package** is a physical container (box, pallet, tote) holding one or more units. A **package type** is a template for that container (dimensions, max weight, barcode sequence, route). Packages nest (a pallet holds boxes).
- **Usage:** Lets you *Put in Pack* on a picking so a whole box is scanned/moved/labelled as a unit. Needed for shipping connectors (which print one label per package) and for multi-step pick → pack → ship flows.
- **Config:** *Operations › Put products in packs* enabled.
- **Workflow:**
  1. `Inventory › Configuration › Delivery › Package Types` → define types (e.g. "Small Box", "Pallet").
  2. Packages themselves are created on the fly via *Put in Pack* on a transfer, or from `Inventory › Products › Packages`.

### 0.8 Reordering Rules — `stock.warehouse.orderpoint`

- **Definition:** A reordering rule is the "when this product drops below X in this location, order up to Y" trigger. It's the engine behind automatic replenishment.
- **Usage:** Each rule has a *Trigger* (`auto` = checked by the daily scheduler, `manual` = appears in the Replenishment list until you act). When forecast stock falls under *Min Quantity*, the scheduler generates the procurement (a draft PO for "Buy", a draft MO for "Manufacture", or a stock move) to bring it back to *Max Quantity*.
- **Config:** Product + route that can fulfil the order (Buy/Manufacture).
- **Workflow:**
  1. `Inventory › Operations › Procurement › Replenishment` (manual list) or open a product → *Reordering Rules* → New.
  2. Set *Product*, *Location*, *Min / Max Quantity*, *Quantity Multiple* (rounding), *Trigger*, *Route* (optional override).
  3. The daily scheduler processes `auto` rules; `manual` rules are actioned from the Replenishment screen.

### 0.9 Putaway Rules & Storage Categories

- **Definition:** A **putaway rule** answers "when product X arrives at location Y, where should it actually go?" A **storage category** is a capacity/mixing policy on a location ("empty only", "same products only", "mixed") with optional weight limits and per-product/per-package-type capacity rows.
- **Usage:** Together they automate bin assignment — a receipt validates and the move lines automatically point at the right rack instead of a generic Stock location, respecting capacity.
- **Config:** Multiple locations enabled.
- **Workflow:**
  1. `Inventory › Configuration › Warehouse Management › Storage Categories` → define categories + *Allow New Products* policy + capacity rows.
  2. `Inventory › Configuration › Warehouse Management › Putaway Rules` → New → *Arrival Location* + *Destination Location* + optional *Product* / *Category* / *Package Types*.
  3. Assign the storage category on the destination location's form.

### 0.10 Security Groups

- **Definition:** Odoo splits inventory access into a visible role hierarchy plus a set of feature-toggle groups. The two visible roles are **User** (does day-to-day transfers) and **Administrator** (configures warehouses, routes, sees reporting). The hidden groups each switch on one feature.
- **Usage:** You assign each user the role that matches their responsibility and tick the feature groups you want company-wide. Odoo then shows/hides menus, locations, and lot fields accordingly.
- **Config:** Users exist.
- **Workflow:** `Settings › Manage Users` → open a user → *Access Rights* → assign the Inventory group. Feature toggles live in `Inventory › Configuration › Settings` and flip these groups:

```
Inventory role hierarchy  (base.module_category_supply_chain)
  User            — work on transfers
  Administrator   — configure + see reports

hidden feature groups (each unlocks a menu/field set):
  Manage Multiple Stock Locations
  Manage Multiple Warehouses
  Manage Lots / Serial Numbers
  Print GS1 Barcodes for Lot & Serial
  Display Serial & Lot on Delivery Slips
  Manage Packages
  Manage Push and Pull inventory flows
  Manage Different Stock Owners
  Partner-Specific Instructions
  Require a signature on delivery orders
  Use Reception Report
```

> Steps 0.1–0.10 done → you can receive, store, pick, pack, ship, scrap,
> reorder, value, and trace.

---

## 1. The Real Odoo 19 Menu Tree

Read directly from `stock_menu_views.xml` and every module's menu XML. The top
menu is `menu_stock_root`, named **"Inventory"**.

```
Inventory  (menu_stock_root)  ← top
│
├── Operations                          (menu_stock_warehouse_mgmt)
│   ├── Transfers                       (menu_stock_transfers)
│   │   ├── Receipts                    incoming  (in_picking)
│   │   ├── Deliveries                  outgoing  (out_picking)
│   │   └── Internal                    internal  (int_picking)  [multi-locations]
│   │
│   ├── Adjustments                     (menu_stock_adjustments)
│   │   ├── Physical Inventory          (menu_action_inventory_tree)  stock.quant
│   │   ├── Scrap                       (menu_stock_scrap)            stock.scrap
│   │   └── Landed Costs                [stock_landed_costs]          stock.landed.cost
│   │
│   ├── Procurement                     (menu_stock_procurement)
│   │   ├── Replenishment               (menu_reordering_rules_replenish) [Manager]
│   │   └── References                  (menu_stock_references)           [technical]
│   │
│   ├── Jobs                            [stock_picking_batch]
│   │   ├── Batch Transfers             stock.picking.batch
│   │   └── Wave Transfers              stock.picking.batch (is_wave)
│   └── Run Scheduler                   (menu_procurement_compute)        [technical]
│
├── Products                            (menu_stock_inventory_control)
│   ├── Products                        (menu_product_variant_config_stock)
│   ├── Product Variants                (product_product_menu)            [product variants]
│   ├── Lots / Serial Numbers           (menu_action_production_lot_form) [production_lot]
│   └── Packages                        (menu_package)                    [tracking_lot]
│
├── Reporting                           (menu_warehouse_report)           [Manager]
│   ├── Stock                           (menu_product_stock)  product on-hand by location
│   ├── Locations                       (menu_valuation)     stock.quant by location
│   ├── Moves History                   (stock_move_line_menu) stock.move.line (done)
│   └── Moves Analysis                  (stock_move_menu)    stock.move pivot/graph
│
└── Configuration                       (menu_stock_config_settings)      [Manager]
    ├── Settings                        (menu_stock_general_settings)     [system]
    │
    ├── Warehouse Management            (menu_warehouse_config)
    │   ├── Warehouses                  (menu_action_warehouse_form)
    │   ├── Operations Types            (menu_pickingtype)
    │   ├── Locations                   (menu_action_location_form)       [multi-locations]
    │   ├── Routes                      (menu_routes_config)              [adv_location]
    │   ├── Rules                       (menu_action_rules_form)          [adv_location]
    │   ├── Storage Categories          (menu_storage_categoty_config)    [multi-locations]
    │   └── Putaway Rules               (menu_putaway)                    [multi-locations]
    │
    ├── Products                        (menu_product_in_config_stock)
    │   ├── Product Categories          (menu_product_category_config_stock)
    │   ├── Attributes                  (menu_attribute_action)           [product variants]
    │   ├── Units & Packagings          (menu_stock_uom_form_action)      [uom]
    │   └── Barcode Nomenclature        (menu_wms_barcode_nomenclature_all) [technical]
    │
    └── Delivery                        (menu_delivery)
        └── Package Types               (menu_packaging_types)            [tracking_lot]
```

**Memorise this picture.** The common errors from pre-17 memory:

- There is **no** "Inventory › Inventory Control › Stock Moves" anymore — the
  live moves list is **Reporting › Moves Analysis**; the executed lines are
  **Reporting › Moves History**.
- "Physical Inventory" lives under **Operations › Adjustments** and opens the
  `stock.quant` list (the "what do I have vs what did I count" screen), not a
  separate `stock.inventory` model (that model was removed; adjustments now
  edit the quant's counted quantity directly).
- The Transfers submenu is **pre-split by direction** (Receipts / Deliveries /
  Internal) via server actions.

---

## 2. Core Models a Developer Must Know

Think of Odoo inventory as one big **move engine** layered over a **quant
ledger**. Behind every screen is a small set of tables, and the addons only
extend these.

```
                    ┌────────────────────────┐
                    │     stock.picking      │  the transfer header
                    │  state: draft|waiting|confirmed|
                    │         assigned|done|cancel
                    │  picking_type_id (in/out/internal),
                    │  location_id, location_dest_id,
                    │  partner_id, scheduled_date, origin,
                    │  backorder_id, move_type (shipping policy)
                    └───────────┬────────────┘
                                │ One2many → move_ids
                                ▼
                    ┌────────────────────────┐
                    │      stock.move        │  the demand line
                    │  product_id, product_uom_qty (Demand),
                    │  quantity (Done),
                    │  procure_method (mts/mto), rule_id,
                    │  location_id → location_dest_id
                    └───────────┬────────────┘
                                │ One2many → move_line_ids
                                ▼
                    ┌────────────────────────┐
                    │   stock.move.line      │  the executed/reserved line
                    │  lot_id, package_id → result_package_id,
                    │  owner_id, quantity (done), picked,
                    │  location_id → location_dest_id
                    └───────────┬────────────┘
                       reserves / synchronises against
                                ▼
                    ┌────────────────────────┐
                    │     stock.quant        │  the truth on the floor
                    │  product_id, location_id, lot_id,
                    │  package_id, owner_id  ← unique key
                    │  quantity (on hand, readonly),
                    │  reserved_quantity,
                    │  available_quantity = quantity − reserved,
                    │  inventory_quantity (the counted field)
                    └────────────────────────┘

       stock.warehouse       → reception_steps / delivery_steps + creates
                                picking types, locations, routes, rules
       stock.picking.type    → code (in/out/internal) + sequence +
                                reservation_method + create_backorder
       stock.location        → nested tree + usage + removal strategy
       stock.route           → rules + selectable flags (product/categ/wh/pkg)
       stock.rule            → action (pull/push/buy/manufacture) + procure_method
       stock.warehouse.orderpoint  → min/max reordering rule
       stock.lot             → lots / serial numbers (traceability)
       stock.scrap           → wraps a scrap move to an inventory loss location
```

**Rule:** a transfer IS a `stock.picking`. Its demand lines are `stock.move`
(what you *want*), which fan out into `stock.move.line` (what you *actually
picked*, per lot/package). `stock.quant` is the single source of truth for
"what is where" — every move line reserves against it and updates it on
validation. 90% of inventory customizations inherit one of these four models.

---

## 3. Menu / Action Reference (4-part)

### 3.1 Operations

#### Receipts (`incoming`)

- **Definition:** A receipt is the transfer that brings goods *into* your stock from an external source (a vendor). Its source location is the virtual "Vendors" location; its destination is your Input or Stock location depending on the warehouse's reception steps.
- **Usage:** This is how purchased stock hits the books. Confirming a draft purchase order generates the receipt; validating it increases on-hand quantity, (with `stock_account`) creates the valuation layer and the accounting entry, and — if it feeds a chained move — releases the next step (Pick or Storage).
- **Config:** Warehouse; a vendor partner; a product with a Buy route; an `incoming` operation type.
- **Workflow:**
  1. `Inventory › Operations › Transfers › Receipts` → New (or open the auto-generated one from a PO).
  2. Set vendor, scheduled date. Add lines (product / quantity).
  3. *Mark as Todo*.
  4. (Optional) *Check Availability* to reserve against existing stock if any.
  5. Enter done quantities, create/assign lots (if tracked), *Put in Pack* if packaging.
  6. *Validate* → state `done`, date done is set, quants updated, valuation layer created.

```
   Transfer life-cycle (one stock.picking record, changing state)
   ═══════════════════════════════════════════════════════════════

   [draft]  ──Mark as Todo──►  [confirmed]  ──Check Availability──►  [assigned]
                                      │                                    │
                                      │ Cancel                             │ Validate
                                      ▼                                    ▼
                                  [cancel]                             [done]
                                                       partial Validate → backorder (remainder)
```

#### Deliveries (`outgoing`)

- **Definition:** A delivery is the transfer that ships goods *out* to a customer. Its destination is the virtual "Customers" location; its source is Stock (1-step) or Output/Pack (multi-step).
- **Usage:** This is how a sale leaves the building. Confirming a sales order generates the delivery (or the first Pick in a pick → pack → ship chain); validating it decreases on-hand quantity, posts the cost-of-goods layer, and (with `stock_account`) credits stock and debits COGS.
- **Config:** Warehouse; a customer partner; a storable product with a delivery route; an `outgoing` operation type.
- **Workflow:**
  1. `Inventory › Operations › Transfers › Deliveries` → open the auto-generated one from an SO, or New.
  2. *Mark as Todo*; *Check Availability* reserves stock.
  3. Pick, scan lots (if tracked), *Put in Pack*.
  4. *Validate* → `done`. If partial, a backorder is created (per the type's backorder policy).
  5. (Optional) *Sign* (with the delivery-signature group) and *Print* the delivery slip.

#### Internal Transfers (`internal`)

- **Definition:** An internal transfer moves goods between two locations you own — Stock to Output, WH-A to WH-B (via transit), or bin to bin. It never touches customers or vendors.
- **Usage:** Used for warehouse-to-warehouse resupply, moving stock between zones, and any manual relocation. Both source and destination are internal/transit locations, so both sides update stock.
- **Config:** Multiple locations enabled; an `internal` operation type with both source and destination set.
- **Workflow:** `Inventory › Operations › Transfers › Internal` → New → from/to locations + product lines → *Mark as Todo* → *Check Availability* → *Validate*.

#### Physical Inventory (quant-based adjustments)

- **Definition:** Not a separate model — it is the `stock.quant` list with an editable *Counted* quantity column next to the readonly *On Hand*. The difference between the two is what gets applied.
- **Usage:** This is how you correct stock — cycle counts, full physical counts, fixing shrinkage. Editing the counted quantity and clicking *Apply* creates an inventory move from/to the Inventory Loss location dated *Inventory Date*, which updates the quant's on-hand.
- **Config:** A storable product with an existing quant (or create one by typing a counted quantity on a new row).
- **Workflow:**
  1. `Inventory › Operations › Adjustments › Physical Inventory`.
  2. Filter by product/location/owner. Type the counted quantity.
  3. Click *Apply* on each row (immediate) **or** select several rows and use the *Inventory Adjustment Reference / Reason* wizard to batch them with a name + counting date.

```
   Inventory adjustment (counted → quant)
   ═══════════════════════════════════════
   On Hand (readonly)   Counted (you type)   Difference
        12         →          10         →       −2
                                              │
                                              ▼  Apply
                                  inventory move   (src/dst = Inventory Loss)
                                              │
                                              ▼
                                  quant.quantity  →  10      (the move is logged
                                                           in Moves History)
```

> The `stock.inventory` model of older versions is gone. Cycle counting is
> now a *Next Count Date* on each quant + the annual-inventory-days setting.

#### Scrap — `stock.scrap`

- **Definition:** A scrap record moves a quantity of a product from a real location to an Inventory Loss location — the formal "this is broken / obsolete / lost, write it off" document.
- **Usage:** Auditable write-off. Unlike an inventory adjustment (which is silent), scrap produces a numbered `stock.scrap` document with a reason and (optionally) triggers replenishment of what you scrapped.
- **Config:** A product with stock; an `inventory` location flagged as a scrap location.
- **Workflow:**
  1. `Inventory › Operations › Adjustments › Scrap` → New (or *Scrap* button on a picking).
  2. Set product, *Quantity*, *Location* (source), *Scrap Location* (the loss location), lot (if tracked), reason tags.
  3. *Scrap Products* → creates a move to the loss location, marks the scrap `done`. If *Replenish* is ticked, a procurement is fired.

#### Landed Costs — `stock_landed_costs`

- **Definition:** A landed cost document takes extra costs incurred on a receipt (freight, insurance, customs duty) and distributes them across the products that were on that receipt, so the true unit cost reflects what getting them into stock actually cost.
- **Usage:** For import-heavy businesses. You receive the container, then create a landed cost pointing at the receipt, add cost lines (each with a split method), compute, and validate — the module creates extra valuation layers and (for real-time valuation) journal entries that bump each product's cost.
- **Config:** `stock_account` installed; products on the receipt use real-time valuation; a landed-cost product (a service/consumable representing the cost type) per cost line.
- **Workflow:**
  1. `Inventory › Operations › Adjustments › Landed Costs` → New.
  2. Pick the *Transfer* the costs apply to.
  3. Add *Cost Lines*: *Product* (the cost type), *Unit Cost*, *Split Method* (`equal` / `by_quantity` / `by_weight` / `by_volume` / `by_current_cost_price`).
  4. *Compute* → preview the *Valuation Adjustments* (former vs additional vs final cost per move).
  5. *Validate* → `done`, layers + journal entries posted.

#### Replenishment — manual reordering-rule worklist

- **Definition:** The Replenishment screen is the reordering-rule worklist: every product whose forecast is below its minimum, with a *Qty To Order* pre-computed and a one-click *Replenish* that fires the procurement.
- **Usage:** The buyer's morning queue. Open it, see everything that needs ordering, review the forecast, then either *Replenish* (auto-creates the right document — PO for Buy, MO for Manufacture, move for transfer) or *Snooze* until later.
- **Config:** Reordering rules configured; products with a fulfilment route.
- **Workflow:**
  1. `Inventory › Operations › Procurement › Replenishment`.
  2. The list is pre-filtered to products to reorder and not currently snoozed.
  3. Tick rows → *Replenish* generates the procurements.
  4. *Snooze* opens a small wizard (1 day / 1 week / 1 month / custom) and sets a snooze date.

#### References (technical)

- **Definition:** A lightweight tag linking a transfer to its origin documents (PO name, SO name, Manufacturing Order name) so chained moves can be grouped into one picking.
- **Usage:** Mostly system-managed. Moves that share the same references, picking type, and locations are grouped into a single transfer. Exposed under the technical menu for debugging broken chains.
- **Config:** None.
- **Workflow:** `Inventory › Operations › Procurement › References` (developer mode only).

#### Run Scheduler (technical)

- **Definition:** A manual trigger for the same code the daily cron runs.
- **Usage:** Click after a config change (new route, edited rule, fixed min qty) to make the system act immediately instead of waiting for tomorrow's cron. The scheduler runs all reordering rules and resolves chained procurements.
- **Config:** None (developer mode).
- **Workflow:** `Inventory › Operations › Run Scheduler`.

### 3.2 Products

#### Products (storable / sellable / purchasable)

- **Definition:** The master catalogue of physical things you stock. In Inventory we only care about *Storable* products — those are the ones with on-hand quantities.
- **Usage:** Set each product up once with the right category, UoM, tracking, and routes; every move afterwards just picks the product and inherits the rules.
- **Config:** Product category + UoM.
- **Workflow:** `Inventory › Products › Products` → New → tick *Storable Product*, set *Category*, *Sale Price* / *Cost*, *Tracking*, *Routes*. The *Forecasted* stat button opens the forecast graph + the replenishment wizard.

#### Product Variants

- **Definition:** Same product template, different attribute combinations (size, colour) — each variant is its own `product.product` and has its own stock.
- **Usage:** Only visible with *Product Variants* enabled. Use when you stock the same item in several distinguishable forms.
- **Workflow:** `Inventory › Products › Product Variants`.

#### Lots / Serial Numbers — `stock.lot`

- **Definition:** The register of every lot (batch) and serial number (one unit) ever created, with its current on-hand quantity and location.
- **Usage:** Traceability, expiry handling (`product_expiry`), FEFO removal, and lot-level valuation (`stock_account`).
- **Config:** Lots enabled; products with tracking on.
- **Workflow:** `Inventory › Products › Lots/Serial Numbers` → New. The *Traceability* stat button on a done picking opens the traceability report.

#### Packages — `stock.package`

- **Definition:** The list of every physical package (box/pallet/tote) and the units currently inside it, with its location and parent package.
- **Usage:** Needed for *Put in Pack*, shipping connectors (one label per package), and the package-level reception/delivery reports. Use *Unpack* to dissolve a package back to loose units.
- **Config:** Packages enabled.
- **Workflow:** `Inventory › Products › Packages`. Packages are usually created in-flight from a transfer's *Put in Pack* button.

### 3.3 Reporting

#### Stock (on-hand by product)

- **Definition:** A product list showing on-hand and forecast quantities per product, with an *On Hand* / *Forecasted* stat button that drills into the locations or moves behind the number.
- **Usage:** The "do we have it?" screen. The *Inventory at Date* wizard re-runs the same view as of a chosen date so you can see what was on hand historically.
- **Config:** Storable products exist.
- **Workflow:** `Inventory › Reporting › Stock` → filter by product/category. *Inventory at Date* to pick a historical snapshot.

#### Locations (on-hand by location)

- **Definition:** A `stock.quant` list grouped by location — each row is one product + lot + package + owner in one location with its on-hand/reserved/available quantities.
- **Usage:** The "where is it?" screen for warehouse staff. Also the cycle-count entry point (the counted column is editable here too).
- **Config:** Multiple locations / different owners enabled (the menu needs that, or developer mode).
- **Workflow:** `Inventory › Reporting › Locations` → group by location/product/lot.

#### Moves History — `stock.move.line`

- **Definition:** Every executed move line ever — each row is one product + lot + package + quantity that physically moved, with its source and destination locations and the picking it belonged to.
- **Usage:** The auditor's "prove it" screen. Filter by product/lot/date and you can trace every unit's journey. Also carries the *Revert Inventory Adjustment* action for undoing a bad count.
- **Config:** Done moves exist.
- **Workflow:** `Inventory › Reporting › Moves History` → filter. Read-only.

#### Moves Analysis — `stock.move`

- **Definition:** A pivot/graph view over `stock.move` — every demand line, with its quantities, locations, partner, and date, slicable any way the business asks.
- **Usage:** Management reporting without Excel — moves by product, by partner, by warehouse, by month.
- **Workflow:** `Inventory › Reporting › Moves Analysis` → pivot by dimensions, swap measures (*Quantity* / *Demand*).

### 3.4 Configuration

#### Settings — Inventory

- **Definition:** The master switches for the whole Inventory app, grouped: *Operations*, *Barcode*, *Shipping*, *Products*, *Traceability*, *Warehouse*, *Advanced Scheduling*, *Logistics*.
- **Usage:** Come here to enable a feature company-wide. Each toggle flips one of the hidden security groups or installs an addon.
- **Workflow:** `Inventory › Configuration › Settings` → toggle → *Save*. Key toggles:
  - *Operations › Put your products in packs* → Packages.
  - *Operations › Process transfers in batch* → installs `stock_picking_batch`.
  - *Operations › Reception Report*.
  - *Traceability › Full traceability* → Lots / Serial Numbers (+ GS1 print + scannable package contents sub-options).
  - *Traceability › Expiration dates* → installs `product_expiry`.
  - *Traceability › Display Serial & Lot on delivery slips*.
  - *Warehouse › Store products in specific locations* → Multiple Locations.
  - *Warehouse › Use your own routes* → Routes + Rules menus.
  - *Logistics › Dropshipping* → installs `stock_dropshipping`.
  - *Logistics › Replenish on Order (MTO)* → re-enables the MTO route.

#### Warehouses — `stock.warehouse`

- **Definition:** The warehouse record itself — name, short code, partner address, the reception/delivery step choices, the resupply warehouses, and (computed) its locations, picking types, routes, and rules.
- **Usage:** Edit this to change how the warehouse flows. Switching reception steps from 1-step to 3-step activates the Input + QC locations, creates the QC/Storage picking types, and rewrites the reception route's rules.
- **Config:** Company exists.
- **Workflow:** `Inventory › Configuration › Warehouse Management › Warehouses`. The *Routes* stat button shows every route + rule generated for this warehouse.

#### Operations Types — `stock.picking.type`

- (Definition/Usage/Config as in §0.3.) **Workflow:** `Inventory › Configuration › Warehouse Management › Operations Types`. Notable fields: *Reservation Method* + *Reservation Days Before*, *Create Backorder*, *Create New Lots* / *Use Existing Lots*, auto-print options (delivery slip / lot labels / reception report / packages), and *Favorite Users* (shows the type on a user's kanban).

#### Locations — `stock.location`

- (As in §0.2.) **Workflow:** `Inventory › Configuration › Warehouse Management › Locations`. Tree view by parent. Set *Usage*, *Barcode*, *Removal Strategy*, *Storage Category*, *Cyclic Count Frequency*.

#### Routes — `stock.route`

- **Definition:** A named bag of rules. The selectable flags decide where the route shows up as a choice: on the product form, on the category form, on the warehouse form, or on package types.
- **Usage:** The user-facing unit of procurement configuration. You tick a route on a product; the rules inside the route do the work.
- **Workflow:** `Inventory › Configuration › Warehouse Management › Routes`. Open a route to inspect/edit its rules (action, from/to, operation type, procure method).

#### Rules — `stock.rule`

- **Definition:** The single step behind a route — "from X to Y via this operation type, pull/push, MTS/MTO".
- **Usage:** Where the actual flow logic lives. Developers add a custom step by adding a rule; the procurement engine finds it via the product's routes.
- **Config:** Routes enabled.
- **Workflow:** `Inventory › Configuration › Warehouse Management › Rules`. Key fields: *Action*, *Procure Method*, *Operation Type*, *Source / Destination Location*, *Propagate Cancel / Carrier*, *Lead Time (days)*.

#### Storage Categories — `stock.storage.category`

- **Definition:** A capacity/mixing policy for a location — max weight, *Allow New Products* (`empty` / `same` / `mixed`), and per-product/per-package-type capacity rows.
- **Usage:** Lets putaway rules respect "this rack is for one product only" or "max 500 kg". Without a category, putaway just returns the first matching destination.
- **Config:** Multiple locations enabled.
- **Workflow:** `Inventory › Configuration › Warehouse Management › Storage Categories`. Assign on the location form's *Storage Category* field.

#### Putaway Rules — `stock.putaway.rule`

- **Definition:** A directed pair: "when product X (or category Y, or package type Z) arrives at location A, send it to location B". The most specific rule wins (product beats category; lower sequence wins ties).
- **Usage:** Automates bin assignment on receipts and internal moves.
- **Config:** Multiple locations enabled.
- **Workflow:** `Inventory › Configuration › Warehouse Management › Putaway Rules` → New → *Arrival Location* + *Destination Location* + optional specificity (*Product* / *Category* / *Package Types*) + optional *Storage Category*.

#### Product Categories — `product.category`

- **Definition:** The folder a product sits in. With `stock_account` it carries the valuation settings (*Valuation* periodic/real_time, *Cost Method* standard/average/fifo) and the stock accounts (stock valuation, input/output accounts, stock journal).
- **Usage:** Set valuation once per category and every product inherits it — far less error-prone than per-product accounting.
- **Workflow:** `Inventory › Configuration › Products › Product Categories`. Routes can also be set on the category and inherited by products.

#### Package Types — `stock.package.type`

- **Definition:** A reusable template for packages — name, dimensions (height/width/length), *Base Weight*, *Max Weight*, a barcode sequence, a *Package Use* (`disposable`/`reusable`), and selectable routes.
- **Usage:** When a worker clicks *Put in Pack* on a transfer, the package they create inherits the type's dimensions and weight for shipping-rate calculations and carrier label printing.
- **Config:** Packages enabled.
- **Workflow:** `Inventory › Configuration › Delivery › Package Types`.

---

### 3.5 The Transfer Engine — How Moves Update Quants

This is the inventory-specific "deep" workflow — the equivalent of bank
reconciliation for stock. It is what makes a transfer not just a form but a
controlled, auditable movement of value. Read this once and the rest of the app
makes sense.

```
   Reservation & validation flow (the core loop)
   ═════════════════════════════════════════════

   draft move  ──Mark as Todo──►  confirmed move
        │                              │
        │                              │  Check Availability
        │                              │   → finds matching quants
        │                              │     (ordered by the location's
        │                              │      removal strategy: FIFO oldest,
        │                              │      LIFO newest, FEFO earliest expiry)
        │                              │   → bumps each quant's reserved qty
        │                              ▼
        │                          assigned (Ready)
        │                              │
        │                              │  Validate
        │                              │   → each move line executes:
        │                              │       −=  on the source quant
        │                              │       +=  on the destination quant
        │                              │       reserved qty reset to 0
        │                              ▼
        │                            done   (immutable once locked)
        │
        └─ partial Validate → backorder picking (carries the remainder)
```

**The single source of truth is `stock.quant`.** A quant is one row per
(product + location + lot + package + owner). Every reservation increments its
*Reserved* quantity; every validation moves its *On Hand*. You never edit a
quant by hand in normal use — you drive a move or a counted quantity, and the
quant follows.

```
   The three quantities on a quant (and how they relate)
   ══════════════════════════════════════════════════════
   On Hand          the authoritative quantity (readonly to users;
                    moved only by validating moves or applying counts)
   Reserved         units already spoken for by assigned moves
   Available        = On Hand − Reserved        ← what's free to promise
   Counted          the value you type; applying it writes the
                    difference onto On Hand via an inventory move
```

#### Procurement: how demand becomes moves

Routes and rules are the engine behind automatic replenishment and chained
transfers. The daily scheduler (or the *Run Scheduler* menu) turns demand into
the moves that carry stock through the warehouse:

```
   demand   (sales order line, orderpoint below min, manual replenish)
        │
        │  the product's routes each offer candidate rules;
        │  the highest-priority matching rule wins
        ▼
   rule action   →   pull / push / pull_push / buy / manufacture
        │
        ▼
   stock.move created, chained to its neighbours
   (its origin ↔ destination moves)
        │
        ├──  make_to_stock  →  Check Availability reserves it now
        └──  make_to_order  →  triggers the upstream rule (recurse)
```

So a confirmed sales order for a "Buy" product spawns a draft purchase order,
while the same order for a stocked product spawns a Pick → (Pack) → Ship chain
of transfers — each one released automatically as the previous one validates.

#### Backorders and returns are themselves transfers

- **Backorder:** when you validate less than the demanded quantity, Odoo clones
  the remaining moves into a new `stock.picking` linked back via *Backorder of*.
  The original validates as `done`; the backorder carries the rest.
- **Return:** the return wizard copies a done transfer's lines with the source
  and destination swapped, re-linking the chain so quantities flow back. A
  return of a return (an exchange) is fully supported.

---

## Appendix — Where to Look in Source

| Need                                                  | File (path under repo root)                                  |
| ----------------------------------------------------- | ------------------------------------------------------------ |
| The core menu tree                                    | `addons/stock/views/stock_menu_views.xml`                  |
| Transfer form + Transfers/Deliveries/Internal actions | `addons/stock/views/stock_picking_views.xml`               |
| How a transfer validates                              | `addons/stock/models/stock_picking.py`                     |
| How a move reserves / confirms / cancels              | `addons/stock/models/stock_move.py`                        |
| How the on-hand quantity is stored                    | `addons/stock/models/stock_quant.py`                       |
| Inventory adjustment wizard                           | `addons/stock/wizard/stock_inventory_adjustment_name.py`   |
| Backorder wizard                                      | `addons/stock/wizard/stock_backorder_confirmation.py`      |
| Return wizard                                         | `addons/stock/wizard/stock_picking_return.py`              |
| Replenish-from-product wizard                         | `addons/stock/wizard/product_replenish.py`                 |
| Reorderpoint snooze                                   | `addons/stock/wizard/stock_orderpoint_snooze.py`           |
| How a warehouse builds its routes/rules               | `addons/stock/models/stock_warehouse.py`                   |
| Reordering rule + scheduler                           | `addons/stock/models/stock_orderpoint.py`                  |
| How procurement dispatches to pull/push/buy           | `addons/stock/models/stock_rule.py`                        |
| Lot / serial number model                             | `addons/stock/models/stock_lot.py`                         |
| Scrap model                                           | `addons/stock/models/stock_scrap.py`                       |
| Traceability report                                   | `addons/stock/report/stock_traceability.py`                |
| Forecast graph on product form                        | `addons/stock/report/stock_forecasted.py`                  |
| Product stock fields (`qty_available`, etc.)        | `addons/stock/models/product.py`                           |
| Inventory valuation (accounts, layer)                 | `addons/stock_account/models/`                             |
| Landed costs                                          | `addons/stock_landed_costs/models/stock_landed_cost.py`    |
| Batch transfers                                       | `addons/stock_picking_batch/models/stock_picking_batch.py` |
| Dropship route + rule                                 | `addons/stock_dropshipping/`                               |
| Delivery carrier on picking                           | `addons/stock_delivery/models/stock_picking.py`            |
| Security groups                                       | `addons/stock/security/stock_security.xml`                 |

---

*Menu paths in this guide are verified against the installed XML on branch
`om_mps_rebuild`. If the inventory stack is upgraded, re-read
`stock_menu_views.xml` and each module's menu XML — the path labels are the
thing most likely to shift between versions.*
