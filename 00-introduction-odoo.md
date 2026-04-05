# Introduction to Odoo

## What is Odoo?

Odoo is a **comprehensive, open-source ERP (Enterprise Resource Planning) platform** that provides a suite of business applications covering all company needs: from CRM to eCommerce, accounting to inventory, manufacturing to project management, and more.

### Key Characteristics

| Feature | Description |
|---------|-------------|
| **Open Source** | Community Edition is free and open source under LGPL license |
| **Modular** | Install only the apps you need, like building blocks |
| **Integrated** | All apps share the same database and work together seamlessly |
| **Scalable** | From small businesses to large enterprises |
| **Customizable** | Create custom modules or extend existing ones |

---

## Technical Stack

### What Odoo is Built On

| Layer | Technology |
|-------|------------|
| **Backend** | Python 3.10+ |
| **Database** | PostgreSQL |
| **Frontend** | JavaScript (OWL framework), QWeb templates |
| **CSS Framework** | Bootstrap 5 |
| **ORM** | Odoo's own Object-Relational Mapping |
| **API** | JSON-RPC, XML-RPC, REST-like endpoints |
| **Architecture** | MVC (Model-View-Controller) |

### Directory Structure

```
odoo/
├── odoo/                    # Core framework
│   ├── addons/              # Base/essential modules
│   ├── models/              # ORM and base models
│   ├── tools/               # Utilities
│   ├── http.py              # Web framework
│   └── service/             # Business logic services
├── addons/                  # Official business apps
│   ├── sale/                # Sales management
│   ├── purchase/            # Purchase management
│   ├── stock/               # Inventory/warehouse
│   ├── account/             # Accounting
│   ├── crm/                 # Customer relationship
│   └── ...                  # 50+ other modules
├── odoo-bin                 # Main entry point
└── requirements.txt         # Python dependencies
```

---

## The Plug & Play Architecture

### Modular Design Philosophy

Odoo follows a **"Lego blocks" approach** - each app is a self-contained module that:

1. **Works independently** - Install only what you need
2. **Integrates automatically** - Apps connect when both installed
3. **Extends easily** - Add features without modifying core code

### How Modules Work

```
┌─────────────────────────────────────────────────────┐
│                    base module                      │
│              (foundation - always loaded)           │
└─────────────────────┬───────────────────────────────┘
                      │
        ┌─────────────┼─────────────┐
        │             │             │
        ▼             ▼             ▼
   ┌─────────┐   ┌─────────┐   ┌─────────┐
   │   sale  │   │  stock  │   │   crm   │
   └────┬────┘   └────┬────┘   └─────────┘
        │             │
        └──────┬──────┘
               │
               ▼
        ┌─────────────┐
        │ sale_stock  │   ← Bridge module (auto-integration)
        └─────────────┘
```

### Module Types

| Type | Description | Example |
|------|-------------|---------|
| **Base Modules** | Core functionality | `base`, `web`, `mail` |
| **Business Apps** | Main features | `sale`, `purchase`, `stock` |
| **Bridge Modules** | Connect apps | `sale_stock`, `purchase_stock` |
| **Localization** | Country-specific | `l10n_us`, `l10n_bd` |
| **Custom** | Your own modules | `book_management` |

---

## Odoo Capabilities

### Core Business Apps (30+ available)

| Category | Modules | What it Does |
|----------|---------|--------------|
| **Sales** | `sale`, `sale_crm`, `sale_product_configurator` | Quotes, orders, contracts |
| **CRM** | `crm` | Lead tracking, pipeline, opportunities |
| **Website** | `website`, `website_sale` | CMS, eCommerce, blogs |
| **Inventory** | `stock`, `stock_account` | Warehouse, logistics, transfers |
| **Purchase** | `purchase`, `purchase_requisition` | Vendor management, POs |
| **Accounting** | `account`, `account_payment` | Invoicing, payments, reports |
| **Manufacturing** | `mrp`, `mrp_account` | BOMs, work orders, production |
| **Project** | `project`, `timesheet` | Tasks, timesheets, planning |
| **HR** | `hr`, `hr_payroll`, `hr_recruitment` | Employees, leaves, recruitment |
| **Marketing** | `mass_mailing`, `marketing_automation` | Email campaigns, automation |
| **Services** | `helpdesk`, `fieldservice` | Support tickets, field service |

### Technical Capabilities

| Feature | Description |
|---------|-------------|
| **Multi-Company** | Manage multiple companies in one database |
| **Multi-Currency** | Handle transactions in different currencies |
| **Multi-Language** | Translate UI and data |
| **Access Rights** | Fine-grained permissions per user/group |
| **APIs** | JSON-RPC, XML-RPC for external integrations |
| **Reports** | PDF/Excel reports using QWeb templates |
| **Workflows** | Automated actions, server actions, automated rules |
| **Chatter** | Built-in communication on records |
| **Activities** | Task scheduling on any record |

---

## Development Features

### ORM (Object-Relational Mapping)

```python
# Define a database table with Python class
class Book(models.Model):
    _name = 'library.book'
    _description = 'Library Book'

    name = fields.Char(required=True)        # VARCHAR
    price = fields.Float(digits=(10, 2))     # NUMERIC
    active = fields.Boolean(default=True)    # BOOLEAN
    category_id = fields.Many2one('book.category')  # FOREIGN KEY
```

### Automatic Database Management

- **Auto-create tables** when module is installed
- **Auto-migrate** when fields are added/modified
- **Constraints** enforced at database level
- **Indexes** created automatically for frequently queried fields

### View System (QWeb)

```xml
<!-- Declarative UI - no HTML/CSS needed -->
<record id="book_view_form" model="ir.ui.view">
    <field name="model">library.book</field>
    <field name="arch" type="xml">
        <form>
            <group>
                <field name="name"/>
                <field name="price"/>
            </group>
        </form>
    </field>
</record>
```

---

## Odoo Editions

| Feature | Community (Free) | Enterprise (Paid) |
|---------|------------------|-------------------|
| **Source Code** | Open source | Proprietary + community |
| **Modules** | 30+ apps | 50+ apps |
| **Accounting** | Basic | Full (bank sync, reports) |
| **Studio** | No | Yes (visual module builder) |
| **Support** | Community forums | Official support |
| **Hosting** | Self-hosted | Odoo.sh or self-hosted |

---

## Typical Use Cases

### Small Business
```
Base → CRM → Sales → Invoicing → Website
```
Perfect for: Freelancers, small shops, consultants

### Medium Business
```
Base → CRM → Sales → Purchase → Inventory → Accounting → Project
```
Perfect for: Retailers, service companies, manufacturers

### Enterprise
```
All modules + Custom modules + Multi-company + Integrations
```
Perfect for: Large organizations with complex workflows


---

## Resources

| Resource | URL |
|----------|-----|
| **Official Documentation** | https://www.odoo.com/documentation/19.0/ |
| **GitHub Repository** | https://github.com/odoo/odoo |
| **Odoo Apps Store** | https://apps.odoo.com/ |
| **Community Forum** | https://www.odoo.com/forum/ |
| **Tutorials** | https://www.odoo.com/slides/ |

---

## Next Steps

1. **Setup Odoo** → [01-setup-odoo.md](01-setup-odoo.md)
2. **Extend a Module** → [02-extend-module.md](02-extend-module.md)
3. **Create Custom Module** → [03-custom-module.md](03-custom-module.md)
