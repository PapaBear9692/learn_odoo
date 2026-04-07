# Database Transactions & Rollback (Atomic Operations)

## What is a Transaction?

A transaction groups database operations so they **ALL succeed or ALL fail**.

```
Transaction Start
    |
    |-- Create Order      OK
    |-- Reserve Stock     OK
    |-- Create Invoice    FAIL
    |
    +-- ROLLBACK (undo everything)
```

---

## Odoo Transactions

Odoo manages transactions **automatically**:
- **Auto-commit** at end of HTTP request
- **Auto-rollback** on any exception

```python
def process_order(self):
    order = self.env['sale.order'].create({...})   # Not saved yet
    order.action_confirm()                          # Not saved yet

    if error:
        raise Exception("Failed")  # Auto rollback - order is deleted

    return result  # Auto commit - everything saved
```

---

## Auto Rollback with Exceptions

The simplest way to rollback is to raise an exception.

```python
from odoo import models, api, _
from odoo.exceptions import UserError

class SaleOrder(models.Model):
    _inherit = 'sale.order'

    @api.model
    def create_order_with_lines(self, order_vals, line_vals_list):
        """
        Create order with lines - atomic operation.
        If anything fails, everything is rolled back automatically.
        """

        # Step 1: Create order
        order = self.create(order_vals)

        # Step 2: Validate
        if not line_vals_list:
            # Raising exception = auto rollback
            # Order is deleted from database
            raise UserError(_("Order must have at least one line"))

        # Step 3: Create lines
        for line_vals in line_vals_list:
            line_vals['order_id'] = order.id
            self.env['sale.order.line'].create(line_vals)

        # Step 4: Confirm
        order.action_confirm()

        # If we reach here without exception, everything is saved
        return order
```

### What happens on exception:

```python
# Before exception:
#   Order SO001 created in memory (not committed)
#   Lines L1, L2 created in memory (not committed)

# raise UserError("...")

# After exception:
#   Everything is rolled back
#   Database is unchanged
```

---

## Savepoints for Partial Rollback

Use savepoints when you want to rollback only part of a transaction.

```python
from odoo.exceptions import UserError

class PaymentProcess(models.Model):
    _name = 'payment.process'

    def process_payment(self, order):
        # Main operation (always kept)
        order.write({'note': 'Processing payment...'})

        # Create a savepoint (checkpoint)
        savepoint = self.env.cr.savepoint()             # Savepoint()

        try:
            # Try risky operation
            payment = self.env['account.payment'].create({
                'amount': order.amount_total,
                'partner_id': order.partner_id.id,
            })
            payment.action_post()

        except Exception as e:
            # Rollback to savepoint
            # Payment is undone, but order note is still there
            self.env.cr.rollback(savepoint)

            order.write({'note': f'Payment failed: {str(e)}'})

        return True
```

### How savepoints work:

```
Transaction Start
    |
    |-- Update order note    OK
    |
    |-- SAVEPOINT created
    |       |
    |       |-- Create payment     FAIL
    |       |
    |       +-- ROLLBACK TO SAVEPOINT (only payment undone)
    |
    |-- Update order note with error    OK
    |
    +-- COMMIT (order notes saved, payment discarded)
```

---

## Manual Commit for Batch Operations

Use manual commit only for long-running batch operations. Odoo's normal operations should never use manual commit.

```python
from odoo import models, api

class DataImport(models.Model):
    _name = 'data.import'

    @api.model
    def import_products(self, product_list):
        """Import products in batches with manual commits"""

        batch_size = 100
        imported = 0

        for i in range(0, len(product_list), batch_size):
            batch = product_list[i:i + batch_size]

            try:
                for item in batch:
                    self.env['product.product'].create(item)

                # Commit after each successful batch
                self.env.cr.commit()        # Manual Commit
                imported += len(batch)

            except Exception as e:
                # Rollback only this batch, continue with next
                self.env.cr.rollback()      # Explicit Rollback
                continue

        return imported
```

### Why manual commit?

```
Without manual commit:
    Import 10,000 products
    Product 9,999 fails
    ALL 10,000 are rolled back (wasted 2 hours)

With manual commit (batch of 100):
    Import 10,000 products
    Product 9,999 fails
    Only last 100 are rolled back
    9,900 are already saved
```

---

## Complete Example: Order + Payment

```python
from odoo import models, api, _
from odoo.exceptions import UserError

class SaleOrder(models.Model):
    _inherit = 'sale.order'

    @api.model
    def create_and_pay(self, order_vals, line_vals_list):
        """
        Atomic: Create order, confirm, process payment.
        All succeed or all fail.
        """

        # 1. Create order
        order = self.create(order_vals)

        # 2. Create lines
        if not line_vals_list:
            raise UserError(_("Need at least one line"))
            # Rollback: order deleted

        for vals in line_vals_list:
            vals['order_id'] = order.id
            self.env['sale.order.line'].create(vals)

        # 3. Check stock
        for line in order.order_line:
            product = line.product_id
            if product.type == 'product':
                if product.qty_available < line.product_uom_qty:
                    raise UserError(_(
                        "Not enough stock: %s (have %s, need %s)"
                    ) % (product.name, product.qty_available, line.product_uom_qty))
                    # Rollback: order + lines deleted

        # 4. Confirm order
        order.action_confirm()

        # 5. Create payment
        payment = self.env['account.payment'].create({
            'amount': order.amount_total,
            'partner_id': order.partner_id.id,
            'payment_type': 'inbound',
        })
        payment.action_post()

        # 6. If we reach here: all succeeded, auto-commit on request end
        return {
            'order_id': order.id,
            'order_name': order.name,
            'payment_id': payment.id,
        }
```

---

## Quick Reference

| Method | Purpose | When to Use |
|--------|---------|-------------|
| Raise exception | Auto rollback everything | Business validation errors |
| `cr.savepoint()` | Create checkpoint | Try risky operation |
| `cr.rollback(savepoint)` | Rollback to checkpoint | Partial undo |
| `cr.commit()` | Save changes now | Large batch imports only |
| `cr.rollback()` | Rollback everything | Rarely needed manually |

## Best Practices

| Do | Don't |
|----|-------|
| Raise exceptions for validation errors | Use `cr.commit()` in normal operations |
| Let Odoo auto-commit at request end | Catch and silence exceptions silently |
| Use savepoints for risky sub-operations | Nest transactions manually |
| Manual commit only for batch imports | Forget that `cr.commit()` can't be undone |
