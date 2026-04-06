# Extend Existing Module

This guide shows how to extend the **Sale Order** (included in base odoo) module by adding a custom field.

## Example: Add "Custom Note" and "Delivery Preference" field to Sale Order

## 1. Scaffold Extension Module

```bash
python odoo-bin scaffold sale_order_ext custom_addons
```
here ```sale_order_ext``` is the name of the module we are creating, 

```custom_addons``` is the name of folder we are creating the model at.

This creates:
```
custom_addons/sale_order_ext/
|
├── __init__.py
├── __manifest__.py
|
├── models/
│   ├── __init__.py
│   └── sale_order_ext.py
|
├── views/
│   └── sale_order_ext_views.xml
|
├── controllers/
│   ├── __init__.py
│   └── controllers.py
|
├── security/
│   └── ir.model.access.csv
|
└── demo/
    └── demo.xml
```

## Directory Structure Explained

| File/Folder | Purpose |
|-------------|---------|
| `__init__.py` | Python package initializer. Imports models and controllers |
| `__manifest__.py` | Module metadata: name, dependencies, our custom made files to load |
| `models/` | Contains Python classes that define/extend database models |
| `models/__init__.py` | Imports all model files |
| `models/sale_order_ext.py` | Extends sale.order model with new fields/methods |
| `views/` | Contains XML files that define UI (forms, lists, menus) |
| `views/sale_order_ext_views.xml` | Extends existing views to show new fields |
| `controllers/` | HTTP controllers for web routes (optional) and API |
| `security/` | Access rights and rules |
| `security/ir.model.access.csv` | Defines who can read/write/create/delete records |
| `demo/` | Demo data loaded when creating database with demo data |

## 2. Update __init__.py

`custom_addons/sale_order_ext/__init__.py`
```python
from . import models
from . import controllers
# should be already there if created with scaffold
```

## 3. Create Model Extension



`custom_addons/sale_order_ext/models/sale_order_ext.py`

```python
from odoo import models, fields

class SaleOrder(models.Model):
    _inherit = 'sale.order'  # Extend existing sale.order model

    # Add new field
    custom_note = fields.Char(
        string="Custom Note",
        help="Add internal notes for this order"
    )

    delivery_preference = fields.Selection([
        ('standard', 'Standard Delivery'),
        ('express', 'Express Delivery'),
        ('pickup', 'Store Pickup'),
    ], string="Delivery Preference", default='standard')

    # Add a computed field
    total_with_shipping = fields.Float(
        string="Total with Shipping",
        compute='_compute_total_with_shipping'
    )

    # Demo Method for the newly added computed field
    def _compute_total_with_shipping(self):
        for order in self:
            shipping = 10.0 if order.delivery_preference == 'express' else 5.0
            order.total_with_shipping = order.amount_total + shipping
```

### Model File Explained

| Code | Description |
|------|-------------|
| `_inherit = 'sale.order'` | Tells Odoo to extend the existing sale.order model. THIS IS VERY INPORTANT|
| `custom_note = fields.Char(...)` | Adds a new text field to the database |
| `string="Custom Note"` | Label shown in UI |
| `help="..."` | Tooltip text to show when user hovers over the field |
| `fields.Selection([...])` | Dropdown field with predefined options |
| `compute='_compute_total...'` | Field value is calculated by a method |

### Finally Add the created model to ```model/__init__.py``` to let odoo know about the changes

`custom_addons/sale_order_ext/models/__init__.py`
```python
from . import sale_order_ext
```

## 4. Create View Extension

`custom_addons/sale_order_ext/views/sale_order_ext_views.xml`
```xml
<?xml version="1.0" encoding="utf-8"?>
<odoo>
    <!-- Extend Sale Order Form View -->
    <record id="view_sale_order_form_inherit" model="ir.ui.view">
        <field name="name">sale.order.form.inherit</field>
        <field name="model">sale.order</field>
        <field name="inherit_id" ref="sale.view_order_form"/>
        <field name="arch" type="xml">
            <!-- Add custom_note after customer reference field -->
            <xpath expr="//field[@name='client_order_ref']" position="after">
                <field name="custom_note"/>
            </xpath>

            <!-- Add delivery preference in shipping section -->
            <xpath expr="//group[@name='sale_shipping']" position="inside">
                <field name="delivery_preference"/>
            </xpath>
        </field>
    </record>

    <!-- Extend Sale Order List View -->
    <record id="view_sale_order_list_inherit" model="ir.ui.view">
        <field name="name">sale.order.list.inherit</field>
        <field name="model">sale.order</field>
        <field name="inherit_id" ref="sale.view_order_tree"/>
        <field name="arch" type="xml">
            <xpath expr="//field[@name='amount_total']" position="after">
                <field name="delivery_preference" widget="badge"/>
            </xpath>
        </field>
    </record>
</odoo>
```

### View File Explained

| Code | Description |
|------|-------------|
| `<record model="ir.ui.view">` | Creates/updates a view record |
| `inherit_id` | Which existing view to extend |
| `ref="sale.view_order_form"` | External ID of the parent view |
| `<xpath expr="..." position="...">` | Finds element and modifies it |
| `position="after"` | Insert after found element |
| `position="before"` | Insert before found element |
| `position="inside"` | Insert inside found element |
| `position="replace"` | Replace found element |
| `expr="//field[@name='field_name']"` | XPath to find elements |

## 5. Update __manifest__.py
### Very important. Updating this lets odoo know what this module is and how to use it
`custom_addons/sale_order_ext/__manifest__.py`
```python
{
    'name': 'Sale Order Extension',
    'summary': 'Add custom fields to Sale Orders',
    'description': 'Extends Sale Order with custom note and delivery preference',
    'author': 'Your Company',
    'category': 'Sales',
    'version': '1.0',
    'depends': ['sale'],  # Must depend on the module being extended
    'data': [
        'views/sale_order_ext_views.xml', # must add the created the view here
    ],
    # if set True, it will appear as an application in app list
    'application': False,  # Not a standalone app, just extends sale
    
    'auto_install': False,
}
```

### Manifest Fields Explained

| Field | Description |
|-------|-------------|
| `name` | Display name in Apps list |
| `summary` | Short one-line description |
| `depends` | List of modules that must be installed first |
| `data` | XML/CSV files to load (views, security, data) |
| `demo` | Files loaded only with demo data |
| `application` | True = standalone app with menu; False = just extends |
| `auto_install` | True = auto-install when dependencies are met |

## 6. Install/Update the Module

### Method 1: Web UI (Developer mode must be active)
0. ```python odoo-bin -c odoo.conf```
1. Go to **Apps**
2. Click **Update Apps List**
3. Search for "Sale Order Extension"
4. Click **Install** or **Upgrade**

### Method 2: Command Line
```bash
 # Install new module
python odoo-bin -c odoo.conf -d my_odoo_db -i sale_order_ext --stop-after-init

# Update existing module
python odoo-bin -c odoo.conf -d my_odoo_db -u sale_order_ext --stop-after-init
```


## 7. Verify

1. Go to **Sales** → **Orders**
2. Open or create a Sale Order
3. You should see the new **Custom Note** and **Delivery Preference** fields

## Common XPath Examples

```xml
<!-- Find field by name -->
<xpath expr="//field[@name='partner_id']" position="after">
    <field name="my_new_field"/>
</xpath>

<!-- Find group by name -->
<xpath expr="//group[@name='sale_shipping']" position="inside">
    <field name="delivery_preference"/>
</xpath>

<!-- Find notebook (tabs) -->
<xpath expr="//notebook" position="inside">
    <page string="Custom Tab">
        <field name="custom_field"/>
    </page>
</xpath>

<!-- Find by position in tree -->
<xpath expr="//tree/field[2]" position="after">
    <field name="new_column"/>
</xpath>

<!-- Find button by name -->
<xpath expr="//button[@name='action_confirm']" position="before">
    <button name="action_custom" string="Custom Action" type="object"/>
</xpath>
```
