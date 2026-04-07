# Advanced Tools in Odoo

## 1. Caching with Redis

Odoo has built-in ORM caching. For external caching, use Redis.

### Built-in ORM Caching

```python
from odoo import models, api, tools

class ProductProduct(models.Model):
    _inherit = 'product.product'

    @tools.ormcache('self.id')  # Cache by record ID
    def _get_product_info(self):
        """Result is cached - runs only once per product ID"""
        return {
            'name': self.name,
            'price': self.list_price,
            'stock': self.qty_available,
        }

    # Clear cache when product is updated
    def write(self, vals):
        result = super().write(vals)
        self._get_product_info.clear_cache(self)
        return result
```

### Redis Setup

```bash
pip install redis
sudo systemctl start redis
```

```python
import redis
import json

class RedisCache:
    def __init__(self):
        self.client = redis.Redis(
            host='localhost',
            port=6379,
            db=0,
            decode_responses=True,
        )

    def get(self, key):
        value = self.client.get(key)
        return json.loads(value) if value else None

    def set(self, key, value, ttl=300):
        self.client.setex(key, ttl, json.dumps(value))

    def delete(self, key):
        self.client.delete(key)


# Usage in model
class SaleOrder(models.Model):
    _inherit = 'sale.order'

    @api.model
    def get_dashboard_stats(self):
        cache = RedisCache()
        stats = cache.get('dashboard_stats')

        if not stats:
            stats = self._compute_stats()
            cache.set('dashboard_stats', stats, ttl=60)  # Cache for 60s

        return stats
```

---

## 2. Bus (Event-Driven Notifications)

Odoo's `bus.bus` provides real-time pub/sub notifications between users.

### How it Works

```
Publisher (Server)              Bus (bus.bus)              Subscriber (Client)
    |                               |                            |
    |-- send notification -------->|                            |
    |                               |-- poll and deliver ------>|
    |                               |                            |-- update UI
```

### Send Notification (Backend)

```python
from odoo import models

class SaleOrder(models.Model):
    _inherit = 'sale.order'

    def action_confirm(self):
        result = super().action_confirm()

        # Send notification to all users in sales channel
        self.env['bus.bus']._sendone(
            'sale_channel',              # Channel name
            'sale_order_confirmed',      # Event type
            {
                'order_id': self.id,
                'order_name': self.name,
                'amount': self.amount_total,
            }
        )

        return result
```

### Receive Notification (JavaScript)

```javascript
// In your JS file (frontend)
import { bus } from "@bus/services/bus_service";

// Listen for events
bus.addEventListener("sale_order_confirmed", (event) => {
    const data = event.detail;
    console.log(`Order ${data.order_name} confirmed! Amount: ${data.amount}`);
    // Update UI, show notification, etc.
});
```

### Common Use Cases

| Use Case | Channel | Event |
|----------|---------|-------|
| Order confirmed | `sale_channel` | `sale_order_confirmed` |
| Payment received | `payment_channel` | `payment_done` |
| Stock alert | `stock_channel` | `low_stock_warning` |
| Chat message | `chat_channel` | `new_message` |

---

## 3. Cron Jobs (Task Scheduling)

Cron jobs run tasks automatically on a schedule.

### Define Cron in XML

`security/ir_cron.xml`

```xml
<?xml version="1.0" encoding="utf-8"?>
<odoo>
    <record id="cron_send_payment_reminders" model="ir.cron">
        <field name="name">Send Payment Reminders</field>
        <field name="model_id" ref="model_sale_order"/>
        <field name="state">code</field>
        <field name="code">model._cron_send_reminders()</field>
        <field name="interval_number">1</field>
        <field name="interval_type">days</field>
        <field name="numbercall">-1</field>  <!-- -1 = run forever -->
        <field name="active">True</field>
    </record>

    <!-- Run every hour -->
    <record id="cron_sync_stock" model="ir.cron">
        <field name="name">Sync Stock Levels</field>
        <field name="model_id" ref="model_product_product"/>
        <field name="state">code</field>
        <field name="code">model._cron_sync_stock()</field>
        <field name="interval_number">1</field>
        <field name="interval_type">hours</field>
        <field name="numbercall">-1</field>
    </record>
</odoo>
```

### Cron Method in Model

```python
import logging
from odoo import models, api

_logger = logging.getLogger(__name__)

class SaleOrder(models.Model):
    _inherit = 'sale.order'

    @api.model
    def _cron_send_reminders(self):
        """Called by cron job - send reminders for unpaid orders"""
        overdue_orders = self.search([
            ('state', '=', 'sale'),
            ('invoice_status', '=', 'to invoice'),
            ('date_order', '<', fields.Date.today() - timedelta(days=7)),
        ])

        for order in overdue_orders:
            try:
                order._send_reminder_email()
                _logger.info("Reminder sent for order %s", order.name)
            except Exception as e:
                _logger.error("Failed to send reminder for %s: %s", order.name, str(e))

        _logger.info("Payment reminders cron completed. %s orders processed.", len(overdue_orders))
```

### Cron Intervals

| interval_type | interval_number | Schedule |
|---------------|-----------------|----------|
| `minutes` | `5` | Every 5 minutes |
| `hours` | `1` | Every hour |
| `days` | `1` | Every day |
| `weeks` | `1` | Every week |
| `months` | `1` | Every month |

### Add to Manifest

```python
{
    'name': 'My Module',
    'data': [
        'security/ir_cron.xml',  # Load cron definitions
    ],
}
```

---

## 4. Celery (Task Queue)

For heavy/long-running tasks that should not block the HTTP request.

### Why Celery?

```
Without Celery:
    User clicks "Generate Report"
    --> Server processes for 30 seconds
    --> User waits... page timeout

With Celery:
    User clicks "Generate Report"
    --> Task queued
    --> User gets immediate response: "Report generating..."
    --> Celery worker processes in background
    --> Notification when done
```

### Setup

```bash
pip install celery redis
```

### Celery Config

```python
# celery_config.py
from kombu import Queue

broker_url = 'redis://localhost:6379/1'
result_backend = 'redis://localhost:6379/2'

task_queues = (
    Queue('default'),
    Queue('heavy_tasks'),
)
```

### Define Tasks

```python
# tasks.py
from celery import Celery

app = Celery('odoo_tasks')
app.config_from_object('celery_config')

@app.task(queue='heavy_tasks')
def generate_large_report(order_ids):
    """Heavy task processed by Celery worker"""
    import odoo
    # Connect to Odoo, process orders, generate report
    # ...
    return {'report_path': '/reports/sales_report.pdf'}

@app.task(queue='default')
def send_bulk_emails(partner_ids, template_id):
    """Send emails without blocking the request"""
    # Process emails...
    return {'sent': len(partner_ids)}
```

### Call from Odoo

```python
from odoo import models, api

class SaleOrder(models.Model):
    _inherit = 'sale.order'

    def action_generate_report(self):
        # Queue the task - returns immediately
        from tasks import generate_large_report
        task = generate_large_report.delay(self.ids)

        # Store task ID to check status later
        self.write({'celery_task_id': task.id})

        return {
            'type': 'ir.actions.client',
            'tag': 'display_notification',
            'params': {
                'title': 'Report',
                'message': 'Report is being generated. You will be notified when ready.',
                'type': 'info',
            }
        }
```

### Run Celery Worker

```bash
# Start worker
celery -A tasks worker --loglevel=info --queues=heavy_tasks,default

# Start flower (monitoring dashboard)
celery -A tasks flower
```

---

## Quick Comparison

| Tool | Purpose | Best For |
|------|---------|----------|
| `ormcache` | In-memory caching | Frequently read, rarely changed data |
| Redis | External caching | Shared cache across workers/servers |
| `bus.bus` | Real-time notifications | UI updates, chat, alerts |
| Cron Jobs | Scheduled tasks | Daily reports, cleanup, sync |
| Celery | Background task queue | Heavy reports, bulk emails, long processing |

## When to Use What

| Situation | Tool |
|-----------|------|
| Cache product info | `ormcache` or Redis |
| Notify user of new order | `bus.bus` |
| Daily report email | Cron job |
| Generate large PDF report | Celery |
| Sync stock every hour | Cron job |
| Real-time dashboard update | `bus.bus` |
| Bulk import 10k records | Celery |
