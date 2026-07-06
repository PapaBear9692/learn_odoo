# Odoo 19 Chart of Accounts (COA) — Training Guide

> Compiled from a ChatGPT learning session. All explanatory text is verbatim from that session; quizzes, answers, score tables, and casual back-and-forth have been removed. Learner questions are kept as framing.
> Source: https://chatgpt.com/share/6a4b4db0-3324-83ee-bb36-37ef207e335b

---

Since you're a software engineer, I'll teach both the **accounting concepts** and the **Odoo implementation details**, because understanding one without the other causes most COA mapping mistakes.

# 📚 Odoo 19 Chart of Accounts (COA) Masterclass

## Course Structure

### Module 1 – Accounting Fundamentals

1. What is accounting?
2. Double-entry bookkeeping
3. Assets, Liabilities, Equity
4. Income and Expenses
5. Debits and Credits

### Module 2 – Chart of Accounts (COA)

1. What is a COA?
2. Account types in Odoo
3. Parent and child accounts
4. Account codes and numbering systems

### Module 3 – Odoo Mapping

1. Product Categories
2. Journals
3. Taxes
4. Receivable & Payable accounts
5. Inventory accounts
6. Manufacturing accounts
7. Localization and Fiscal Positions

### Module 4 – Advanced Topics

1. Multi-company COA
2. Consolidation
3. Custom COA design
4. Common implementation mistakes

---

# Lesson 1 – What is Accounting?

Imagine you own a company.

### Day 1

You invest 100,000 BDT into your company.

What happened?

You now have:

| Thing           | Amount  |
| --------------- | ------- |
| Cash            | 100,000 |
| Owner's Capital | 100,000 |

Notice:

- Cash increased.
- Owner's investment increased.

Everything in accounting follows:

# Assets = Liabilities + Equity

This is called the **Accounting Equation**.

---

## Example 1

You buy a laptop for 50,000 BDT using cash.

Before:

| Assets | Amount  |
| ------ | ------- |
| Cash   | 100,000 |

After:

| Assets | Amount |
| ------ | ------ |
| Cash   | 50,000 |
| Laptop | 50,000 |

Did total assets change?

No.

This is why accounting works.

---

## The Five Account Types

### 1. Assets

Things your company owns.

Examples:

- Cash
- Bank
- Inventory
- Computers
- Vehicles
- Accounts Receivable

---

### 2. Liabilities

Things your company owes.

Examples:

- Loans
- Supplier Payables
- VAT Payable

---

### 3. Equity

Owner's share in the company.

Examples:

- Capital
- Retained Earnings

---

### 4. Revenue (Income)

Money the company earns.

Examples:

- Sales
- Service Revenue

---

### 5. Expenses

Money the company spends.

Examples:

- Salary
- Electricity
- Rent

---

# How Odoo Sees These

| Accounting Type | Odoo Account Types                           |
| --------------- | -------------------------------------------- |
| Asset           | Current Assets, Fixed Assets                 |
| Liability       | Current Liabilities, Non-current Liabilities |
| Equity          | Equity                                       |
| Revenue         | Income                                       |
| Expense         | Expenses                                     |

---

# Very Important Concept

Every transaction affects **at least two accounts**.

This is called:

# Double Entry Accounting

Example:

Sold product for cash worth 10,000.

| Account       | Debit  | Credit |
| ------------- | ------ | ------ |
| Cash          | 10,000 |        |
| Sales Revenue |        | 10,000 |

Debits always equal Credits.

---

# Why This Matters for Odoo

Every time you:

- create an invoice,
- validate a bill,
- receive inventory,
- pay a vendor,

Odoo automatically creates these journal entries.

If your COA mapping is wrong, Odoo posts entries into the wrong accounts and your financial reports become incorrect.

---

> **Learner question:** I have a question. I understand that selling increases (debit) my cash, but how does it decrease (credit) my sales revenue?

Let's fix the foundation first.

# The Biggest Beginner Mistake

People think:

> Debit = Increase
> Credit = Decrease

This is **wrong**.

Debits and credits do **not** mean increase or decrease by themselves.

They mean **left side** and **right side** of an account.

Whether they increase or decrease depends on the account type.

---

# The Golden Rule

| Account Type | Increase | Decrease |
| ------------ | -------- | -------- |
| Assets       | Debit    | Credit   |
| Expenses     | Debit    | Credit   |
| Liabilities  | Credit   | Debit    |
| Equity       | Credit   | Debit    |
| Revenue      | Credit   | Debit    |

This table is extremely important.

---

# Your Question

> How does crediting Sales Revenue decrease it?

It doesn't.

In a **Revenue account**, a **Credit increases** the balance.

So:

| Account       | Debit  | Credit |
| ------------- | ------ | ------ |
| Cash          | 10,000 |        |
| Sales Revenue |        | 10,000 |

means:

- Cash (Asset) increased by 10,000 because Assets increase with Debit.
- Sales Revenue (Revenue) increased by 10,000 because Revenue increases with Credit.

Nothing decreased.

---

# Why Does Revenue Increase on the Credit Side?

Because of the accounting equation:

```text
Assets = Liabilities + Equity
```

Revenue eventually increases the owner's wealth (Equity).

Think of it like this:

```text
Revenue → Profit → Equity
```

Since Equity increases with **Credit**, Revenue also increases with **Credit**.

---

# Example

You started with:

```text
Cash = 100,000
Equity = 100,000
```

You sell goods for cash worth 10,000.

Now:

```text
Cash = 110,000
Revenue = 10,000
```

At the end of the year:

```text
Revenue → Retained Earnings → Equity
```

So:

```text
Cash = 110,000
Equity = 110,000
```

The equation still balances.

---

# Memory Trick

```text
DEA = Debit increases
Assets
Expenses

CLE = Credit increases
Liabilities
Equity
Revenue
```

Or simply:

```text
DEALER

D -> Dividends (ignore for now)
E -> Expenses
A -> Assets

L -> Liabilities
E -> Equity
R -> Revenue
```

Left side (Debit): DEAL
Right side (Credit): LER

---

# Why Salary Expense is Debit

Expenses reduce your profit and therefore reduce Equity.

To keep the accounting equation balanced, Expenses are given their own category that **increases with Debits**.

Example:

Pay salary of 20,000.

| Account        | Debit  | Credit |
| -------------- | ------ | ------ |
| Salary Expense | 20,000 |        |
| Cash           |        | 20,000 |

Result:

- Expense increased.
- Cash decreased.

---

# Why Owner's Capital is Credit

Owner's Capital is an **Equity** account.

Example:

Owner invests 100,000.

| Account         | Debit   | Credit  |
| --------------- | ------- | ------- |
| Cash            | 100,000 |         |
| Owner's Capital |         | 100,000 |

Result:

- Asset increased.
- Equity increased.

---

# A Better Way to Understand Debits & Credits

The accounting equation:

```text
Assets = Liabilities + Equity
```

Expanded version:

```text
Assets + Expenses = Liabilities + Equity + Revenue
```

Notice:

### Left Side (Debit-Increase)

- Assets
- Expenses

### Right Side (Credit-Increase)

- Liabilities
- Equity
- Revenue

This is why the rule works.

---

# Journal Entry Examples

### Example 1 – Buy inventory with cash for 5,000

| Account   | Debit | Credit |
| --------- | ----- | ------ |
| Inventory | 5,000 |        |
| Cash      |       | 5,000  |

---

### Example 2 – Buy inventory on credit

| Account          | Debit | Credit |
| ---------------- | ----- | ------ |
| Inventory        | 5,000 |        |
| Accounts Payable |       | 5,000  |

---

### Example 3 – Customer pays invoice

| Account             | Debit  | Credit |
| ------------------- | ------ | ------ |
| Cash                | 10,000 |        |
| Accounts Receivable |        | 10,000 |

---

# Important for Odoo

When you validate an invoice, Odoo is literally creating journal entries like these behind the scenes.

Understanding these entries is the key to correctly mapping the COA.

---

You now understand the most important foundation of accounting:

- Debit and Credit are **sides**, not increase/decrease.
- Whether they increase or decrease depends on the **account type**.
- Every transaction affects **at least two accounts** and must balance.

Now we can finally talk about the **Chart of Accounts (COA)**.

---

# Lesson 3 – What is a Chart of Accounts?

A **Chart of Accounts (COA)** is simply:

> A list of all the accounts your company uses to record transactions.

Think of it as a database schema for accounting.

In software terms:

```text
Transaction
     ↓
Journal Entry
     ↓
Accounts (COA)
     ↓
Financial Reports
```

Without a proper COA, Odoo cannot generate:

- Balance Sheet
- Profit & Loss
- Cash Flow
- Tax Reports

---

# Example COA

| Code   | Account Name        | Type      |
| ------ | ------------------- | --------- |
| 101000 | Cash                | Asset     |
| 102000 | Bank                | Asset     |
| 110000 | Accounts Receivable | Asset     |
| 120000 | Inventory           | Asset     |
| 201000 | Accounts Payable    | Liability |
| 210000 | VAT Payable         | Liability |
| 300000 | Owner's Capital     | Equity    |
| 400000 | Sales Revenue       | Income    |
| 500000 | Salary Expense      | Expense   |
| 510000 | Rent Expense        | Expense   |

---

# Why Account Codes?

Humans read:

```text
Cash
```

Computers like:

```text
101000
```

Account codes help:

- organize accounts
- generate reports
- make searching easier
- maintain standards

---

# Typical Numbering Convention

## Assets

```text
100000 - 199999
```

## Liabilities

```text
200000 - 299999
```

## Equity

```text
300000 - 399999
```

## Revenue

```text
400000 - 499999
```

## Expenses

```text
500000 - 599999
```

This is not mandatory, but it is very common.

---

# How Odoo Uses the COA

Every account in Odoo has:

1. Code
2. Name
3. Account Type
4. Allow Reconciliation (optional)
5. Currency (optional)

Example:

| Code   | Name                | Type       |
| ------ | ------------------- | ---------- |
| 110100 | Accounts Receivable | Receivable |
| 210100 | Accounts Payable    | Payable    |
| 401000 | Sales               | Income     |
| 501000 | Salary Expense      | Expense    |

---

# Very Important: Account Type vs Account Name

This is where many Odoo developers get confused.

These two are different:

| Name            | Type                  |
| --------------- | --------------------- |
| "Employee Loan" | Current Asset         |
| "VAT Payable"   | Current Liability     |
| "Bank Loan"     | Non-current Liability |

Odoo doesn't care much about the **name**.

Odoo cares a lot about the **Account Type**.

The account type determines:

- Which report it appears on
- Whether reconciliation is allowed
- Whether it's receivable/payable
- Whether it affects cash flow
- Whether year-end closing behaves correctly

---

# Example of Wrong Mapping

Suppose:

```text
Salary Expense
```

is created as an **Asset**.

Then Odoo thinks:

> "The company owns salaries."

Which makes no accounting sense.

Your Balance Sheet and P&L become wrong.

---

# Odoo 19 Important Account Types

| Odoo Type               | Category  |
| ----------------------- | --------- |
| Receivable              | Asset     |
| Bank and Cash           | Asset     |
| Current Assets          | Asset     |
| Fixed Assets            | Asset     |
| Payable                 | Liability |
| Credit Card             | Liability |
| Current Liabilities     | Liability |
| Non-current Liabilities | Liability |
| Equity                  | Equity    |
| Current Year Earnings   | Equity    |
| Income                  | Revenue   |
| Other Income            | Revenue   |
| Expenses                | Expense   |
| Depreciation            | Expense   |
| Direct Costs            | Expense   |

You will use these constantly in implementations.

---

# Real Example

A customer invoice for 10,000 creates:

| Account             | Debit  | Credit |
| ------------------- | ------ | ------ |
| Accounts Receivable | 10,000 |        |
| Sales Revenue       |        | 10,000 |

How does Odoo know:

- AR goes to Balance Sheet?
- Sales goes to P&L?

Because of the **Account Type**.

---

# Why Account Type Matters More Than Code

The account code is mostly for:

- Organization
- Searching
- Company standards
- Import/export

Odoo can generate perfectly correct financial statements even if your codes are:

```text
1
2
3
4
5
```

As long as the **Account Types** are correct.

But if the types are wrong:

```text
401000 Sales Revenue → Current Asset
```

your reports become completely wrong.

---

# Lesson 4 – COA Mapping in Odoo

This is where most implementations fail.

---

# Think of Odoo Like This

```text
Business Event
       ↓
Journal Entry
       ↓
Chart of Accounts
       ↓
Financial Reports
```

Your job as an implementer is:

> "For every business event, which accounts should be debited and credited?"

---

# Business Flow #1 – Customer Invoice

You sell goods worth 10,000.

Invoice Validation:

| Account             | Debit  | Credit |
| ------------------- | ------ | ------ |
| Accounts Receivable | 10,000 |        |
| Sales Revenue       |        | 10,000 |

---

# Mapping Required

### Accounts Receivable

Type:

```text
Receivable
```

### Sales Account

Type:

```text
Income
```

---

# Business Flow #2 – Customer Payment

Customer pays 10,000.

| Account             | Debit  | Credit |
| ------------------- | ------ | ------ |
| Bank                | 10,000 |        |
| Accounts Receivable |        | 10,000 |

---

# Mapping Required

### Bank Account

Type:

```text
Bank and Cash
```

---

# Business Flow #3 – Vendor Bill

Supplier sends a bill for office supplies.

| Account          | Debit | Credit |
| ---------------- | ----- | ------ |
| Office Expense   | 5,000 |        |
| Accounts Payable |       | 5,000  |

---

# Mapping Required

### Expense Account

Type:

```text
Expenses
```

### Payable Account

Type:

```text
Payable
```

---

# Business Flow #4 – Pay Supplier

| Account          | Debit | Credit |
| ---------------- | ----- | ------ |
| Accounts Payable | 5,000 |        |
| Bank             |       | 5,000  |

---

# Business Flow #5 – Inventory Purchase

This is where people usually get confused.

When you buy inventory, should Odoo do this?

| Inventory | Dr |
| Payable | Cr |

Or:

| Expense | Dr |
| Payable | Cr |

The answer is:

> **It depends on the inventory valuation method.**

In modern Odoo implementations:

### Automated Valuation

```text
Inventory Asset     Dr
Accounts Payable    Cr
```

The expense is recognized later when the product is sold.

---

# Business Flow #6 – Selling Inventory

Suppose:

Selling Price = 10,000

Cost = 6,000

Odoo creates TWO entries.

### Revenue Entry

| Account             | Debit  | Credit |
| ------------------- | ------ | ------ |
| Accounts Receivable | 10,000 |        |
| Sales Revenue       |        | 10,000 |

### Cost Entry

| Account            | Debit | Credit |
| ------------------ | ----- | ------ |
| Cost of Goods Sold | 6,000 |        |
| Inventory          |       | 6,000  |

---

# Therefore, every stockable product usually needs these accounts:

### Inventory Valuation Account

Type:

```text
Current Assets
```

### Stock Input Account

Type:

```text
Current Assets
```

### Stock Output Account

Type:

```text
Current Assets
```

### Cost of Goods Sold

Type:

```text
Direct Costs
```

---

# The Three Inventory Accounts

The three accounts are **not based on inventory status** (available, ordered, to be delivered).

They are used to handle **timing differences between inventory movements and accounting documents (vendor bills/customer invoices).**

---

## 1. Inventory Valuation Account

This is easy.

It represents:

> "How much inventory do we currently own?"

Example:

| Product | Quantity | Cost   |
| ------- | -------- | ------ |
| Laptop  | 10       | 50,000 |

Inventory value:

```text
10 × 50,000 = 500,000
```

Journal:

| Account             | Debit   | Credit |
| ------------------- | ------- | ------ |
| Inventory Valuation | 500,000 |        |

This appears on the Balance Sheet as an Asset.

---

# 2. Stock Input Account

This is a **temporary clearing account**.

It answers:

> "We have received the goods, but accounting is not finished yet."

---

## Example

You order 100 laptops from a supplier.

### Step 1: Receive the goods

Warehouse receives inventory.

Odoo creates:

| Account             | Debit   | Credit  |
| ------------------- | ------- | ------- |
| Inventory Valuation | 100,000 |         |
| Stock Input         |         | 100,000 |

Notice:

- Inventory increased.
- But we haven't processed the vendor bill yet.

So Odoo needs a temporary account.

---

### Step 2: Vendor bill arrives

| Account          | Debit   | Credit  |
| ---------------- | ------- | ------- |
| Stock Input      | 100,000 |         |
| Accounts Payable |         | 100,000 |

Stock Input becomes:

```text
-100,000 + 100,000 = 0
```

Perfect.

---

# Why not directly credit Accounts Payable?

Because:

**Warehouse people receive products.**

**Accounting people validate bills later.**

The two events happen at different times.

Odoo must keep accounting correct even if they happen days apart.

---

# 3. Stock Output Account

This is another temporary account.

It answers:

> "The goods have left the warehouse, but invoicing is not finished yet."

---

## Example

You deliver goods to the customer first.

### Delivery

| Account             | Debit  | Credit |
| ------------------- | ------ | ------ |
| Stock Output        | 60,000 |        |
| Inventory Valuation |        | 60,000 |

---

### Later, invoice customer

Depending on the accounting method, Odoo clears this account.

Again, it acts as a bridge between:

- Inventory movement
- Sales accounting

---

# Software Engineer Analogy

Think of:

```text
Stock Input
Stock Output
```

as **message queues** or **staging tables**.

```text
Warehouse Event
      ↓
Temporary Account
      ↓
Accounting Document
```

The temporary account keeps everything balanced until the second event happens.

---

# Summary

| Account             | Purpose                                     |
| ------------------- | ------------------------------------------- |
| Inventory Valuation | Value of inventory you own                  |
| Stock Input         | Goods received but not fully accounted for  |
| Stock Output        | Goods delivered but not fully accounted for |

---

# Let's go one level deeper

Why doesn't Odoo simply do this?

```text
Receive Goods:
Dr Inventory
Cr Accounts Payable
```

Because Odoo allows these events to happen independently:

### Scenario 1

```text
Receive Goods → Vendor Bill later
```

### Scenario 2

```text
Vendor Bill → Receive Goods later
```

### Scenario 3

```text
Partial Receipt → Partial Bill
```

### Scenario 4

```text
Multiple Receipts → One Bill
```

The clearing account (**Stock Input**) handles all these situations elegantly.

---

# Example: Bill Comes First

Suppose:

1. Vendor bill arrives for 100,000.
2. Goods haven't arrived yet.

Vendor Bill:

| Account          | Debit   | Credit  |
| ---------------- | ------- | ------- |
| Stock Input      | 100,000 |         |
| Accounts Payable |         | 100,000 |

Stock Input balance:

```text
+100,000
```

Later, goods arrive:

| Account             | Debit   | Credit  |
| ------------------- | ------- | ------- |
| Inventory Valuation | 100,000 |         |
| Stock Input         |         | 100,000 |

Stock Input becomes:

```text
+100,000 -100,000 = 0
```

Beautiful.

---

# This is why it's called a Clearing Account

Think of it as:

```text
Warehouse Module  ←→ Stock Input ←→ Accounting Module
```

It temporarily holds the value until both sides are complete.

---

# Where Are These Accounts Configured?

Most inventory accounts are configured in:

```text
Inventory
    ↓
Configuration
    ↓
Product Categories
```

This is extremely important.

The **Product Category**, not the product itself, usually controls the accounting.

---

# Example Category

## Electronics

| Setting             | Account                      |
| ------------------- | ---------------------------- |
| Inventory Valuation | Inventory Asset              |
| Stock Input         | Goods Received Not Invoiced  |
| Stock Output        | Goods Delivered Not Invoiced |
| Expense Account     | Cost of Goods Sold           |
| Income Account      | Sales Revenue                |

Every product in this category inherits these accounts.

---

# Why Use Categories?

Imagine:

- 10,000 products
- Same accounting rules

Would you configure each product manually?

No.

You configure once on the category.

---

# Another Important Concept

## Product Type Matters

### Consumable

Examples:

- Pens
- Paper
- Cleaning supplies

Usually:

```text
No inventory valuation.
No stock journal entries.
```

---

### Stockable Product

Examples:

- Laptop
- Phone
- Car parts

Usually:

```text
Inventory Valuation
Stock Input
Stock Output
COGS
```

---

### Service

Examples:

- Consulting
- Installation
- Training

Usually:

```text
No inventory accounts.
Only Income and Expense accounts.
```

---

# Odoo Mapping Cheat Sheet

## Customer Invoice

```text
Dr Accounts Receivable
Cr Sales Revenue
```

---

## Customer Payment

```text
Dr Bank
Cr Accounts Receivable
```

---

## Vendor Bill

```text
Dr Expense or Stock Input
Cr Accounts Payable
```

---

## Vendor Payment

```text
Dr Accounts Payable
Cr Bank
```

---

## Receive Stock

```text
Dr Inventory Valuation
Cr Stock Input
```

---

## Deliver Stock

```text
Dr Stock Output
Cr Inventory Valuation
```

---

> **Learner question:** So inventory needs 3 accounts, same as banks need a suspense account? The money is supposed to be at the bank, I have the money in cash but I have not deposited it yet, so does it also stay in the suspense account?

# Inventory Accounts vs Suspense Accounts

Both are examples of **temporary holding (clearing) accounts**.

They exist because:

```text
Event A happened
Event B hasn't happened yet
```

The accounting system needs somewhere to temporarily store the value.

---

# Inventory Example

### Goods received today

```text
Warehouse receives inventory
```

### Vendor bill comes next week

```text
Accounting receives bill
```

So Odoo uses:

```text
Stock Input
```

to hold the value temporarily.

---

# Banking Example

Suppose you received cash from a customer.

| Account             | Debit  | Credit |
| ------------------- | ------ | ------ |
| Cash                | 10,000 |        |
| Accounts Receivable |        | 10,000 |

Now, two days later, you deposit it into the bank.

| Account | Debit  | Credit |
| ------- | ------ | ------ |
| Bank    | 10,000 |        |
| Cash    |        | 10,000 |

No suspense account is needed because you already know where the money is:

```text
It is physically in your cash box.
```

---

# When Do Suspense Accounts Come In?

Suppose your bank statement says:

```text
Deposit received: 10,000
```

But you don't know:

- Which customer paid?
- Which invoice does it belong to?

You cannot post:

```text
Bank
    ↓
Accounts Receivable
```

because you don't know the customer yet.

So you do:

| Account          | Debit  | Credit |
| ---------------- | ------ | ------ |
| Bank             | 10,000 |        |
| Suspense Account |        | 10,000 |

Later, when you identify the customer:

| Account             | Debit  | Credit |
| ------------------- | ------ | ------ |
| Suspense Account    | 10,000 |        |
| Accounts Receivable |        | 10,000 |

The Suspense Account goes back to zero.

---

# This is very similar to Stock Input

```text
Goods Received
        ↓
   Stock Input
        ↓
Vendor Bill
```

and

```text
Bank Statement Imported
        ↓
   Suspense Account
        ↓
Customer Payment Identified
```

---

# In Odoo

When you configure a bank journal, you'll see:

```text
Outstanding Receipts
Outstanding Payments
Suspense Account
```

These are also temporary accounts.

---

# Even More Accurate Comparison

Inventory accounts are actually closer to:

- Outstanding Receipts
- Outstanding Payments
- Goods Received Not Invoiced (GRNI)
- Goods Delivered Not Invoiced (GDNI)

than to a pure Suspense Account.

Because:

| Account          | Purpose                               |
| ---------------- | ------------------------------------- |
| Stock Input      | Waiting for another business document |
| Stock Output     | Waiting for another business document |
| Suspense Account | Waiting for identification/correction |

---

# Think Like a Developer

Any time you see:

```text
Process A and Process B happen independently
```

you should expect:

```text
Temporary/Clearing Account
```

Examples:

| Module    | Clearing Account     |
| --------- | -------------------- |
| Inventory | Stock Input          |
| Inventory | Stock Output         |
| Banking   | Outstanding Receipts |
| Banking   | Outstanding Payments |
| Banking   | Suspense Account     |
| Payroll   | Salary Payable       |
| Taxes     | VAT Payable          |

---

Whenever two business events can happen independently, you often need a clearing account.

---

# Lesson 5 – Real COA Mapping in Odoo

Now we'll map the actual Odoo modules to the COA.

# 1. Sales Module

## Customer Invoice

Customer buys goods worth 10,000.

```text
Dr Accounts Receivable
Cr Sales Revenue
```

### Mapping

| Purpose             | Account Type |
| ------------------- | ------------ |
| Accounts Receivable | Receivable   |
| Sales Revenue       | Income       |

---

## Customer Payment

```text
Dr Bank
Cr Accounts Receivable
```

### Mapping

| Purpose | Account Type  |
| ------- | ------------- |
| Bank    | Bank and Cash |
| AR      | Receivable    |

---

# 2. Purchase Module

## Vendor Bill

Office supplies for 5,000.

```text
Dr Office Expense
Cr Accounts Payable
```

### Mapping

| Purpose | Account Type |
| ------- | ------------ |
| Expense | Expenses     |
| AP      | Payable      |

---

## Vendor Payment

```text
Dr Accounts Payable
Cr Bank
```

---

# 3. Inventory Module

## Receive Goods

```text
Dr Inventory Valuation
Cr Stock Input
```

---

## Deliver Goods

```text
Dr Stock Output
Cr Inventory Valuation
```

---

# 4. Taxes

Suppose you sell for 10,000 with 15% VAT.

Customer owes:

```text
10,000 + 1,500 = 11,500
```

Journal:

| Account             | Debit  | Credit |
| ------------------- | ------ | ------ |
| Accounts Receivable | 11,500 |        |
| Sales Revenue       |        | 10,000 |
| VAT Payable         |        | 1,500  |

---

### Mapping

| Account     | Type              |
| ----------- | ----------------- |
| VAT Payable | Current Liability |

---

# Why?

The VAT is not your money.

You collected it on behalf of the government.

---

# Purchase VAT

Suppose:

Purchase = 10,000

VAT = 1,500

Journal:

| Account           | Debit  | Credit |
| ----------------- | ------ | ------ |
| Expense/Inventory | 10,000 |        |
| VAT Receivable    | 1,500  |        |
| Accounts Payable  |        | 11,500 |

---

### Mapping

| Account        | Type          |
| -------------- | ------------- |
| VAT Receivable | Current Asset |

---

# 5. Product Category Mapping

This is where most Odoo implementations happen.

Example:

## Electronics Category

### Income Account

```text
Sales Revenue
```

### Expense Account

```text
COGS
```

### Inventory Valuation

```text
Inventory Asset
```

### Stock Input

```text
Goods Received Not Invoiced
```

### Stock Output

```text
Goods Delivered Not Invoiced
```

---

# Example COA for a Trading Company

## Assets

```text
101000 Cash
102000 Bank
110000 Accounts Receivable
120000 Inventory
130000 VAT Receivable
```

---

## Liabilities

```text
201000 Accounts Payable
210000 VAT Payable
220000 Bank Loan
```

---

## Equity

```text
301000 Owner Capital
302000 Retained Earnings
```

---

## Income

```text
401000 Sales Revenue
402000 Other Income
```

---

## Expenses

```text
501000 Cost of Goods Sold
502000 Salary Expense
503000 Rent Expense
504000 Utility Expense
```

---

# How Odoo Decides Which Account to Use

This is extremely important.

The precedence is usually:

```text
Product
    ↓
Product Category
    ↓
Company Default
```

If an account is set on the product, Odoo uses that.

Otherwise:

```text
Product Category
```

Otherwise:

```text
Company default accounts
```

---

> **Learner question:** I need explanation of why these accounts are there so that I can understand it.

# First Principle

A Chart of Accounts exists because we want to answer questions about the business.

For example:

### Question 1

> How much cash do we have?

→ Need a **Cash** account.

---

### Question 2

> How much do customers owe us?

→ Need an **Accounts Receivable** account.

---

### Question 3

> How much do we owe suppliers?

→ Need an **Accounts Payable** account.

---

### Question 4

> How much inventory do we own?

→ Need an **Inventory Asset** account.

---

### Question 5

> How much profit did we make?

→ Need **Revenue** and **Expense** accounts.

---

# So the real question is:

> "What information do I need from my business?"

The COA exists to answer that.

---

# Why does Accounts Receivable exist?

Suppose:

You sold a laptop today.

Customer says:

> "I'll pay next month."

Did you earn revenue?

**Yes.**

Did you receive cash?

**No.**

If you only had a Cash account, your books would say:

```text
Revenue = 10,000
Cash = 0
```

But something is missing.

You have a legal right to receive 10,000.

That right is an asset.

Therefore:

```text
Accounts Receivable
```

exists.

---

# Think of AR as:

```text
Money people owe me.
```

---

# Why does Accounts Payable exist?

You bought inventory.

Supplier says:

> "Pay me in 30 days."

Did inventory increase?

Yes.

Did cash decrease?

No.

So where did the money come from?

You now owe the supplier.

That obligation needs an account.

Hence:

```text
Accounts Payable
```

---

# Think of AP as:

```text
Money I owe others.
```

---

# Why does Sales Revenue exist?

You need to answer:

```text
How much money did my business earn?
```

You cannot put sales into Cash because:

- some customers pay later
- some pay immediately
- some pay partially

Revenue must exist independently from cash.

---

# Why do Expense Accounts exist?

Question:

```text
How much did it cost to run the business?
```

Examples:

- Salaries
- Rent
- Electricity
- Internet

Without expense accounts, you cannot calculate profit.

---

# Why does Inventory Asset exist?

This one is huge.

Suppose:

You buy 1,000 laptops.

Cost:

```text
50,000,000
```

Should you immediately record:

```text
Expense = 50,000,000
```

No.

Why?

Because you haven't sold them yet.

You still own them.

They still have value.

They are an asset.

Therefore:

```text
Inventory Asset
```

exists.

---

# This is one of the biggest accounting principles:

## Expenses happen when value is consumed.

Not when purchased.

---

# Example

Today:

Bought laptop.

Tomorrow:

Still sitting in warehouse.

Did you consume it?

No.

So it is:

```text
Asset
```

---

# Later:

Sold laptop.

Now it left the company.

Now you consumed that asset.

Now it becomes:

```text
Cost of Goods Sold (Expense)
```

---

# This is why COGS exists.

Question:

```text
How much inventory did I consume to generate revenue?
```

---

# Why does Stock Input exist?

This is where ERP systems become interesting.

Question:

Can these happen at different times?

1. Receive goods.
2. Receive vendor bill.

Absolutely.

Accounting must remain correct during the gap.

Therefore:

```text
Stock Input
```

exists.

It answers:

> "Inventory has arrived, but the accounting document hasn't."

---

# Why does Stock Output exist?

Same idea.

Question:

Can these happen separately?

1. Deliver goods.
2. Create customer invoice.

Yes.

Therefore:

```text
Stock Output
```

exists.

It answers:

> "Goods have left, but sales accounting isn't finished."

---

# Why does VAT Payable exist?

Suppose:

Customer pays:

```text
11,500
```

Product:

```text
10,000
```

VAT:

```text
1,500
```

Is the entire 11,500 your revenue?

No.

The government owns the 1,500.

You are merely holding it temporarily.

Therefore:

```text
VAT Payable
```

exists.

---

# Why does Bank exist separately from Cash?

Question:

Can money be:

- in your pocket?
- in a cash drawer?
- in a bank account?

Yes.

Management wants to know:

```text
Cash on hand
vs
Money in bank
```

Therefore:

- Cash account
- Bank account

---

# Why do COAs get so big?

Because every account answers a different business question.

| Question                   | Account            |
| -------------------------- | ------------------ |
| How much cash?             | Cash               |
| How much in bank?          | Bank               |
| How much inventory?        | Inventory          |
| How much customers owe us? | AR                 |
| How much we owe suppliers? | AP                 |
| How much did we sell?      | Sales              |
| How much did we spend?     | Expenses           |
| How much tax do we owe?    | VAT Payable        |
| How much profit?           | Revenue - Expenses |

---

# The Mental Model I Want You to Build

When someone asks:

> "Why does this account exist?"

Never think:

> "Because Odoo needs it."

Instead ask:

> **"What business question is this account trying to answer?"**

If you can answer that, you can design a COA for almost any company.

---

# Cost of Goods Sold (COGS)

The subtle issue is the word **spend**.

You may have bought the laptop 6 months ago.

The cash was already spent back then.

COGS asks:

> **How much inventory value was consumed to generate this revenue?**

Example:

You bought 100 laptops in January.

You sold 1 laptop today.

COGS doesn't care when you spent the money.

It only cares:

```text
How much inventory value left the company because of this sale?
```

---

# Stock Input

The accounting perspective is slightly different:

> **What is the value difference between goods receipts and vendor bills?**

Because the account can also be positive in the opposite direction.

Example:

### Bill first

```text
Vendor Bill
↓
Dr Stock Input
Cr AP
```

### Goods arrive later

```text
Dr Inventory
Cr Stock Input
```

So Stock Input is not literally:

```text
Unbilled inventory
```

It is more accurately:

```text
Temporary account that reconciles goods receipts and vendor bills.
```

Think:

```text
Warehouse ↔ Stock Input ↔ Accounting
```

---

# Now we are ready for a very important topic:

# Why do we even need COGS?

This leads us directly to:

# Anglo-Saxon vs Continental Accounting

This is one of the biggest "aha!" moments in Odoo.

---

# Scenario

You buy a laptop for $600.

Three months later, you sell it for $1,000.

When should the $600 become an expense?

### Option A

The moment you purchase the laptop.

### Option B

The moment you sell the laptop.

Your answer will determine whether you naturally think in a **Continental** or an **Anglo-Saxon** accounting model, and then Odoo's implementation will suddenly make a lot more sense.

---

> The moment I sell the laptop — until then it's an asset, actually.
>
> **Learner question:** Now explain the mapping. When invoicing a customer, which COA do I include there, and why am I including that specific COA? For all the places where I need a COA, explain it like this.

# First Principle

In Odoo, a business process is often split into multiple actions:

```text
Sale Order
↓
Delivery
↓
Invoice
↓
Payment
```

Each step answers a different business question and therefore creates different journal entries.

---

# Your Example

> When invoicing, I think I should: Dr Cash, Cr Stock Output

This is incorrect because **invoicing does not mean:**

1. Customer paid.
2. Inventory left.

These are separate business events.

---

# Event 1: Delivery

Question:

> Did inventory leave the company?

Yes.

Therefore:

```text
Dr Stock Output
Cr Inventory Valuation
```

Why?

Inventory Asset decreased.

But we haven't invoiced the customer yet, so we use the temporary clearing account:

```text
Stock Output
```

---

# Event 2: Customer Invoice

Question:

> Did we earn revenue?

Yes.

Question:

> Did customer pay?

Not necessarily.

Most invoices are on credit.

Therefore:

```text
Dr Accounts Receivable
Cr Sales Revenue
```

NOT:

```text
Dr Cash
Cr Sales Revenue
```

because cash may come later.

---

# Event 3: Customer Payment

Question:

> Did we receive money?

Yes.

Therefore:

```text
Dr Bank/Cash
Cr Accounts Receivable
```

---

# Why Not Debit Cash During Invoice?

Imagine this:

Today:

```text
Invoice customer for 100,000
```

Customer pays:

```text
30 days later.
```

If you debit cash during invoicing, your books would say:

```text
Bank = +100,000
```

even though you never received any money.

That would be wrong.

---

# The Biggest Rule in Odoo

## Journal entries follow business events.

Not what you expect to happen.

---

# Now let's go through every place where Odoo asks for a COA.

---

# 1. Customer (Partner)

### Receivable Account

Question:

> If this customer owes me money, where should I store it?

Usually:

```text
Accounts Receivable
```

Type:

```text
Receivable
```

---

### Payable Account

Question:

> If I owe this partner money, where should I store it?

Usually:

```text
Accounts Payable
```

Type:

```text
Payable
```

---

# Why are these on the partner?

Because different customers can have different receivable accounts.

Example:

```text
Domestic Customers AR
Foreign Customers AR
Employee Receivables
```

---

# 2. Product Category

This is the most important mapping.

---

## Income Account

Question:

> When I sell this product, where should revenue go?

Example:

```text
Sales Revenue
```

---

## Expense Account

Question:

> When this product becomes an expense, where should the cost go?

Example:

```text
Cost of Goods Sold
```

---

## Inventory Valuation

Question:

> Where should I keep the value of inventory I currently own?

Example:

```text
Inventory Asset
```

---

## Stock Input

Question:

> Where should I temporarily store the value difference between goods receipt and vendor bill?

Example:

```text
Goods Received Not Invoiced
```

---

## Stock Output

Question:

> Where should I temporarily store the value difference between delivery and customer invoice?

Example:

```text
Goods Delivered Not Invoiced
```

---

# 3. Sales Journal

Question:

> Which journal should contain customer invoice entries?

Usually:

```text
Customer Invoices
```

This is a journal, not an account.

A common beginner mistake is confusing:

```text
Journal
```

with

```text
Account
```

---

# 4. Bank Journal

### Liquidity Account

Question:

> Where is my money physically located?

Example:

```text
Bank Account
Cash Account
```

---

### Outstanding Receipts

Question:

> Payment initiated but not yet on the bank statement.

---

### Outstanding Payments

Question:

> Payment sent but not yet cleared by the bank.

---

### Suspense Account

Question:

> I know money moved, but I don't know what it belongs to yet.

---

# 5. Taxes

### Sales Tax Account

Question:

> Where should the VAT I collected be stored?

Example:

```text
VAT Payable
```

---

### Purchase Tax Account

Question:

> Where should the VAT I can reclaim be stored?

Example:

```text
VAT Receivable
```

---

# 6. Employee Expense Accounts

Question:

> Employee bought something for the company.

Where do I owe the employee money?

Example:

```text
Employee Payable
```

---

# 7. Fixed Assets

Question:

> I bought a laptop for company use.

Should this go to:

```text
Expense?
```

No.

It will be used for years.

Therefore:

```text
Computer Equipment Asset
```

---

# A Better Mental Model

Whenever Odoo asks you to choose an account, ask yourself these three questions:

### Question 1

```text
What business event happened?
```

### Question 2

```text
What value changed?
```

### Question 3

```text
Which account should remember this change?
```

---

> **Learner question:** But for sales, first we update Stock Output. When the payment was complete, Stock Output was never emptied. When does that happen?

# When Does Stock Output Clear? (Continental vs Anglo-Saxon)

This question leads directly to the difference between **Continental Accounting** and **Anglo-Saxon Accounting** in Odoo.

Your observation is absolutely correct:

```text
Delivery:
Dr Stock Output
Cr Inventory Valuation
```

At this point:

```text
Stock Output = +60,000
```

Then:

```text
Invoice:
Dr Accounts Receivable
Cr Sales Revenue
```

Nothing touched Stock Output.

So when does it become zero?

The answer is:

> **It depends on whether your company uses Continental or Anglo-Saxon accounting.**

---

# 1. Continental Accounting (Default in many countries)

On delivery:

| Account             | Debit  | Credit |
| ------------------- | ------ | ------ |
| Stock Output        | 60,000 |        |
| Inventory Valuation |        | 60,000 |

And that's it.

The Stock Output account may remain with balances and is used as an expense account or clearing account depending on the localization.

The invoice does **not** clear it.

---

# 2. Anglo-Saxon Accounting (Common in US/UK)

This is the one that probably matches your intuition.

### Delivery

| Account             | Debit  | Credit |
| ------------------- | ------ | ------ |
| Stock Output        | 60,000 |        |
| Inventory Valuation |        | 60,000 |

---

### Invoice

Odoo automatically creates an additional entry:

| Account            | Debit  | Credit |
| ------------------ | ------ | ------ |
| Cost of Goods Sold | 60,000 |        |
| Stock Output       |        | 60,000 |

Now:

```text
Stock Output:
+60,000
-60,000
---------
0
```

Perfect.

---

# Why does Anglo-Saxon do this?

Because it follows the **matching principle**:

> Expenses should be recognized when the related revenue is recognized.

You sold for:

```text
Revenue = 100,000
```

Cost:

```text
COGS = 60,000
```

Both should appear on the Profit & Loss statement at the same time.

---

# Full Flow in Anglo-Saxon

Suppose:

- Cost = 60,000
- Sale Price = 100,000

## Delivery

| Dr                  | Cr               |
| ------------------- | ---------------- |
| Stock Output 60,000 | Inventory 60,000 |

---

## Invoice

### Revenue Entry

| Dr         | Cr                    |
| ---------- | --------------------- |
| AR 100,000 | Sales Revenue 100,000 |

### Cost Entry

| Dr          | Cr                  |
| ----------- | ------------------- |
| COGS 60,000 | Stock Output 60,000 |

---

## Payment

| Dr           | Cr         |
| ------------ | ---------- |
| Bank 100,000 | AR 100,000 |

---

# Final Balances

Inventory:

```text
-60,000
```

Stock Output:

```text
0
```

COGS:

```text
+60,000
```

Sales Revenue:

```text
+100,000
```

Profit:

```text
40,000
```

---

# Why does Odoo even need Stock Output then?

Because:

```text
Delivery
```

and

```text
Invoice
```

can happen at different times.

Example:

```text
Today: Deliver goods
Next month: Invoice customer
```

Odoo cannot create COGS yet because there is no revenue yet.

So it temporarily parks the value in:

```text
Stock Output
```

until the invoice happens.

---

# This is exactly the same idea as Stock Input.

```text
Goods Receipt
↓
Stock Input
↓
Vendor Bill
```

and

```text
Delivery
↓
Stock Output
↓
Customer Invoice
```

---

# Why Anglo-Saxon Feels More Natural

You naturally think in **Anglo-Saxon accounting**, and honestly, most software engineers do because it feels more intuitive:

> **I shouldn't have an expense until I've actually sold something and earned revenue from it.**

This is called the **Matching Principle**.

```text
Revenue from Sale
        ↕
Cost of making that sale (COGS)
```

They should appear together.

---

Suppose:

January:

- Buy 100 laptops for 6,000,000.

February:

- Sell only 10 laptops.

If you expense everything in January:

### January P&L

```text
Revenue = 0
Expense = 6,000,000
Loss = -6,000,000
```

### February P&L

```text
Revenue = 1,000,000
Expense = 0
Profit = 1,000,000
```

This doesn't reflect reality.

---

Instead, with Anglo-Saxon:

### January

```text
Inventory Asset = 6,000,000
Expense = 0
```

### February

```text
Revenue = 1,000,000
COGS = 600,000
Profit = 400,000
```

This makes much more business sense.

---

# This Explains Why We Map Certain Accounts

Now let's answer the original question:

> "When Odoo asks me for an account mapping, why am I choosing that account?"

The answer is:

> **Because Odoo needs to know where the value should temporarily or permanently live after a business event.**

---

# Example 1: Income Account on Product Category

Odoo asks:

```text
Income Account = ?
```

You answer:

```text
Sales Revenue
```

Why?

Because when the invoice is posted:

```text
Dr Accounts Receivable
Cr Sales Revenue
```

Odoo needs to know:

> "Where should I store the fact that we earned money?"

Answer:

```text
Sales Revenue
```

---

# Example 2: Expense Account on Product Category

Odoo asks:

```text
Expense Account = ?
```

In Anglo-Saxon, this is usually:

```text
Cost of Goods Sold
```

Why?

Because after invoicing:

```text
Dr Cost of Goods Sold
Cr Stock Output
```

Odoo needs to know:

> "Where should I store the fact that inventory value was consumed to generate this sale?"

Answer:

```text
COGS
```

---

# Example 3: Inventory Valuation Account

Odoo asks:

```text
Inventory Valuation = ?
```

Why?

Because when you receive inventory:

```text
Dr Inventory Valuation
Cr Stock Input
```

Odoo needs to know:

> "Where do I store the value of inventory I currently own?"

Answer:

```text
Inventory Asset
```

---

# Example 4: Stock Input

Why does Odoo ask for it?

Because:

```text
Goods Receipt
```

and

```text
Vendor Bill
```

can happen on different days.

Odoo needs a waiting room.

```text
Warehouse → Stock Input → Accounting
```

---

# Example 5: Stock Output

Why does Odoo ask for it?

Because:

```text
Delivery
```

and

```text
Customer Invoice
```

can happen on different days.

Odoo needs another waiting room.

```text
Warehouse → Stock Output → Invoice
```

---

# A Rule That Will Help You Design COAs

Whenever Odoo asks you for an account, ask:

### Question 1

```text
What business event is happening?
```

### Question 2

```text
Has the process finished?
```

If yes:

→ Permanent account (Revenue, Expense, Inventory, AR, AP).

If no:

→ Temporary account (Stock Input, Stock Output, Suspense, Outstanding Receipts).

---

> **Learner question:** Now back to the explanations of which actions need which COA and why those COA exactly.

# Mapping by Business Action

For each **action**:

1. **What happened in the business**
2. **What accounts are affected**
3. **Why exactly those accounts are used**
4. **Why other accounts are NOT used**

---

# 1. Create Customer Invoice

Business event:

> "I sold something and now the customer owes me money."

Journal:

| Account             | Dr  | Cr  |
| ------------------- | --- | --- |
| Accounts Receivable | XXX |     |
| Sales Revenue       |     | XXX |

---

## Why Accounts Receivable?

Because:

- Customer has **not paid yet**.
- You now have a legal claim against the customer.
- That claim is an Asset.

Business question:

> How much money do customers owe me?

---

## Why NOT Cash?

Because:

- No money was received.
- If the customer pays next month, your bank balance shouldn't increase today.

---

## Why Sales Revenue?

Because:

- You earned revenue by selling something.

Business question:

> How much did we earn from sales?

---

# 2. Register Customer Payment

Business event:

> Customer finally paid.

Journal:

| Account             | Dr  | Cr  |
| ------------------- | --- | --- |
| Bank/Cash           | XXX |     |
| Accounts Receivable |     | XXX |

---

## Why Bank/Cash?

Because:

Money physically arrived.

Business question:

> How much money do I currently have?

---

## Why Credit Accounts Receivable?

Because:

The customer no longer owes you.

The claim disappears.

---

## Why NOT Sales Revenue?

Because:

You already recorded the sale during invoicing.

Payment is only:

```text
Converting receivable → cash
```

No new revenue is generated.

---

# 3. Create Vendor Bill

Business event:

> Supplier sent an invoice.

Journal:

| Account                | Dr  | Cr  |
| ---------------------- | --- | --- |
| Expense or Stock Input | XXX |     |
| Accounts Payable       |     | XXX |

---

## Why Accounts Payable?

Because:

You owe the supplier money.

Business question:

> How much money do I owe suppliers?

---

## Why Expense?

If you bought:

- electricity
- internet
- office supplies

they are consumed immediately.

---

## Why Stock Input instead of Expense?

If you bought inventory:

you haven't consumed it yet.

Inventory is still an asset.

Therefore:

```text
Vendor Bill
↓
Dr Stock Input
Cr Accounts Payable
```

---

# 4. Pay Vendor

Business event:

> You paid your supplier.

Journal:

| Account          | Dr  | Cr  |
| ---------------- | --- | --- |
| Accounts Payable | XXX |     |
| Bank             |     | XXX |

---

## Why Debit AP?

Because your obligation disappears.

---

## Why Credit Bank?

Because money left the company.

---

# 5. Receive Inventory

Business event:

> Inventory physically entered the warehouse.

Journal:

| Account             | Dr  | Cr  |
| ------------------- | --- | --- |
| Inventory Valuation | XXX |     |
| Stock Input         |     | XXX |

---

## Why Inventory Valuation?

Because:

You now own more inventory.

Business question:

> How much inventory value do I currently own?

---

## Why Stock Input?

Because:

The accounting process may not be complete yet.

The vendor bill may not exist.

---

## Why NOT Accounts Payable?

Because receiving inventory does NOT automatically mean:

```text
I owe supplier today.
```

The bill may arrive later.

---

# 6. Deliver Inventory

Business event:

> Goods physically left your warehouse.

Journal:

| Account             | Dr  | Cr  |
| ------------------- | --- | --- |
| Stock Output        | XXX |     |
| Inventory Valuation |     | XXX |

---

## Why Credit Inventory?

Because:

You no longer own those goods.

Inventory decreased.

---

## Why Debit Stock Output?

Because:

You may not have invoiced the customer yet.

Odoo needs a temporary parking place.

---

## Why NOT COGS immediately?

Because:

Depending on accounting policy (especially Anglo-Saxon), the expense should be recognized when revenue is recognized.

---

# 7. Customer Invoice (Anglo-Saxon only)

Additional journal:

| Account            | Dr  | Cr  |
| ------------------ | --- | --- |
| Cost of Goods Sold | XXX |     |
| Stock Output       |     | XXX |

---

## Why Debit COGS?

Because:

The inventory has now been consumed to generate revenue.

Business question:

> How much inventory cost did we incur to make this sale?

---

## Why Credit Stock Output?

Because:

The waiting period is over.

Delivery and invoice are now matched.

The temporary account becomes zero.

---

# 8. Sales Tax (VAT)

Customer invoice:

| Account       | Dr     | Cr     |
| ------------- | ------ | ------ |
| AR            | 11,500 |        |
| Sales Revenue |        | 10,000 |
| VAT Payable   |        | 1,500  |

---

## Why VAT Payable?

Because:

That money belongs to the government.

You are merely holding it.

---

# 9. Purchase Tax (VAT)

Vendor bill:

| Account           | Dr     | Cr     |
| ----------------- | ------ | ------ |
| Inventory/Expense | 10,000 |        |
| VAT Receivable    | 1,500  |        |
| AP                |        | 11,500 |

---

## Why VAT Receivable?

Because:

The government owes you a tax credit.

---

# 10. Bank Reconciliation

Bank statement imported:

| Account  | Dr  | Cr  |
| -------- | --- | --- |
| Bank     | XXX |     |
| Suspense |     | XXX |

---

## Why Suspense?

Because:

You know money moved.

You don't yet know:

- customer?
- supplier?
- invoice?

So Odoo temporarily parks it.

---

# The Mental Framework for Implementations

Whenever Odoo asks for an account mapping, ask:

### Step 1

```text
What business event happened?
```

### Step 2

```text
What value increased?
```

### Step 3

```text
What value decreased?
```

### Step 4

```text
Is the process complete?
```

- Yes → permanent account.
- No → temporary/clearing account.

---

This framework is powerful enough that later, when you configure things like **Manufacturing**, **Landed Costs**, or **Dropshipping** in Odoo, you'll be able to figure out the accounting entries yourself instead of memorizing them.

---

> **Learner question:** How do journals fit into this? What is a journal and how does it help?

# Journals

# Short Answer

**Accounts (COA)** answer:

> "Where did the money/value go?"

**Journals** answer:

> "Where did this transaction come from?"

or

> "What type of transaction is this?"

---

# Real-World Accounting Before Computers

Accountants used physical books.

They had:

### Sales Book

All customer invoices.

### Purchase Book

All vendor bills.

### Cash Book

All cash transactions.

### Bank Book

All bank transactions.

### Miscellaneous Book

Adjustments, depreciation, etc.

These books were called:

# Journals

The entries from these books were then posted into the ledger (COA).

---

# In Odoo

The same concept exists.

```text
Business Event
       ↓
Journal
       ↓
Journal Entry
       ↓
Accounts (COA)
       ↓
Reports
```

---

# Example

Customer invoice:

```text
Dr Accounts Receivable
Cr Sales Revenue
```

Which accounts are affected?

- Accounts Receivable
- Sales Revenue

Which journal does it belong to?

- Sales Journal

---

# Why Do We Need the Journal?

Because later you may ask:

> Show me all customer invoices.

Odoo can simply filter:

```text
Journal = Customer Invoices
```

instead of scanning the entire ledger.

---

# Think Like a Database Engineer

### COA = Tables holding balances

```text
accounts
---------
AR
AP
Cash
Sales
Inventory
```

### Journal = Transaction category / source

```text
sales
purchases
bank
cash
misc
```

---

# Example

You have these entries:

| Journal  | Debit   | Credit        |
| -------- | ------- | ------------- |
| Sales    | AR      | Sales Revenue |
| Bank     | Bank    | AR            |
| Purchase | Expense | AP            |

Notice:

The same account can appear in many journals.

---

# Example

Accounts Receivable can appear in:

- Sales Journal
- Bank Journal
- Miscellaneous Journal

because AR can be:

- created by invoices
- cleared by payments
- adjusted manually

---

# Therefore:

## Account

Stores:

```text
Current balance
```

## Journal

Stores:

```text
Transaction origin/type
```

---

# Why does Odoo ask me to choose a journal?

Because Odoo needs:

1. Number sequence
2. Default accounts
3. Permissions
4. Reports
5. Source of transaction

---

# Example: Sales Journal

Configuration:

```text
Customer Invoices
Prefix: INV
```

Invoices become:

```text
INV/2026/0001
INV/2026/0002
INV/2026/0003
```

---

# Bank Journal

Configuration:

```text
Bank
Prefix: BNK
```

Payments become:

```text
BNK/2026/0001
BNK/2026/0002
```

---

# Another Important Thing

A journal can have:

### Default Account

Example:

Bank Journal:

```text
Default Debit Account:
Bank Account

Default Credit Account:
Bank Account
```

This saves Odoo from asking:

> Which bank account?

every single time.

---

# Example

You register payment.

Odoo sees:

```text
Journal = Bank
```

It automatically knows:

```text
Debit Bank Account
```

because the journal is linked to the Bank COA.

---

# Example

Cash Journal:

```text
Cash Journal
↓
Cash Account
```

Bank Journal:

```text
Bank Journal
↓
Bank Account
```

Same payment screen.

Different journal.

Different COA used automatically.

---

# Why Do Journals Exist If We Already Have Accounts?

Because these questions are different:

### Question 1

> How much money do I have?

Answer:

```text
Bank Account balance
```

---

### Question 2

> Show me all bank transactions.

Answer:

```text
Bank Journal
```

---

### Question 3

> Show me all customer invoices.

Answer:

```text
Sales Journal
```

---

### Question 4

> Show me all manual adjustments.

Answer:

```text
Miscellaneous Journal
```

---

# Odoo's Main Journal Types

| Journal       | Purpose                           |
| ------------- | --------------------------------- |
| Sales         | Customer invoices                 |
| Purchase      | Vendor bills                      |
| Bank          | Bank transactions                 |
| Cash          | Cash transactions                 |
| Miscellaneous | Manual entries                    |
| Inventory     | Automatic stock valuation entries |

---

# One More Important Concept

A journal is basically:

```text
Container of journal entries
```

while accounts are:

```text
Buckets that hold balances
```

---

# Software Engineer Analogy

Think of:

### Journal

```python
source = "sales"
```

### Account

```python
destination = "accounts_receivable"
```

Every transaction has both:

```python
{
    "source": "Sales Journal",
    "debit": "Accounts Receivable",
    "credit": "Sales Revenue"
}
```
