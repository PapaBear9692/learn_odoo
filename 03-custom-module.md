# Create Custom Module

This guide shows how to create a complete **Book Management** module from scratch.

## Example: Library Book Tracker

## 1. Scaffold or Create Manually

### Option A: Scaffold (Quick Start)
```bash
python odoo-bin scaffold book_management custom_addons
```

### Option B: Create Manually
```bash
mkdir -p custom_addons/book_management/{models,views,controllers,security,demo}
touch custom_addons/book_management/__init__.py
touch custom_addons/book_management/__manifest__.py
touch custom_addons/book_management/models/__init__.py
touch custom_addons/book_management/models/book.py
touch custom_addons/book_management/views/book_views.xml
touch custom_addons/book_management/security/ir.model.access.csv
```

## Complete Directory Structure

```
custom_addons/book_management/
├── __init__.py              # Package init - imports submodules
├── __manifest__.py          # Module metadata and configuration
|
├── models/
│   ├── __init__.py          # Imports all model files
│   └── book.py              # Book and Category model definitions
|
├── views/
│   └── book_views.xml       # UI definitions (forms, lists, menus)
|
├── controllers/
│   ├── __init__.py          # Imports controllers
│   └── controllers.py       # HTTP endpoints (optional)
|
├── security/
│   └── ir.model.access.csv  # Access rights for models
└── demo/
    └── demo.xml             # Sample data (optional)
```

## File Explanations

| File | Contains | Purpose |
|------|----------|---------|
| `__init__.py` | `from . import models` | Makes the directory a Python package |
| `__manifest__.py` | Dict with metadata | Tells Odoo about the module |
| `models/__init__.py` | `from . import book` | Imports model files |
| `models/book.py` | Python classes | Defines database tables and business logic |
| `views/book_views.xml` | XML records | Defines UI (forms, lists, kanban, menus) |
| `security/ir.model.access.csv` | CSV rows | Defines who can access the models |
| `controllers/controllers.py` | Python classes | HTTP endpoints for web and API |
| `demo/demo.xml` | XML records | Sample data for testing |

---

## 2. Create __init__.py

`custom_addons/book_management/__init__.py`
```python
from . import models
from . import controllers
```

---

## 3. Create __manifest__.py

`custom_addons/book_management/__manifest__.py`
```python
{
    'name': 'Book Management',
    'summary': 'Manage library books and borrowings',
    'description': '''
        A complete library management system.
        - Track books with title, author, ISBN
        - Categorize books
        - Manage book borrowing and returns
    ''',
    'author': 'Your Company',
    'website': 'https://www.yourcompany.com',
    'category': 'Services',
    'version': '1.0.0',
    'license': 'LGPL-3',

    # Modules that must be installed first
    'depends': ['base'],

    # Data files loaded in order
    'data': [
        'security/ir.model.access.csv',
        'views/book_views.xml', # the view that we will create
    ],

    # Demo data (loaded only with --demo flag)
    'demo': [
        'demo/demo.xml',
    ],

    # Makes it appear as an installable app
    'application': True,

    # Auto-install when dependencies are met
    'auto_install': False,
}
```

### Manifest Fields Reference

| Field | Required | Description |
|-------|----------|-------------|
| `name` | Yes | Module display name |
| `version` | Yes | Version number (e.g., 1.0.0) |
| `depends` | Yes | List of required modules |
| `data` | Yes | XML/CSV files to load |
| `summary` | No | Short description |
| `description` | No | Long description (can use triple quotes) |
| `author` | No | Author name |
| `category` | No | Category in Apps list |
| `application` | No | True = shows in Apps as installable |
| `demo` | No | Demo data files |
| `auto_install` | No | Auto-install when dependencies met |

---

## 4. Create Models

### models/__init__.py
```python
from . import book
```

### models/book.py
```python
from odoo import models, fields, api, _
from odoo.exceptions import ValidationError

class BookCategory(models.Model):
    """Category for organizing books"""
    _name = 'book.category' # Notice we added name and did not add _inherit
    _description = 'Book Category'
    _order = 'name'

    # Fields
    name = fields.Char(
        string='Category Name',
        required=True,
        translate=True
    )
    code = fields.Char(
        string='Code',
        size=10,
        help='Short code for the category'
    )
    active = fields.Boolean(default=True)
    book_ids = fields.One2many(
        'book.management',      # Related model
        'category_id',          # Field in related model
        string='Books'
    )
    book_count = fields.Integer(
        string='Book Count',
        compute='_compute_book_count'
    )

    @api.depends('book_ids')
    def _compute_book_count(self):
        for category in self:
            category.book_count = len(category.book_ids)

# We could create seperate files for the classed, 
# but would have to add the names in ```models/__init__.py```
class BookManagement(models.Model):
    """Main book model"""
    _name = 'book.management'
    _description = 'Book Management'
    _order = 'name, publish_date desc'
    _rec_name = 'name'

    # Basic Info Fields
    name = fields.Char(
        string='Title',
        required=True,
        index=True
    )
    author = fields.Char(
        string='Author',
        required=True
    )
    isbn = fields.Char(
        string='ISBN',
        size=17,
        copy=False
    )
    category_id = fields.Many2one(
        'book.category',
        string='Category',
        ondelete='set null'
    )
    description = fields.Text(
        string='Description',
        translate=True
    )
    cover_image = fields.Binary(
        string='Cover Image',
        attachment=True
    )

    # Book Details
    page_count = fields.Integer(
        string='Number of Pages'
    )
    publish_date = fields.Date(
        string='Publish Date'
    )
    publisher = fields.Char(
        string='Publisher'
    )
    language = fields.Selection([
        ('en', 'English'),
        ('bn', 'Bengali'),
        ('es', 'Spanish'),
        ('fr', 'French'),
        ('de', 'German'),
    ], string='Language', default='en')

    # Status Fields
    active = fields.Boolean(
        string='Active',
        default=True
    )
    state = fields.Selection([
        ('available', 'Available'),
        ('borrowed', 'Borrowed'),
        ('reserved', 'Reserved'),
        ('maintenance', 'Under Maintenance'),
    ], string='Status',
       default='available',
       readonly=True,
       copy=False,
       index=True)

    # Borrowing Fields
    borrower_id = fields.Many2one(
        'res.partner',
        string='Borrower',
        readonly=True,
        copy=False
    )
    borrow_date = fields.Date(
        string='Borrow Date',
        readonly=True,
        copy=False
    )
    return_date = fields.Date(
        string='Return Date',
        readonly=True,
        copy=False
    )
    due_date = fields.Date(
        string='Due Date',
        readonly=True,
        copy=False
    )

    # Computed Fields
    is_overdue = fields.Boolean(
        string='Is Overdue',
        compute='_compute_is_overdue',
        store=True
    )
    days_borrowed = fields.Integer(
        string='Days Borrowed',
        compute='_compute_days_borrowed'
    )

    @api.depends('due_date', 'state')
    def _compute_is_overdue(self):
        today = fields.Date.today()
        for book in self:
            book.is_overdue = (
                book.state == 'borrowed' and
                book.due_date and
                book.due_date < today
            )

    @api.depends('borrow_date', 'state')
    def _compute_days_borrowed(self):
        today = fields.Date.today()
        for book in self:
            if book.borrow_date and book.state == 'borrowed':
                delta = today - book.borrow_date
                book.days_borrowed = delta.days
            else:
                book.days_borrowed = 0

    # Constraint Example
    @api.constrains('isbn')
    def _check_isbn_format(self):
        for book in self:
            if book.isbn:
                # Simple ISBN-13 check
                isbn_clean = book.isbn.replace('-', '').replace(' ', '')
                if len(isbn_clean) not in [10, 13]:
                    raise ValidationError(_(
                        'ISBN must be 10 or 13 characters long'
                    ))

    # Action Methods
    def action_borrow(self):
        """Mark book as borrowed"""
        self.ensure_one()
        if self.state != 'available':
            raise ValidationError(_(
                'Only available books can be borrowed'
            ))
        self.write({
            'state': 'borrowed',
            'borrow_date': fields.Date.today(),
            'due_date': fields.Date.add(fields.Date.today(), days=14),
        })

    def action_return(self):
        """Mark book as returned"""
        self.ensure_one()
        self.write({
            'state': 'available',
            'return_date': fields.Date.today(),
            'borrower_id': False,
            'borrow_date': False,
            'due_date': False,
        })

    def action_reserve(self):
        """Reserve the book"""
        self.ensure_one()
        if self.state != 'available':
            raise ValidationError(_(
                'Only available books can be reserved'
            ))
        self.write({'state': 'reserved'})

    def action_maintenance(self):
        """Put book under maintenance"""
        self.write({'state': 'maintenance'})

    def action_set_available(self):
        """Set book as available"""
        self.write({'state': 'available'})
```

### Model Field Types Reference

| Field Type | Description | Example |
|------------|-------------|---------|
| `Char` | Short text | `name = fields.Char(required=True)` |
| `Text` | Long text | `description = fields.Text()` |
| `Integer` | Whole number | `page_count = fields.Integer()` |
| `Float` | Decimal number | `price = fields.Float(digits=(10,2))` |
| `Boolean` | True/False | `active = fields.Boolean(default=True)` |
| `Date` | Date only | `publish_date = fields.Date()` |
| `Datetime` | Date and time | `created_at = fields.Datetime()` |
| `Selection` | Dropdown | `state = fields.Selection([('a','A')])` |
| `Many2one` | Foreign key | `category_id = fields.Many2one('book.category')` |
| `One2many` | Reverse relation | `book_ids = fields.One2many('book', 'category_id')` |
| `Many2many` | Many to many | `tag_ids = fields.Many2many('book.tag')` |
| `Binary` | File/Image | `cover_image = fields.Binary()` |
| `Html` | HTML content | `content = fields.Html()` |
| `Computed` | Calculated | `total = fields.Float(compute='_compute_total')` |

### Common Field Parameters

| Parameter | Description |
|-----------|-------------|
| `string` | Label in UI |
| `required` | Must have a value |
| `readonly` | Cannot be edited in UI |
| `default` | Default value |
| `help` | Tooltip text |
| `translate` | Translatable |
| `copy` | Copy when duplicating record |
| `index` | Create database index |
| `ondelete` | Action when related record deleted |
| `domain` | Filter for relational fields |

---

## 5. Create Views

### views/book_views.xml
```xml
<?xml version="1.0" encoding="utf-8"?>
<odoo>

    <!-- ==================== CATEGORY VIEWS ==================== -->

    <!-- Category List View -->
    <record id="book_category_view_list" model="ir.ui.view">
        <field name="name">book.category.list</field>
        <field name="model">book.category</field>
        <field name="arch" type="xml">
            <list string="Categories">
                <field name="name"/>
                <field name="code"/>
                <field name="book_count"/>
            </list>
        </field>
    </record>

    <!-- Category Form View -->
    <record id="book_category_view_form" model="ir.ui.view">
        <field name="name">book.category.form</field>
        <field name="model">book.category</field>
        <field name="arch" type="xml">
            <form string="Category">
                <sheet>
                    <widget name="web_ribbon" title="Archived" bg_color="bg-danger"
                        invisible="active == True"/>
                    <group>
                        <field name="name"/>
                        <field name="code"/>
                        <field name="active" invisible="1"/>
                    </group>
                    <group string="Books in this Category">
                        <field name="book_ids" nolabel="1" readonly="1">
                            <list>
                                <field name="name"/>
                                <field name="author"/>
                                <field name="state"/>
                            </list>
                        </field>
                    </group>
                </sheet>
            </form>
        </field>
    </record>

    <!-- Category Action -->
    <record id="book_category_action" model="ir.actions.act_window">
        <field name="name">Categories</field>
        <field name="res_model">book.category</field>
        <field name="view_mode">list,form</field>
        <field name="help" type="html">
            <p class="o_view_nocontent_smiling_face">
                Create a new book category
            </p>
        </field>
    </record>

    <!-- ==================== BOOK VIEWS ==================== -->

    <!-- Book List View -->
    <record id="book_management_view_list" model="ir.ui.view">
        <field name="name">book.management.list</field>
        <field name="model">book.management</field>
        <field name="arch" type="xml">
            <list string="Books"
                  decoration-warning="state=='borrowed'"
                  decoration-danger="is_overdue==True"
                  decoration-success="state=='available'">
                <field name="name"/>
                <field name="author"/>
                <field name="category_id" optional="show"/>
                <field name="isbn" optional="hide"/>
                <field name="publisher" optional="hide"/>
                <field name="publish_date" optional="hide"/>
                <field name="borrower_id" optional="hide"/>
                <field name="due_date" optional="show"/>
                <field name="is_overdue" invisible="1"/>
                <field name="state" widget="badge"
                    decoration-success="state=='available'"
                    decoration-warning="state=='borrowed'"
                    decoration-info="state=='reserved'"
                    decoration-muted="state=='maintenance'"/>
            </list>
        </field>
    </record>

    <!-- Book Form View -->
    <record id="book_management_view_form" model="ir.ui.view">
        <field name="name">book.management.form</field>
        <field name="model">book.management</field>
        <field name="arch" type="xml">
            <form string="Book">
                <header>
                    <!-- Action Buttons -->
                    <button name="action_borrow" string="Borrow" type="object"
                        class="btn-primary" invisible="state != 'available'"/>
                    <button name="action_return" string="Return" type="object"
                        class="btn-primary" invisible="state != 'borrowed'"/>
                    <button name="action_reserve" string="Reserve" type="object"
                        invisible="state != 'available'"/>
                    <button name="action_maintenance" string="Maintenance" type="object"
                        class="btn-warning" invisible="state == 'maintenance'"/>
                    <button name="action_set_available" string="Set Available" type="object"
                        invisible="state == 'available'"/>
                    <!-- Status Bar -->
                    <field name="state" widget="statusbar"
                        statusbar_visible="available,borrowed,reserved"/>
                </header>
                <sheet>
                    <!-- Cover Image -->
                    <field name="cover_image"
                        widget="image"
                        class="oe_avatar"
                        nolabel="1"/>
                    <!-- Title -->
                    <div class="oe_title">
                        <h1>
                            <field name="name" placeholder="Book Title"/>
                        </h1>
                        <h3>
                            <field name="author" placeholder="Author Name"/>
                        </h3>
                    </div>
                    <!-- Main Content -->
                    <group>
                        <group string="Book Information">
                            <field name="isbn"/>
                            <field name="category_id"/>
                            <field name="publisher"/>
                            <field name="publish_date"/>
                            <field name="page_count"/>
                            <field name="language"/>
                        </group>
                        <group string="Borrowing Information"
                            invisible="state == 'available'">
                            <field name="borrower_id"/>
                            <field name="borrow_date"/>
                            <field name="due_date"/>
                            <field name="return_date"/>
                            <field name="days_borrowed"/>
                            <field name="is_overdue"/>
                        </group>
                    </group>
                    <!-- Description -->
                    <group string="Description">
                        <field name="description" nolabel="1" placeholder="Book description..."/>
                    </group>
                </sheet>
            </form>
        </field>
    </record>

    <!-- Book Kanban View -->
    <record id="book_management_view_kanban" model="ir.ui.view">
        <field name="name">book.management.kanban</field>
        <field name="model">book.management</field>
        <field name="arch" type="xml">
            <kanban default_group_by="state">
                <field name="id"/>
                <field name="name"/>
                <field name="author"/>
                <field name="state"/>
                <field name="category_id"/>
                <templates>
                    <t t-name="kanban-box">
                        <div class="oe_kanban_card oe_kanban_global_click">
                            <div class="o_kanban_image">
                                <img t-att-src="kanban_image('book.management', 'cover_image', record.id.raw_value)"
                                    alt="Cover"/>
                            </div>
                            <div class="oe_kanban_content">
                                <strong><field name="name"/></strong><br/>
                                <small><field name="author"/></small><br/>
                                <field name="state" widget="badge"/>
                            </div>
                        </div>
                    </t>
                </templates>
            </kanban>
        </field>
    </record>

    <!-- Book Search View -->
    <record id="book_management_view_search" model="ir.ui.view">
        <field name="name">book.management.search</field>
        <field name="model">book.management</field>
        <field name="arch" type="xml">
            <search string="Search Books">
                <field name="name"/>
                <field name="author"/>
                <field name="isbn"/>
                <field name="category_id"/>
                <filter string="Available" name="available"
                    domain="[('state','=','available')]"/>
                <filter string="Borrowed" name="borrowed"
                    domain="[('state','=','borrowed')]"/>
                <filter string="Overdue" name="overdue"
                    domain="[('is_overdue','=',True)]"/>
                <group expand="0" string="Group By">
                    <filter string="Category" name="category"
                        context="{'group_by':'category_id'}"/>
                    <filter string="Status" name="state"
                        context="{'group_by':'state'}"/>
                    <filter string="Language" name="language"
                        context="{'group_by':'language'}"/>
                </group>
            </search>
        </field>
    </record>

    <!-- Book Action -->
    <record id="book_management_action" model="ir.actions.act_window">
        <field name="name">Books</field>
        <field name="res_model">book.management</field>
        <field name="view_mode">list,form,kanban</field>
        <field name="search_view_id" ref="book_management_view_search"/>
        <field name="help" type="html">
            <p class="o_view_nocontent_smiling_face">
                Add your first book to the library
            </p>
        </field>
    </record>

    <!-- ======= MENU ITEMS (The Top Menu ) ====== -->

    <!-- Root Menu -->
    <menuitem id="book_management_menu_root"
        name="Book Management"
        web_icon="book_management,static/description/icon.png"
        sequence="50"/>

    <!-- Sub Menus -->
    <menuitem id="book_management_menu_books"
        name="Books"
        parent="book_management_menu_root"
        action="book_management_action"
        sequence="10"/>

    <menuitem id="book_management_menu_categories"
        name="Categories"
        parent="book_management_menu_root"
        action="book_category_action"
        sequence="20"/>

</odoo>
```

### View Elements Reference

| Element | Description |
|---------|-------------|
| `<list>` | Table/list view (was `<tree>` in older Odoo) |
| `<form>` | Detail/edit form |
| `<kanban>` | Card view |
| `<search>` | Search panel and filters |
| `<group>` | Grouped fields in forms |
| `<sheet>` | Main content area in forms |
| `<header>` | Top bar with buttons and status |
| `<field>` | Display a field |
| `<button>` | Action button |
| `<filter>` | Search filter |
| `<xpath>` | Used when inheriting views |

### Common Widgets

| Widget | Use |
|--------|-----|
| `widget="badge"` | Colored status badge |
| `widget="statusbar"` | Status progression bar |
| `widget="image"` | Image field |
| `widget="progressbar"` | Progress bar |
| `widget="monetary"` | Currency field |
| `widget="priority"` | Stars/priority |
| `widget="mail_thread"` | Chatter |

---

## 6. Create Security (Check 06-security-rules.md for details)

### security/ir.model.access.csv
```csv
id,name,model_id:id,group_id:id,perm_read,perm_write,perm_create,perm_unlink
access_book_management_user,book.management.user,model_book_management,base.group_user,1,1,1,0
access_book_management_manager,book.management.manager,model_book_management,base.group_system,1,1,1,1
access_book_category_user,book.category.user,model_book_category,base.group_user,1,1,1,0
access_book_category_manager,book.category.manager,model_book_category,base.group_system,1,1,1,1
```

### CSV Columns Explained

| Column | Description |
|--------|-------------|
| `id` | Unique identifier for this access rule |
| `name` | Display name |
| `model_id:id` | Model ID (prefix `model_` + model name with dots to underscores) |
| `group_id:id` | User group (base.group_user = all internal users) |
| `perm_read` | Can view (1=yes, 0=no) |
| `perm_write` | Can edit (1=yes, 0=no) |
| `perm_create` | Can create (1=yes, 0=no) |
| `perm_unlink` | Can delete (1=yes, 0=no) |

### Common Groups

| Group ID | Description |
|----------|-------------|
| `base.group_user` | All internal users |
| `base.group_system` | Administration/Settings |
| `base.group_portal` | Portal users (external) |
| `base.group_public` | Public users (not logged in) |

---

## 7. Create Demo Data (Optional)

### demo/demo.xml
```xml
<?xml version="1.0" encoding="utf-8"?>
<odoo noupdate="1">
    <!-- Sample Categories -->
    <record id="demo_category_fiction" model="book.category">
        <field name="name">Fiction</field>
        <field name="code">FIC</field>
    </record>

    <record id="demo_category_tech" model="book.category">
        <field name="name">Technology</field>
        <field name="code">TECH</field>
    </record>

    <!-- Sample Books -->
    <record id="demo_book_1" model="book.management">
        <field name="name">The Great Gatsby</field>
        <field name="author">F. Scott Fitzgerald</field>
        <field name="isbn">978-0743273565</field>
        <field name="category_id" ref="demo_category_fiction"/>
        <field name="page_count">180</field>
        <field name="publisher">Scribner</field>
        <field name="language">en</field>
    </record>

    <record id="demo_book_2" model="book.management">
        <field name="name">Python Programming</field>
        <field name="author">John Doe</field>
        <field name="isbn">978-1234567890</field>
        <field name="category_id" ref="demo_category_tech"/>
        <field name="page_count">450</field>
        <field name="publisher">Tech Books</field>
        <field name="language">en</field>
    </record>
</odoo>
```

---

## 8. Install the Module

### Method 1: Web UI
0. Turn on Developer Mode in settings (in Odoo web ui)
1. Start Odoo: `python odoo-bin -c odoo.conf`
2. Go to **Apps** → **Update Apps List**
3. Search for "Book Management"
4. Click **Install**

### Method 2: Command Line
```bash
python odoo-bin -c odoo.conf -d my_odoo_db_name -i book_management --stop-after-init
```

---

## 9. Update Module After Changes

When you modify the module files: In Web UI: **Apps** → Search module → Click **Upgrade**

Or 
```bash
# Restart Odoo and update the module
python odoo-bin -c odoo.conf -d my_odoo_db_name -u book_management
```



---

## Quick Reference

### Model Inheritance
```python
# Extend existing model
class SaleOrder(models.Model):
    _inherit = 'sale.order'
```

### View Inheritance
```xml
<record id="view_inherit" model="ir.ui.view">
    <field name="inherit_id" ref="sale.view_order_form"/>
    <xpath expr="//field[@name='name']" position="after">
        <field name="my_field"/>
    </xpath>
</record>
```

### Model Methods
```python
@api.model  # No record context
def create(self, vals):
    return super().create(vals)

@api.depends('field1', 'field2')  # Recompute when these change
def _compute_total(self):
    for rec in self:
        rec.total = rec.field1 + rec.field2

@api.onchange('field1')  # Trigger on UI change
def _onchange_field1(self):
    self.field2 = self.field1 * 2

@api.constrains('field1')  # Validate on save
def _check_field1(self):
    if self.field1 < 0:
        raise ValidationError('Must be positive')
```
