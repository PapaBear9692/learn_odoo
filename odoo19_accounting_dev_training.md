# Odoo 19 Community — Accounting Developer Training Guide

**Scope:** Odoo 19 Community `account` module (`addons/account/`) + the OdooMates
stack installed under `custom_addons/` (`om_account_accountant`, `om_account_asset`,
`om_account_budget`, `om_account_daily_reports`, `om_account_followup`) and their
siblings (`accounting_pdf_reports`, `om_fiscal_year`, `om_recurring_payments`).

> **Every menu path in this guide is read from the actual XML in
> `addons/account/views/account_menuitem.xml` and the menu XML of each module
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
4. [Developer Patterns Extracted from the Stack](#4-developer-patterns-extracted-from-the-stack)
5. [Odoo 19-Specific Notes](#5-odoo-19-specific-notes)
6. [Extension Recipes / Exercises](#6-extension-recipes--exercises)

---

## 0. Complete Configuration Checklist (do this first)

Do these in order. Each step unblocks the next. When step 9 is done the whole
accounting module is ready.

```
            ACCOUNTING SETUP (strict order)

  Company ─► Fiscal Year ─► Chart of Accounts ─► Journals
     │              │              │                 │
     │              │              ▼                 │
     │              │       Account Groups/Tags      │
     │              │              │                 ▼
     ▼              ▼              │          Payment Methods
  Currencies   Lock Dates ◄────────┘          (inbound/outbound)
     │                                              │
     ▼                                              ▼
  Taxes ◄── Fiscal Positions ◄── Payment Terms ◄── Partners/Bank
     │                                              │
     ▼                                              ▼
  Products ──────────────────────────────────► READY TO TRANSACT
```

### 0.1 Company & Currencies

- **Definition:** The company is the legal entity whose books you're keeping — "whose money is this?" Every transaction belongs to one company. The currency is the money those books are counted in (e.g. BDT, USD).
- **Usage:** You set this up first because nothing else makes sense without it — you can't post an entry without knowing which company and currency it belongs to.
- **Config:** Database + admin user exist.
- **Workflow:**
  1. Open `Settings › Companies › Manage Companies`, create the company.
  2. Open the company, set `currency_id`.
  3. In `Settings`, enable *Multi-Currencies* if you trade in more than one currency.
  4. `Accounting › Configuration › Accounting › Currencies` → activate the ones you need.

### 0.2 Fiscal Year & Lock Dates — `om_fiscal_year`

- **Definition:** The fiscal year is the 12-month window the company reports on (a year doesn't have to be Jan–Dec). Lock dates are "no more changes past this point" cut-offs — once you set one, the past is frozen.
- **Usage:** Auditors and tax authorities need a clean, closed period. Lock dates guarantee that once March is closed, nobody can sneak a back-dated entry into March later. The fiscal year also tells depreciation and budgets how to slice time.
- **Config:** Company exists.
- **Workflow:**
  1. Open the company form, set `fiscalyear_last_month` + `fiscalyear_last_day`.
  2. `Accounting › Configuration › Accounting › Fiscal Year` → New → name + date_from + date_to.
  3. (Later) `Accounting › Accounting › Lock Dates` → set tax/period/fiscalyear lock dates.

### 0.3 Chart of Accounts — core

- **Definition:** The Chart of Accounts is the complete list of "buckets" money can fall into — like the table of contents of the general ledger. Each bucket is an *account* (e.g. "Office Rent", "Bank — City Bank", "VAT Payable"). Every transaction drops money into one or more of these buckets.
- **Usage:** When you record a sale you say *which* accounts to credit and debit. Without accounts you can't post anything. Reports (Balance Sheet, P&L) are just these buckets filtered up.
- **Config:** Company + currency set.
- **Workflow:**
  1. `Accounting › Configuration › Accounting › Chart of Accounts`.
  2. Install a chart template (button *New* prompts for template) or import a `l10n_*` module first.
  3. For each row confirm `code`, `name`, `account_type` (`asset_receivable`, `asset_cash`, `liability_payable`, `equity`, `income`, `expense`, `asset_non_current`, …).
  4. (Optional) `Accounting › Configuration › Accounting › Account Groups` to roll up by code prefix.

```
account.account types that matter most:
  asset_receivable   customer owes us      (debit normal)
  asset_cash         cash on hand
  asset_current      inventory, bank, prepaids
  asset_non_current  fixed assets (used by om_account_asset)
  liability_payable  we owe vendor         (credit normal)
  equity             capital, retained earnings
  income             revenue
  expense            operating costs
```

### 0.4 Journals — core

- **Definition:** A journal is a sequential notebook for one kind of transaction. You typically have one each for Sales, Purchases, Cash, Bank, and Miscellaneous (manual adjustments). Each entry gets a running number inside its journal.
- **Usage:** Journals keep entries organised by source and give each entry a unique, gap-free number — which is what auditors check. They also carry the default accounts and payment methods for their type.
- **Config:** Chart of Accounts has cash/bank accounts available.
- **Workflow:**
  1. `Accounting › Configuration › Accounting › Journals` → New.
  2. Set `code`, `type` (sale/purchase/cash/bank/general).
  3. For cash/bank set `default_account_id`.
  4. Set the entry sequence and refund sequence.
  5. Under *Available Payment Methods* tick the inbound/outbound method lines.

### 0.5 Taxes & Fiscal Positions — core

- **Definition:** A tax is a percentage or fixed amount added to an invoice line (VAT, AIT, customs duty). A fiscal position is a smart rule that swaps taxes automatically based on *who* you're dealing with — e.g. zero-rate VAT on an export, apply 15% VAT on a local sale.
- **Usage:** You define each tax once with the account it should post to. Fiscal positions then pick the right tax for each customer so users don't have to remember "export = 0%, local = 15%".
- **Config:** CoA has dedicated tax liability/expense accounts.
- **Workflow:**
  1. `Accounting › Configuration › Accounting › Taxes` → New.
  2. Set `amount`, `amount_type` (percent/fixed/group).
  3. In *Distribution*, add repartition lines (one *Base* + one *Tax* row, each with an account) for both Invoice and Credit Note.
  4. `Accounting › Configuration › Accounting › Fiscal Positions` → New → set country/country_group auto-detect → add *Tax Mapping* rows (source tax → target tax).

### 0.6 Payment Terms — core

- **Definition:** A payment term is the agreed schedule for when an invoice gets paid — e.g. "30% advance, 70% within 30 days of delivery", or simply "Net 45".
- **Usage:** It tells Odoo the due date(s) for each invoice, which then drives the aged-report buckets and the follow-up reminders. Without it, every invoice is "due immediately".
- **Config:** None.
- **Workflow:**
  1. `Accounting › Configuration › Invoicing › Payment Terms` → New.
  2. Add term lines: `value` (percent/balance/fixed) + `nb_days`/`days_after`.
  3. Set on partner (`property_payment_term_id`) or pick on each invoice.

### 0.7 Partners & Bank Accounts — core

- **Definition:** A partner is anyone you do business with — customers and vendors are both partners, just flagged differently. A bank account is a partner's IBAN/account number at a bank.
- **Usage:** Every invoice and payment points at a partner. Setting them up with the right receivable/payable accounts and payment terms means invoices post to the correct ledger bucket automatically.
- **Config:** CoA + receivable/payable accounts exist.
- **Workflow:**
  1. `Accounting › Customers › Customers` (or `Vendors › Vendors`) → New.
  2. Set receivable/payable accounts, payment terms.
  3. *Bank Accounts* tab → add `acc_number` + `bank_id`.
  4. Add a child contact `type='invoice'` (used by follow-up emails).

### 0.8 Products — core

- **Definition:** A product is anything you sell or buy — a physical item (laptop, paper roll) or a service (printing job, consulting hour).
- **Usage:** When you pick a product on an invoice line, Odoo auto-fills the price, the income/expense account, and the taxes. It's the shortcut that makes invoicing fast and consistent.
- **Config:** Income/expense accounts and taxes exist.
- **Workflow:**
  1. `Accounting › Customers › Products` (or `Vendors › Products`) → New.
  2. Set `categ_id`, `list_price`, `standard_price`.
  3. *General* tab → set income account + sale taxes; expense account + purchase taxes.
  4. For `om_account_asset` (optional): set `asset_category_id` or `deferred_revenue_category_id`.

### 0.9 Security Groups — `om_account_accountant/security/group.xml`

- **Definition:** These are the four accounting job-roles: **Invoicing/Bookkeeper** (creates day-to-day invoices/bills), **Accountant** (runs depreciation, budgets, follow-ups), **Advisor** (closes periods, changes settings), **Auditor** (read-only oversight).
- **Usage:** You assign each user the role that matches their responsibility, and Odoo shows/hides menus and buttons accordingly. It's how you stop a junior bookkeeper from locking the fiscal year.
- **Config:** Users exist.
- **Workflow:** `Settings › Manage Users` → open a user → *Access Rights* tab → assign the accounting group.

> Steps 0.1–0.9 done → you can invoice, pay, depreciate, budget, dun, and report.

---

## 1. The Real Odoo 19 Menu Tree

Read directly from `account_menuitem.xml` + every module's menu XML. The top
menu is `menu_finance`, named **"Invoicing"** in core, renamed to
**"Accounting"** by `om_account_accountant/views/menu.xml`.

```
Accounting  (menu_finance)  ← top, renamed from "Invoicing"
│
├── Dashboard                          [journal kanban]
│
├── Customers                          (menu_finance_receivables)
│   ├── Invoices                       out_invoice
│   ├── Credit Notes                   out_refund
│   ├── Payments                       account.payment (customer)
│   ├── Products
│   └── Customers                      res.partner (customers)
│
├── Vendors                             (menu_finance_payables)
│   ├── Bills                          in_invoice
│   ├── Refunds                        in_refund
│   ├── Payments                       account.payment (vendor)
│   ├── Products
│   └── Vendors                        res.partner (vendors)
│
├── Follow-Ups                          [om_account_followup]
│   ├── Send Letters and Emails        followup.print wizard (the dunning run)
│   ├── Do Manual Follow-Ups           res.partner filtered by overdue
│   ├── My Follow-Ups                  res.partner (mine)
│   └── Follow-ups Analysis            followup.stat graph
│
├── Accounting                          (menu_finance_entries)
│   ├── Transactions
│   │   ├── Journal Items              account.move.line list
│   │   └── Analytic Items
│   ├── Bank and Cash                  [om_account_accountant]
│   │   ├── Bank Statements
│   │   └── Cash Statements
│   ├── Journals                       [om_account_accountant]
│   │   ├── Sales / Purchase / Bank and Cash / Miscellaneous
│   ├── Ledgers                        [accounting_pdf_reports]
│   │   ├── General Ledger             live move.line list
│   │   └── Partner Ledger             live move.line list
│   ├── Generate Entries               [om_account_asset]
│   │   └── Generate Assets Entries    depreciation wizard
│   ├── Closing
│   └── Lock Dates                     [om_fiscal_year] (group: Advisor)
│
├── Review                              (account_audit_menu)
│   ├── Control
│   │   └── Journal Entries            account.move list (full moves)
│   └── Logs
│
├── Reporting                           (menu_finance_reports)
│   ├── Financial Reports
│   │   ├── Balance Sheet
│   │   └── Profit and Loss
│   ├── Partner Reports
│   │   ├── Partner Ledger             PDF
│   │   ├── Aged Partner Balance       PDF
│   │   ├── Aged Receivable            PDF
│   │   └── Aged Payable               PDF
│   ├── Audit Reports
│   │   ├── General Ledger             PDF
│   │   ├── Trial Balance              PDF
│   │   ├── Tax Report                 PDF
│   │   └── Journals Audit             PDF
│   ├── Management
│   │   ├── Invoice Analysis           core
│   │   ├── Analytic Report            core
│   │   ├── Assets                     asset.asset.report  [om_account_asset]
│   │   └── Budgets Analysis           crossovered.budget.lines [om_account_budget]
│   ├── Statement Reports
│   ├── Taxes & Fiscal
│   └── Daily Reports                  [om_account_daily_reports]
│       ├── Day Book
│       ├── Cash Book
│       └── Bank Book
│
└── Configuration                       (menu_finance_configuration, group: Advisor)
    ├── Settings                       res.config.settings
    ├── Accounting                     (account_account_menu)
    │   ├── Chart of Accounts
    │   ├── Taxes
    │   ├── Journals
    │   ├── Asset Category             [om_account_asset]
    │   ├── Budgetary Positions        [om_account_budget]
    │   ├── Fiscal Year                [om_fiscal_year]
    │   ├── Account Tags               [om_account_accountant]
    │   ├── Account Groups             [om_account_accountant]
    │   ├── Assets                     [om_account_asset] (group: Advisor)
    │   ├── Budgets                    [om_account_budget] (group: Advisor)
    │   ├── Currencies
    │   ├── Fiscal Positions
    │   ├── Multi-Ledger
    │   ├── Tax Groups                 (technical)
    │   └── Cash Roundings
    ├── Invoicing
    │   ├── Payment Terms
    │   ├── Incoterms                  (technical)
    │   └── Product Categories
    ├── Online Payments
    │   └── Payment Methods            (technical — base.group_no_one)
    ├── Recurring Payment              [om_recurring_payments]
    │   ├── Recurring Template
    │   └── Recurring Payment
    ├── Analytic Accounting
    │   ├── Analytic Distribution Models
    │   ├── Analytic Accounts
    │   └── Analytic Plans
    ├── Templates                      (technical)
    └── Follow-up                      [om_account_followup]
        └── Follow-up Levels           followup.followup config
```

**Memorise this picture.** Most errors come from assuming pre-17 paths (e.g. an
"Invoices" menu under a top-level "Accounting → Customers" branch that doesn't
exist). In Odoo 19, invoices live under **Customers**, bills under **Vendors**,
the full-move list under **Review › Control**, and configuration items are flat
under **Configuration › Accounting**.

---

## 2. Core Models a Developer Must Know

Think of Odoo accounting as one big double-entry engine. Behind every screen is
a small set of tables, and the om_* modules only extend these.

```
                    ┌────────────────────────┐
                    │     account.move       │  the header — one journal entry
                    │  state: draft|posted|cancel
                    │  move_type: out_invoice|in_invoice|
                    │              out_refund|in_refund|entry
                    │  partner_id, journal_id, date,
                    │  invoice_date, invoice_date_due, amount_*
                    └───────────┬────────────┘
                                │ One2many → line_ids
                                ▼
                    ┌────────────────────────┐
                    │   account.move.line    │  a single debit or credit
                    │  account_id, debit, credit,
                    │  partner_id, date_maturity,
                    │  tax_ids, analytic_distribution,
                    │  full_reconcile_id  ← "settled"
                    └───────────┬────────────┘
                                │ reconciled against
              ┌─────────────────┼─────────────────┐
              ▼                 ▼                 ▼
       account.payment   account.bank.statement   account.partial.reconcile
       (register pay)    (bank import/match)      (reconciliation primitive)

       account.journal  → sequence + default accounts + payment method lines
       account.account  → the ledger account (CoA row)
       account.tax      → with repartition lines (base vs tax)
```

**Rule:** an invoice IS an `account.move`. `out_invoice` = customer invoice,
`in_invoice` = vendor bill, `entry` = generic journal entry. 90% of finance
customizations inherit `account.move` or `account.move.line`.

---

## 3. Menu / Action Reference (4-part)

### 3.1 Dashboard

#### Dashboard

- **Definition:** A visual card-per-journal landing page — each card shows the journal's current balance and the actions you can take on it (new invoice, items to reconcile, etc.).
- **Usage:** It's the accountant's home screen. Instead of hunting through menus, you click the journal you want to work on and jump straight in.
- **Config:** At least one journal created.
- **Workflow:** Open `Accounting › Dashboard`. Click a journal card to act on it (create an invoice, reconcile bank lines, etc.).

### 3.2 Customers

#### Invoices

- **Definition:** The bill you send to a customer for goods or services — "you owe us this much by this date", with a line-by-line breakdown of what was sold.
- **Usage:** This is how a sale hits the books. Confirming an invoice increases what the customer owes you (receivable), records the revenue, and posts any tax due.
- **Config:** Customer partner, product with income account + sale tax, a Sale journal.
- **Workflow:**
  1. Open `Accounting › Customers › Invoices`.
  2. Click *New*.
  3. Pick the customer → fiscal position auto-selects taxes.
  4. Add lines (product / quantity / price); taxes auto-apply.
  5. Click *Confirm* → the move is `posted`.
  6. Click *Register Payment* → choose journal + amount → *Create Payment*.

#### Credit Notes

- **Definition:** A "reverse invoice" you issue to a customer — used when an invoice was wrong, goods were returned, or you're writing off a bad debt. It cancels part or all of the original.
- **Usage:** Instead of deleting a posted invoice (which auditors hate), you issue a credit note that offsets it. The two are then linked together.
- **Config:** A posted source invoice.
- **Workflow:**
  1. `Accounting › Customers › Credit Notes` → *New*, **or** open the invoice → *Add Credit Note*.
  2. Pick the *Credit Method* (full refund / partial), *Reason*, *Date*.
  3. *Confirm* → the refund move is created and auto-reconciled against the original.

#### Payments (Customers)

- **Definition:** A record that a customer paid you — money coming in. It might be against a specific invoice or a general advance.
- **Usage:** When a payment arrives, you record it here so the customer's receivable goes down and the bank/cash account goes up. Once matched to the invoice, the invoice is "paid".
- **Config:** Bank/Cash journal with inbound payment method; customer partner.
- **Workflow:**
  1. `Accounting › Customers › Payments` → *New*.
  2. Set Customer, Amount, Journal, Payment Date.
  3. *Confirm* → posts + reconciles against the open invoice(s).

#### Customers (partners)

- **Definition:** Your customer master list — name, address, contacts, the accounts they post to, their payment terms and credit limit.
- **Usage:** Set each customer up once with the right defaults; every invoice afterwards just picks the customer and inherits everything.
- **Config:** None.
- **Workflow:** `Accounting › Customers › Customers` → *New* → fill receivable/payable accounts + payment terms → *Save*.

#### Products (sellable)

- **Definition:** The catalogue of things you sell — each carrying its price, sales account, and default taxes.
- **Usage:** Pick a product on an invoice line and the price/account/tax fill in automatically. Maintaining the catalogue keeps invoicing fast and consistent.
- **Config:** Product category + income account.
- **Workflow:** `Accounting › Customers › Products` → *New* → set price, income account, sale tax.

### 3.3 Vendors

#### Bills

- **Definition:** A bill from a supplier — the invoice they sent *you* for things you bought. It's the vendor-side mirror of a customer invoice.
- **Usage:** Recording a bill is how a purchase hits the books: it increases what you owe the vendor (payable), records the expense (or inventory), and captures any input tax you can reclaim.
- **Config:** Vendor partner, product with expense account + purchase tax, a Purchase journal.
- **Workflow:**
  1. Open `Accounting › Vendors › Bills`.
  2. *New* → pick vendor.
  3. (Optional) attach the PDF bill; use *Send to OCR*.
  4. Add lines → *Confirm*.
  5. *Register Payment* to pay.

#### Refunds

- **Definition:** A credit note you *receive* from a vendor — they owe you money back (returned goods, overcharge correction).
- **Usage:** Reduces what you owe that vendor. Linked back to the original bill so the pair is auditable.
- **Config:** A posted source bill.
- **Workflow:** `Accounting › Vendors › Refunds` → *New* (or *Add Credit Note* on the bill) → reason + reversal → *Confirm*.

#### Payments (Vendors)

- **Definition:** A record that you paid a vendor — money going out.
- **Usage:** When you send money to a supplier, record it here so the payable comes down and the bank/cash account comes down too.
- **Config:** Bank/Cash journal with outbound payment method; vendor partner.
- **Workflow:** `Accounting › Vendors › Payments` → *New* → vendor + amount + journal → *Confirm*.

#### Vendors (partners)

- **Definition:** Your supplier master list — same idea as the customer list, but for who you buy from.
- **Usage:** Set up each vendor with their payable account and supplier payment terms once, then bills just reference them.
- **Workflow:** `Accounting › Vendors › Vendors` → *New* → fill payable account + supplier payment terms.

#### Products (purchasable)

- **Definition:** Things you buy — each carrying its cost, purchase account, and default purchase taxes.
- **Workflow:** `Accounting › Vendors › Products` → *New* → set cost, expense account, purchase tax.

### 3.4 Follow-Ups — `om_account_followup`

Follow-ups are the structured "please pay us" process for overdue customers. You
define tiers (e.g. a polite email at 7 days late, a stronger letter at 30, a
manual call at 60), then run the cycle on all overdue customers at once.

```
   Dunning Run Flow
   ═══════════════
   Configure Follow-up Levels  (one plan per company)
        │   followup.line rows ordered by delay (days past due)
        │     each: send_email / send_letter / manual_action
        │           + email_template_id + manual_action_note
        ▼
   Follow-Ups › Send Letters and Emails   (followup.print wizard)
        │
        ▼
   _get_partners_followp()  ── SQL scan of overdue receivable move lines
        │                     assigns each the highest delay tier reached
        ▼
   do_process()
     ├─ do_update_followup_level()  stamps followup_line_id + date on aml
     ├─ process_partners()
     │     ├─ do_partner_manual_action()
     │     ├─ do_partner_mail()           sends level's email_template
     │     └─ collects partner_ids_to_print
     └─ clear_manual_actions()            clears partners now at 0
        │
        ▼
   followup.sending.results  ── summary screen + PDF letter
        │   report.om_account_followup.report_followup
        │   renders overdue table (_lines_get_with_partner)
```

#### Send Letters and Emails (the dunning run)

- **Definition:** A single wizard that processes *every* overdue customer in one go — emailing them, queuing printed letters, and assigning manual follow-up tasks, all based on how late each one is.
- **Usage:** Run it once a week or month. It replaces hundreds of individual emails and reminders with one button, and leaves an audit trail of who was contacted at which tier.
- **Config:** Follow-up Levels configured; customers with posted overdue invoices.
- **Workflow:**
  1. Open `Accounting › Follow-Ups › Send Letters and Emails`.
  2. Confirm the *Follow-up Sending Date*.
  3. (Optional) tick *Email Confirmation* / set subject.
  4. Click *Send emails and generate letters*.
  5. Read the summary screen (#emails / #letters / #manual actions) → print the PDF.

#### Do Manual Follow-Ups

- **Definition:** A filtered customer list showing only those who need a human action (a phone call, a visit) according to their follow-up tier.
- **Usage:** This is the collector's work queue — work each one, log what you did, mark it done.
- **Config:** Partners flagged by a prior run.
- **Workflow:** `Accounting › Follow-Ups › Do Manual Follow-Ups` → open a partner → *Follow-up* tab → read `payment_next_action` → do it → *Action Done*.

#### My Follow-Ups

- **Definition:** Same list as above but filtered to the customers assigned to *you* — your personal to-do list.
- **Usage:** Each collector sees only their own accounts, so nobody steps on each other's work.
- **Config:** You are assigned as responsible on at least one partner.
- **Workflow:** `Accounting › Follow-Ups › My Follow-Ups` → work your list.

#### Follow-ups Analysis

- **Definition:** A graph showing how many customers are at each follow-up tier and how much money sits in each bucket.
- **Usage:** Management uses it to see the shape of the overdue book — is the problem concentrated at the 30-day tier, or drifting into 90+?
- **Config:** At least one run executed.
- **Workflow:** `Accounting › Follow-Ups › Follow-ups Analysis` → switch measures/dimensions.

### 3.5 Accounting (entries, ledgers, statements, lock)

#### Journal Items

- **Definition:** A flat, searchable list of every single debit and credit ever posted — every line of every journal entry, all in one grid.
- **Usage:** This is the auditor's favourite screen. Filter by account, partner, or date range and you can trace exactly where a number came from or bulk-edit many lines at once.
- **Config:** Posted moves exist.
- **Workflow:** `Accounting › Accounting › Transactions › Journal Items` → filter by account/partner/date → pivot or export.

#### Analytic Items

- **Definition:** The "analytic" mirror of journal lines — same transactions but sliced by project/department/cost-centre instead of by ledger account.
- **Usage:** Lets you answer "how much did Project X cost?" even when the costs are spread across many accounts.
- **Config:** Analytic Accounting enabled and used.
- **Workflow:** `Accounting › Accounting › Transactions › Analytic Items`.

#### Bank Statements

- **Definition:** A bank statement as the bank sees it — a list of money in/out on a given account over a period, ending in a closing balance.
- **Usage:** This is the raw input to bank reconciliation. You import it (or type it in), then match each bank line to the corresponding Odoo entry so the books agree with the bank.
- **Config:** A `bank` journal.
- **Workflow:**
  1. `Accounting › Accounting › Bank and Cash › Bank Statements` → *New*.
  2. Pick the bank journal → set period + starting/ending balance.
  3. Add lines (or import a file).
  4. Click *Reconcile* on a line → match to an existing move *or* *Create and Match* → repeat until ending balance matches.

```
   Bank statement        Odoo journal entries
   ──────────────        ────────────────────
   +BDT 50,000  ─────────  Customer A payment   ✓
   -BDT  2,300  ─────────  Vendor X payment     ✓
   -BDT    500  ─? ──────  none → Create "Bank charge"
   = balance_end_real      = balance_end         ✓ closed
```

#### Cash Statements

- **Definition:** The same idea as a bank statement, but for a petty-cash / cash-on-hand tin — a count of what's physically there.
- **Usage:** Proves the cash box matches the books. Done daily or weekly to catch shortages early.
- **Config:** A `cash` journal.
- **Workflow:** `Accounting › Accounting › Bank and Cash › Cash Statements` → same steps as bank statements.

#### Journals (by type)

- **Definition:** Four pre-filtered views of journal entries, one per type — Sales entries, Purchase entries, Bank/Cash entries, and Miscellaneous (manual) entries.
- **Usage:** Quick way to look at just the sales, or just the manual adjustments, without filtering every time.
- **Workflow:** `Accounting › Accounting › Journals › Sales` (or *Purchase* / *Bank and Cash* / *Miscellaneous*).

#### General Ledger (live) — `accounting_pdf_reports`

- **Definition:** The General Ledger on screen — every transaction grouped by account, ready to expand and drill into. (The PDF version lives under Reporting.)
- **Usage:** Day-to-day browsing of the GL. Faster than printing a report when you just need to check something.
- **Config:** Posted entries exist.
- **Workflow:** `Accounting › Accounting › Ledgers › General Ledger` → use filters (account / date / journal) → expand groups.

#### Partner Ledger (live) — `accounting_pdf_reports`

- **Definition:** The same list grouped by customer/vendor instead of by account — each partner's running balance with the transactions behind it.
- **Usage:** The collector's and AP clerk's working screen — "show me everything Customer X owes me and why".
- **Workflow:** `Accounting › Accounting › Ledgers › Partner Ledger` → filter by partner.

#### Generate Assets Entries — `om_account_asset`

- **Definition:** A one-click wizard that posts all depreciation entries due up to a date you choose — for every running asset in one go.
- **Usage:** Month-end batch job: instead of posting depreciation lines one by one, run this once and all the month's depreciation hits the books together. (The monthly cron runs the same thing automatically.)
- **Config:** At least one `open` asset with unposted due lines.
- **Workflow:**
  1. `Accounting › Accounting › Generate Entries › Generate Assets Entries` (group: Advisor).
  2. Set *Account Date* (cutoff).
  3. Click *Generate Entries* → a list of the created `account.move` rows opens.

#### Lock Dates — `om_fiscal_year`

- **Definition:** A wizard that freezes the past by setting cut-off dates on the company — once a date is locked, ordinary users simply cannot post or edit anything before it.
- **Usage:** The formal "this period is closed" step. Only Advisors can post into a locked period. It's the guarantee behind every audit sign-off.
- **Config:** A period with postings.
- **Workflow:** `Accounting › Accounting › Lock Dates` (group: Advisor) → set the dates → *Save*.

### 3.6 Review

#### Journal Entries

- **Definition:** The full list of journal-entry *headers* (the parent move, not its individual lines) — every invoice, bill, payment, and manual entry in one grid.
- **Usage:** Use it to find, reopen, or cancel a whole document. When you care about the entry as a unit ("cancel invoice #42"), start here; when you care about individual debit/credit lines, start at *Journal Items*.
- **Config:** Posted or draft moves exist.
- **Workflow:** `Accounting › Review › Control › Journal Entries` → filter by journal/date/state → open a move to see its lines.

#### Logs

- **Definition:** A record of system events — who opened what, who changed what.
- **Usage:** Forensic tracing when something is off and you need to know "who touched this record and when".
- **Workflow:** `Accounting › Review › Logs`.

### 3.7 Reporting

Reports are read-only answers to business questions — "how much did we sell?",
"what do customers owe us?", "did the books balance?". The PDF reports
(`accounting_pdf_reports`) all share the same shape: a small wizard asks for
filters (dates, journals, display options), then a PDF is generated. The *live*
General/Partner Ledgers under **Accounting › Ledgers** are on-screen versions
for quick browsing instead of printing.

```
Wizard pattern (all PDF reports)
   ─────────────────────────────
   1. Open the report menu → wizard opens
   2. Set: date range, target moves (posted/all),
          journals, display option, initial balance, sort
   3. Click the print/export button → PDF
```

#### Financial Reports › Balance Sheet

- **Definition:** A snapshot of what the company *owns* and *owes* on a single date. It always balances: Assets = Liabilities + Owner's Equity.
- **Usage:** The "how healthy are we?" statement for management, banks, and shareholders — taken at a period end.
- **Config:** CoA accounts typed correctly (`asset_*`, `liability_*`, `equity`).
- **Workflow:** `Accounting › Reporting › Financial Reports › Balance Sheet` → wizard: date → print.

#### Financial Reports › Profit and Loss

- **Definition:** A movie (over a period) of income minus expenses — "did we make money this quarter?". Also called the income statement.
- **Usage:** Measures performance. Compare it period over period to see if the business is improving.
- **Config:** Income/expense accounts typed correctly.
- **Workflow:** `Accounting › Reporting › Financial Reports › Profit and Loss` → wizard: date range → print.

#### Partner Reports › Partner Ledger (PDF)

- **Definition:** A printed, per-customer (or per-vendor) breakdown of every unpaid and recently paid invoice, with running balances.
- **Usage:** Sent to a customer to reconcile, or used internally as the collector/AP working document.
- **Config:** Partners with receivable/payable postings.
- **Workflow:** `Accounting › Reporting › Partner Reports › Partner Ledger` → wizard: partner range + dates → print.

#### Partner Reports › Aged Partner Balance

- **Definition:** A list of who owes you (or who you owe), with each amount sorted into "buckets" by how late it is — current, 1–30 days, 31–60, 61–90, 90+.
- **Usage:** The classic credit-control and bad-debt provisioning report. It shows not just *how much* is outstanding but *how stale*.
- **Workflow:** `Accounting › Reporting › Partner Reports › Aged Partner Balance` → wizard: ageing length + dates → print.

#### Partner Reports › Aged Receivable / Aged Payable

- **Definition:** The aged report split by direction — Aged Receivable is just customers (money owed *to* you), Aged Payable is just vendors (money *you* owe).
- **Workflow:** same path under `Reporting › Partner Reports`.

#### Audit Reports › General Ledger (PDF)

- **Definition:** A printable, dated, signed copy of the entire General Ledger for a period — every account, every transaction, in order.
- **Usage:** This is the document an external auditor takes away and signs off against. The PDF is preferred over the live view because it's a fixed snapshot.
- **Config:** Posted period.
- **Workflow:** `Accounting › Reporting › Audit Reports › General Ledger` → wizard: dates + accounts + journals + display + initial balance → print.

#### Audit Reports › Trial Balance

- **Definition:** A one-line-per-account summary showing each account's total debit and credit movement and closing balance. The bottom line proves total debits equal total credits.
- **Usage:** The "did the books balance?" check before closing a period. If debits ≠ credits, something is wrong and must be found before proceeding.
- **Workflow:** `Accounting › Reporting › Audit Reports › Trial Balance` → wizard → print → confirm D = C.

#### Audit Reports › Tax Report

- **Definition:** A summary of every tax you charged or paid during a period, broken down by tax rate — the base amount and the tax amount for each.
- **Usage:** This is the source document for filling a VAT/Mushak return. You take the numbers from here and copy them onto the tax authority's form.
- **Config:** Taxes have correct repartition lines.
- **Workflow:** `Accounting › Reporting › Audit Reports › Tax Report` → wizard: period + taxes → print.

#### Audit Reports › Journals Audit

- **Definition:** A straight dump of every entry in a journal, in sequence-number order, for a period.
- **Usage:** The sequence-number integrity check. Auditors scan it for gaps or duplicates in entry numbers — a gap can mean a deleted or tampered entry, which is a red flag.
- **Config:** A journal with entries.
- **Workflow:** `Accounting › Reporting › Audit Reports › Journals Audit` → wizard: journal + range → print → inspect for gaps.

#### Management › Invoice Analysis

- **Definition:** A pivot/graph view of your invoices — revenue by customer, by product, by month, etc., draggable into whatever shape you need.
- **Usage:** Management reporting without exporting to Excel. Slice sales data any way the business asks for.
- **Workflow:** `Accounting › Reporting › Management › Invoice Analysis` → pivot.

#### Management › Assets — `om_account_asset`

- **Definition:** A pivot/graph view of depreciation — gross value vs already-depreciated vs still-to-post, per asset category and month.
- **Usage:** Answers "what's our fixed-asset base worth right now?" and "how much depreciation will hit the P&L in coming months?".
- **Config:** Assets with depreciation lines.
- **Workflow:** `Accounting › Reporting › Management › Assets` → pivot by category/month → filter by state.

#### Management › Budgets Analysis — `om_account_budget`

- **Definition:** A side-by-side view of each budget line showing what you *planned* to spend vs what you *actually* spent, with a percentage and a time-apportioned "theoretical" amount.
- **Usage:** Variance analysis — the conversation starter for "marketing is 40% over budget, why?". Click a line to jump to the underlying accounting entries.
- **Config:** A validated budget; postings in the period.
- **Workflow:** `Accounting › Reporting › Management › Budgets Analysis` → pivot by budget → click a line → *Budget Entries* jumps to the GL rows behind `practical_amount`.

#### Daily Reports › Day Book — `om_account_daily_reports`

- **Definition:** A daily diary of the company — every transaction posted on a single day, listed together. One page per day in the PDF.
- **Usage:** End-of-day verification. The accountant scans it before going home to confirm nothing was missed or posted by mistake.
- **Config:** Posted entries for the day.
- **Workflow:** `Accounting › Reporting › Daily Reports › Day Book` → wizard: date range + journals + posted/all → *Print PDF* → one page per day.

#### Daily Reports › Cash Book — `om_account_daily_reports`

- **Definition:** A running ledger of every cash movement, with an optional opening balance brought forward — essentially the cash account's bank statement.
- **Usage:** Proves how much cash is on hand and where it went. Used to reconcile the physical cash tin against the books.
- **Config:** A `cash` journal whose `default_account_id` / payment-method accounts exist.
- **Workflow:** `Accounting › Reporting › Daily Reports › Cash Book` → wizard: dates + display (all/movement/not_zero) + sort + *Include Initial Balances* → print.

#### Daily Reports › Bank Book — `om_account_daily_reports`

- **Definition:** The same running ledger, but for each bank account — every deposit and withdrawal in date order with a closing balance.
- **Usage:** The supporting document for bank reconciliation and for handing to auditors as proof of bank movements.
- **Config:** A `bank` journal with bank accounts attached.
- **Workflow:** `Accounting › Reporting › Daily Reports › Bank Book` → wizard → print.

> The cash/bank books default their accounts from each journal's
> `default_account_id` and its inbound/outbound `payment_method_line_ids.payment_account_id`
> (see `_get_default_account_ids`). If those are empty the report prints nothing.

### 3.8 Configuration

Configuration is the Advisor-only area where master data and rules live. You set
these up once and then day-to-day work just references them.

#### Settings

- **Definition:** The master switches for the whole accounting app — multi-currency on/off, anglo-saxon on/off, exchange-rate mode, analytic on/off, etc.
- **Usage:** Come here when you need to enable a feature company-wide. Each toggle unlocks new fields or menus across the app.
- **Workflow:** `Accounting › Configuration › Settings` → toggle → *Save*. The *Anglo-Saxon Accounting* toggle (added by `om_account_accountant/models/settings.py`) lives here.

#### Chart of Accounts

- (Definition/Usage/Config as in §0.3.) **Workflow:** `Accounting › Configuration › Accounting › Chart of Accounts` → edit or archive.

#### Taxes

- (As in §0.5.) **Workflow:** `Accounting › Configuration › Accounting › Taxes`.

#### Journals

- (As in §0.4.) **Workflow:** `Accounting › Configuration › Accounting › Journals`.

#### Asset Category — `om_account_asset`

- **Definition:** A template for a class of assets (e.g. "Computers — 5 year linear") that pre-fills the depreciation method and the accounts to use.
- **Usage:** Set up a category once per asset class. Every individual asset then inherits the rules, so you don't re-enter the same depreciation settings each time.
- **Config:** CoA has `account_asset_id`, `account_depreciation_id`, `account_depreciation_expense_id`; a journal.
- **Workflow:**
  1. `Accounting › Configuration › Accounting › Asset Category` → *New*.
  2. Set *Asset Type* name, type (`purchase`=asset / `sale`=deferred revenue).
  3. Pick the three accounts + journal.
  4. Set *Computation Method* (Linear/Degressive), *Number of Entries*, *One Entry Every N months*.
  5. Optional: *Auto-Confirm Assets*, *Group Journal Entries*.

#### Budgetary Positions — `om_account_budget`

- **Definition:** A named group of accounts that represents one budget concept — e.g. "Marketing" might bundle 5 expense accounts together.
- **Usage:** Lets a budget line target "Marketing" as one concept instead of listing 5 accounts each time. Keeps budgets readable.
- **Config:** CoA exists.
- **Workflow:** `Accounting › Configuration › Accounting › Budgetary Positions` → *New* → name → add `account_ids` (at least one, enforced by `_check_account_ids`) → *Save*.

#### Fiscal Year — `om_fiscal_year`

- **Definition:** An explicit record marking the start and end of a reporting year.
- **Usage:** Used by budgets, depreciation, and reports to bound "this year" consistently across the app.
- **Config:** Company FY month/day set.
- **Workflow:** `Accounting › Configuration › Accounting › Fiscal Year` → *New* → name + date_from + date_to.

#### Account Tags — `om_account_accountant`

- **Definition:** Free-form labels you can stick on accounts or on individual journal lines — e.g. tag every "operating expense" account, even if they live in different parts of the CoA.
- **Usage:** Reports can roll up by tag, so you can build custom report rows that don't follow the chart-of-accounts structure.
- **Config:** None.
- **Workflow:** `Accounting › Configuration › Accounting › Account Tags` → *New* → apply to accounts/lines.

#### Account Groups — `om_account_accountant`

- **Definition:** Parent folders that bundle accounts by code prefix — e.g. everything starting "10…" sits under "Current Assets".
- **Usage:** Makes a large CoA navigable as a tree and gives the Balance Sheet/P&L their roll-up structure.
- **Config:** CoA exists.
- **Workflow:** `Accounting › Configuration › Accounting › Account Groups` → *New* → set code-prefix range.

#### Assets — `om_account_asset`

- **Definition:** An individual fixed asset — a laptop, a machine, a vehicle — with its gross value, salvage value, and a scheduled depreciation plan.
- **Usage:** Each asset tracks how much value is left and posts a little depreciation to the P&L each month, so the books reflect the asset wearing out over time.
- **Config:** Asset Category created.
- **Workflow:**
  1. `Accounting › Configuration › Accounting › Assets` → *New* (or auto-created from a posted vendor bill with `asset_category_id`).
  2. Pick *Asset Category*, set *Gross Value*, *Date*, *Salvage Value*.
  3. *Compute Depreciation* (header button) → board fills.
  4. *Confirm* (`validate()` → state `open`).
  5. Other header buttons (state-driven): *Sell or Dispose* (`set_to_close`), *Set to Draft*, *Modify Depreciation* (wizard `asset.modify`).

#### Budgets — `om_account_budget`

- **Definition:** A planned-spending container for a period — "in FY2026 we plan to spend BDT 5M on Marketing", broken into lines per budgetary position and (optionally) per analytic account.
- **Usage:** The baseline against which actual spend is compared. As the year progresses, every budget line shows planned vs actual vs a time-apportioned "should-be-by-now" amount.
- **Config:** Budgetary Position(s); (optional) analytic accounts.
- **Workflow:**
  1. `Accounting › Configuration › Accounting › Budgets` → *New*.
  2. Set name + date range + responsible.
  3. Add lines: position / analytic / date slice / `planned_amount`.
  4. *Confirm* → *Validate* → *Done*. (Buttons: `action_budget_confirm` / `action_budget_validate` / `action_budget_done` / `action_budget_cancel` / `action_budget_draft`.)

```
   Budget line computed amounts (from account_budget.py)
   ═════════════════════════════════════════════════════
   practical_amount   = SUM(credit − debit) on the position's
                        accounts in the slice (GL or analytic)
   theoritical_amount = planned × (elapsed / total) of the slice
   percentage         = practical / theoretical
```

#### Currencies / Fiscal Positions / Multi-Ledger / Tax Groups / Cash Roundings

- Standard core menus under `Configuration › Accounting`.
- **Currencies:** activate currencies and set exchange rates.
- **Fiscal Positions:** as in §0.5.
- **Multi-Ledger:** `account.journal.group` — group journals for combined views.
- **Tax Groups** / **Cash Roundings:** technical (`base.group_no_one`).

#### Payment Terms — core

- (As in §0.6.) **Workflow:** `Accounting › Configuration › Invoicing › Payment Terms`.

#### Payment Methods — core (technical)

- **Definition:** The catalogue of payment instruments — manual, cheque, SEPA, etc. Read-only here; you pick which ones a journal uses on the journal itself.
- **Workflow:** `Accounting › Configuration › Online Payments › Payment Methods` (only visible in developer mode, `base.group_no_one`).

#### Recurring Template / Recurring Payment — `om_recurring_payments`

- **Definition:** A *template* defines a repeating schedule ("rent of BDT 50k every month"); a *payment* is one instance of that template posting an actual entry.
- **Usage:** Automates entries that repeat identically — rent, subscriptions, monthly accruals — so nobody has to remember to post them each month.
- **Config:** A bank/cash journal (template's `journal_id` must be bank/cash).
- **Workflow:**
  1. `Accounting › Configuration › Recurring Payment › Recurring Template` → *New* → name + journal + recurring_period + recurring_interval → *Confirm*.
  2. Generate payments under `Configuration › Recurring Payment › Recurring Payment`.

#### Analytic Accounting

- Standard core/analytic menus under `Configuration › Analytic Accounting` — Plans, Accounts, Distribution Models. Used by assets and budgets to slice numbers by project/department.

#### Follow-up Levels — `om_account_followup`

- **Definition:** The tiered reminder plan — one per company — that defines what happens at each stage of lateness: a polite email at day 7, a stronger letter at day 30, a manual call at day 60.
- **Usage:** This is the rulebook the dunning run reads from. Configure it once to match your credit policy, then the *Send Letters and Emails* wizard executes it consistently every time.
- **Config:** An email template (the default ships in `data/mail_template_data.xml`).
- **Workflow:**
  1. `Accounting › Configuration › Follow-up › Follow-up Levels`.
  2. On the company's plan, add `followup.line` rows: *Due Days*, *Name* (action label).
  3. Tick *Send an Email* / *Send a Letter* / *Manual Action* per tier.
  4. Pick `email_template_id`, write `manual_action_note`, optionally assign `manual_action_responsible_id`.
  5. Save. Constraints (`models.Constraint`): one plan per company; unique delay per tier.

---

## Appendix — Where to Look in Source

| Need                                    | File (path under repo root)                                                                                       |
| --------------------------------------- | ----------------------------------------------------------------------------------------------------------------- |
| The core menu tree                      | `addons/account/views/account_menuitem.xml`                                                                     |
| How the umbrella renames/menus the top  | `custom_addons/om_account_accountant/views/menu.xml`                                                            |
| How an asset depreciates                | `custom_addons/om_account_asset/models/account_asset.py` (`compute_depreciation_board`, `_compute_entries`) |
| How an asset is born from a vendor bill | `custom_addons/om_account_asset/models/account_move.py` (`AccountMoveLine.asset_create`)                      |
| Asset form buttons                      | `custom_addons/om_account_asset/views/account_asset_views.xml`                                                  |
| How budgets compute practical $         | `custom_addons/om_account_budget/models/account_budget.py` (`_compute_practical_amount`)                      |
| How the day book builds rows            | `custom_addons/om_account_daily_reports/report/report_daybook.py`                                               |
| How overdue partners are detected       | `custom_addons/om_account_followup/wizard/followup_print.py` (`_get_partners_followp`)                        |
| How the overdue letter renders          | `custom_addons/om_account_followup/report/followup_print.py` (`_lines_get_with_partner`)                      |
| How groups are named                    | `custom_addons/om_account_accountant/security/group.xml`                                                        |
| The monthly depreciation cron           | `custom_addons/om_account_asset/data/account_asset_data.xml`                                                    |
| Multi-company rules                     | `custom_addons/om_account_*/security/security.xml`                                                              |

---

*Menu paths in this guide are verified against the installed XML on branch
`om_accounting`. If the OdooMates stack is upgraded, re-read
`account_menuitem.xml` and each module's menu XML — the path labels are the
thing most likely to shift between versions.*
