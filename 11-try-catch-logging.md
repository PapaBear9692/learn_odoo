# Try-Catch & Logging in Odoo

## Exception Handling

### Basic Try-Catch

```python
from odoo import models, api
from odoo.exceptions import UserError, ValidationError

class SaleOrder(models.Model):
    _inherit = 'sale.order'

    @api.model
    def process_order(self, order_id):
        try:
            order = self.browse(order_id)
            order.action_confirm()
            return {'success': True, 'order': order.name}

        except UserError as e:
            # Business logic error - show to user
            return {'success': False, 'error': str(e)}

        except ValidationError as e:
            # Field validation error
            return {'success': False, 'error': str(e)}

        except Exception as e:
            # Unexpected error - log it, show generic message
            return {'success': False, 'error': 'An unexpected error occurred'}
```

### Common Odoo Exceptions

| Exception | Use When |
|-----------|----------|
| `UserError` | Business rule violation (show to user) |
| `ValidationError` | Field validation fails |
| `AccessError` | Permission denied |
| `MissingError` | Record not found |

```python
from odoo.exceptions import UserError, ValidationError, AccessError

# UserError - shown to user
if not self.partner_id:
    raise UserError(_("Customer is required"))

# ValidationError - field level
if self.amount < 0:
    raise ValidationError(_("Amount cannot be negative"))

# AccessError - permission
if not self.env.user.has_group('sales_team.group_sale_manager'):
    raise AccessError(_("Only managers can perform this action"))
```

---

## Logging in Odoo

### Setup Logger

```python
import logging

_logger = logging.getLogger(__name__)

class SaleOrder(models.Model):
    _inherit = 'sale.order'

    def process_order(self):
        _logger.info("Processing order %s", self.id)
```

### Log Levels

```python
import logging
_logger = logging.getLogger(__name__)

# DEBUG - Detailed debugging info
_logger.debug("Variable x = %s", x)

# INFO - General information
_logger.info("Order %s confirmed by user %s", self.id, self.env.user.id)

# WARNING - Something unexpected but handled
_logger.warning("Order %s has no lines", self.id)

# ERROR - Error occurred but code continues
_logger.error("Failed to send email for order %s", self.id)

# CRITICAL - Serious error
_logger.critical("Database connection lost")
```

### When to Use Each Level

| Level | Use For |
|-------|---------|
| `debug` | Development, tracing variables |
| `info` | Normal operations (created, updated, sent) |
| `warning` | Unexpected but recoverable situations |
| `error` | Operation failed but app continues |
| `critical` | System-level failures |

---

## Logging with Try-Catch

```python
import logging
from odoo import models, api
from odoo.exceptions import UserError

_logger = logging.getLogger(__name__)

class PaymentProcess(models.Model):
    _name = 'payment.process'

    @api.model
    def process_payment(self, order_id, amount):
        order = self.browse(order_id)

        try:
            _logger.info("Processing payment for order %s, amount %s", order_id, amount)

            # Validate
            if amount <= 0:
                raise UserError(_("Amount must be positive"))

            # Process
            payment = self.env['account.payment'].create({
                'amount': amount,
                'partner_id': order.partner_id.id,
            })
            payment.action_post()

            _logger.info("Payment %s created for order %s", payment.id, order_id)
            return {'success': True, 'payment_id': payment.id}

        except UserError as e:
            # Expected business error - don't log stack trace
            _logger.warning("Payment failed for order %s: %s", order_id, str(e))
            return {'success': False, 'error': str(e)}

        except Exception as e:
            # Unexpected error - log full details
            _logger.error("Unexpected error processing payment for order %s: %s", order_id, str(e), exc_info=True)
            return {'success': False, 'error': 'Payment processing failed'}
```

### exc_info Parameter

```python
# Log with stack trace
_logger.error("Error occurred: %s", str(e), exc_info=True)

# Same as above (shortcut)
_logger.exception("Error occurred: %s", str(e))
```

---

## Log to File

### Configure in odoo.conf

```ini
[options]
# Log file location
logfile = /var/log/odoo/odoo.log

# Log level (debug, info, warn, error)
log_level = info

# Log to both file and console
log_handler = :INFO
```

### Per-Module Logging

```ini
# In odoo.conf - different levels for different modules
log_handler = odoo.addons.sale:DEBUG
log_handler = odoo.addons.stock:WARNING
log_handler = odoo.sql_db:ERROR
```

### Log to Custom File

```python
import logging
import os

class CustomLogger:
    def __init__(self, name, log_file):
        self.logger = logging.getLogger(name)

        # Create file handler
        log_path = os.path.join('/var/log/odoo', log_file)
        handler = logging.FileHandler(log_path)
        handler.setFormatter(logging.Formatter(
            '%(asctime)s - %(name)s - %(levelname)s - %(message)s'
        ))
        self.logger.addHandler(handler)
        self.logger.setLevel(logging.DEBUG)

    def info(self, msg, *args):
        self.logger.info(msg, *args)

    def error(self, msg, *args):
        self.logger.error(msg, *args)


# Usage
payment_log = CustomLogger('payment', 'payments.log')
payment_log.info("Payment processed for order %s", order_id)
```

---

## Console Logging (Terminal)

### Print to Terminal

```python
# Simple print (shows in terminal where Odoo runs)
print(f"Debug: order_id = {order_id}")

# Using logger (controlled by log_level)
_logger.info("This appears in terminal if no logfile set")
```

### Enable Console Logging in odoo.conf

```ini
[options]
# No logfile = logs go to console
# logfile =

# Or explicit console
logfile = False
```

### Log to Both File and Console

```python
import logging
import sys

# Add console handler to existing logger
_logger = logging.getLogger(__name__)
console_handler = logging.StreamHandler(sys.stdout)
console_handler.setLevel(logging.DEBUG)
_logger.addHandler(console_handler)

_logger.info("This logs to both file and console")
```

---

## Quick Reference

### Try-Catch Pattern

```python
try:
    # Operation that might fail
    result = risky_operation()

except SpecificException as e:
    # Handle specific error
    _logger.warning("Handled error: %s", str(e))
    return {'error': str(e)}

except Exception as e:
    # Catch-all for unexpected errors
    _logger.exception("Unexpected error: %s", str(e))
    raise UserError(_("An error occurred"))

else:
    # Runs if no exception
    _logger.info("Operation successful")

finally:
    # Always runs
    cleanup()
```

### Logger Setup

```python
import logging
_logger = logging.getLogger(__name__)  # Use __name__ for module-specific logs
```

### Log Level Reference

| Code | Level | Shows In |
|------|-------|----------|
| `_logger.debug()` | DEBUG | Dev mode, detailed logs |
| `_logger.info()` | INFO | Production, normal ops |
| `_logger.warning()` | WARNING | Production, attention needed |
| `_logger.error()` | ERROR | Production, failures |
| `_logger.exception()` | ERROR | Includes stack trace |

### Common Patterns

```python
# Log start of operation
_logger.info("Starting import of %s records", len(records))

# Log progress
_logger.debug("Processed %s/%s records", i, total)

# Log errors with context
_logger.error("Failed to process order %s: %s", order.name, str(e), exc_info=True)

# Log business decisions
_logger.info("Order %s auto-approved (amount < %s)", order.name, threshold)
```
