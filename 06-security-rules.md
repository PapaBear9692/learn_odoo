# Odoo Security Rules

## Security Layers

```
┌─────────────────────────────────────────────┐
│  1. Access Rights (ir.model.access.csv)     │  ← What operations (CRUD)
├─────────────────────────────────────────────┤
│  2. Record Rules (ir.rule)                  │  ← Which records
├─────────────────────────────────────────────┤
│  3. Field Access (groups attribute)         │  ← Which fields
└─────────────────────────────────────────────┘
```

---

## Step-by-Step: Implement Security

### Step 1: Create Security Groups

`security/security_groups.xml`

```xml
<?xml version="1.0" encoding="utf-8"?>
<odoo>
    <!--
        Create a Librarian group.
        - id: unique identifier for this group (used in code as group_book_librarian)
        - name: display name shown in user settings
        - category_id: where this group appears in Settings > Users > Groups
        - implied_ids: users in this group also get base.group_user permissions
        - eval="[(4, ref(...))]": adds the referenced group to this group's implied groups
          - (4, id) = add id to the list (like list.append())
    -->
    <record id="group_book_librarian" model="res.groups">
        <field name="name">Librarian</field>
        <field name="category_id" ref="base.module_category_services"/>
        <field name="implied_ids" eval="[(4, ref('base.group_user'))]"/>
    </record>

    <!--
        Create a Manager group that inherits from Librarian.
        - implied_ids points to group_book_librarian (created above)
        - Manager gets all Librarian permissions + additional ones
        - This creates a hierarchy: User < Librarian < Manager
    -->
    <record id="group_book_manager" model="res.groups">
        <field name="name">Manager</field>
        <field name="category_id" ref="base.module_category_services"/>
        <field name="implied_ids" eval="[(4, ref('group_book_librarian'))]"/>
    </record>
</odoo>
```

### Step 2: Define Access Rights

`security/ir.model.access.csv`

```csv
id,name,model_id:id,group_id:id,perm_read,perm_write,perm_create,perm_unlink
access_book_user,book.user,model_book_management,base.group_user,1,0,0,0
access_book_librarian,book.librarian,model_book_management,group_book_librarian,1,1,1,0
access_book_manager,book.manager,model_book_management,group_book_manager,1,1,1,1
```

---

#### Column-by-Column Explanation

| Column | What It Does | How to Write | Example |
|--------|--------------|--------------|---------|
| `id` | Unique identifier for this access rule | `access_<model>_<group>` | `access_book_user` |
| `name` | Display name (shown in Settings > Security) | `<model>.<group>` | `book.user` |
| `model_id:id` | Which model this applies to | `model_` + model name (`.` → `_`) | `model_book_management` |
| `group_id:id` | Which user group gets these permissions | Group XML ID | `base.group_user` |
| `perm_read` | Can view/search records | `1` = yes, `0` = no | `1` |
| `perm_write` | Can edit/update records | `1` = yes, `0` = no | `1` |
| `perm_create` | Can create new records | `1` = yes, `0` = no | `1` |
| `perm_unlink` | Can delete records | `1` = yes, `0` = no | `1` |

---

#### How to Convert Model Name to model_id:id

```
Model in Python     →  model_id:id in CSV
─────────────────────────────────────────
book.management     →  model_book_management
sale.order          →  model_sale_order
res.partner         →  model_res_partner
hr.employee         →  model_hr_employee
account.move        →  model_account_move
project.task        →  model_project_task
```

**Rule:** Replace every `.` with `_` and add `model_` prefix

---

#### Permission Combinations Explained

```
perm_read, perm_write, perm_create, perm_unlink
   ↓         ↓           ↓           ↓
  View     Edit       Create      Delete
```

| Permissions | Meaning | Use Case |
|-------------|---------|----------|
| `1,0,0,0` | Read Only | Regular users viewing data |
| `1,1,0,0` | Read + Write | Users who can edit but not create/delete |
| `1,1,1,0` | Read + Write + Create | Supervisors - can create but not delete |
| `1,1,1,1` | Full Access | Managers/Admins - can do everything |
| `0,1,0,0` | Write Only | Rare - system updates only |
| `1,0,0,1` | Read + Delete | Rare - archive only access |

---

#### Complete Examples with Comments

```csv
# ═══════════════════════════════════════════════════════════════════════════
# ACCESS RIGHTS CSV - Controls what operations each group can perform
# ═══════════════════════════════════════════════════════════════════════════
# Format: id, name, model_id:id, group_id:id, perm_read, perm_write, perm_create, perm_unlink
# ─────────────────────────────────────────────────────────────────────────

# ─────────────────────────────────────────────────────────────────────────
# EXAMPLE 1: Regular Users - READ ONLY
# ─────────────────────────────────────────────────────────────────────────
# id: access_book_user (unique identifier)
# name: book.user (display name in UI)
# model_id:id: model_book_management (the model we're controlling)
# group_id:id: base.group_user (all internal users - Odoo's default group)
# perm_read: 1 (CAN view/search books)
# perm_write: 0 (CANNOT edit books)
# perm_create: 0 (CANNOT create new books)
# perm_unlink: 0 (CANNOT delete books)
# Result: Users can see books but cannot modify them at all
access_book_user,book.user,model_book_management,base.group_user,1,0,0,0

# ─────────────────────────────────────────────────────────────────────────
# EXAMPLE 2: Librarians - READ + WRITE + CREATE (No Delete)
# ─────────────────────────────────────────────────────────────────────────
# id: access_book_librarian (unique identifier)
# name: book.librarian (display name in UI)
# model_id:id: model_book_management (same model as above)
# group_id:id: group_book_librarian (our custom group from Step 1)
# perm_read: 1 (CAN view/search books)
# perm_write: 1 (CAN edit existing books - update info, change status)
# perm_create: 1 (CAN create new books - add to catalog)
# perm_unlink: 0 (CANNOT delete books - only managers can delete)
# Result: Librarians can manage books but can't permanently delete them
access_book_librarian,book.librarian,model_book_management,group_book_librarian,1,1,1,0

# ─────────────────────────────────────────────────────────────────────────
# EXAMPLE 3: Managers - FULL ACCESS (All Permissions)
# ─────────────────────────────────────────────────────────────────────────
# id: access_book_manager (unique identifier)
# name: book.manager (display name in UI)
# model_id:id: model_book_management (same model as above)
# group_id:id: group_book_manager (our custom manager group from Step 1)
# perm_read: 1 (CAN view/search books)
# perm_write: 1 (CAN edit books)
# perm_create: 1 (CAN create books)
# perm_unlink: 1 (CAN delete books - full control)
# Result: Managers have complete control over all book records
access_book_manager,book.manager,model_book_management,group_book_manager,1,1,1,1

# ─────────────────────────────────────────────────────────────────────────
# EXAMPLE 4: Category Model - Different Model, Same Groups
# ─────────────────────────────────────────────────────────────────────────
# Users can view categories (needed to select category when viewing books)
access_category_user,category.user,model_book_category,base.group_user,1,0,0,0

# Only managers can manage categories (create new categories, edit names)
access_category_manager,category.manager,model_book_category,group_book_manager,1,1,1,1
```

---

#### How Multiple Access Rights Work Together

When a user belongs to multiple groups, they get the **MOST permissive** combination:

```
User is in: base.group_user (1,0,0,0) AND group_book_librarian (1,1,1,0)
Result:     User gets (1,1,1,0) - the higher permissions win

User is in: group_book_librarian (1,1,1,0) AND group_book_manager (1,1,1,1)
Result:     User gets (1,1,1,1) - full access from manager group
```

---

#### Common Group IDs

| Group ID | Description | Who Should Use |
|----------|-------------|----------------|
| `base.group_user` | All internal users | Default for employees |
| `base.group_system` | Administration | Super admins only |
| `base.group_portal` | Portal users | External customers |
| `base.group_public` | Public users | Website visitors |
| `sales_team.group_sale_salesman` | Sales | Sales team |
| `sales_team.group_sale_manager` | Sales Manager | Sales managers |
| `account.group_account_user` | Accounting | Accountants |
| `account.group_account_manager` | Accounting Manager | Finance managers |

---

#### Quick Template

```csv
# Copy and modify this template:
id,name,model_id:id,group_id:id,perm_read,perm_write,perm_create,perm_unlink

# Read-only for regular users
access_<model>_user,<model>.user,model_<your_model>,base.group_user,1,0,0,0

# Full access for admins
access_<model>_admin,<model>.admin,model_<your_model>,base.group_system,1,1,1,1
```

### Step 3: Add Record Rules

`security/book_security.xml`

```xml
<?xml version="1.0" encoding="utf-8"?>
<!--
    noupdate="1" means:
    - These records won't be overwritten when you update the module
    - Preserves any customizations made in the UI
    - Important for security rules that might be modified by admins
-->
<odoo noupdate="1">

    <!--
        Record Rule: What records can users SEE
        - This filters the query results automatically
        - When a user searches for books, this domain is added to their query

        domain_force breakdown:
        '|' = OR operator (applies to the next 2 conditions)
        ('state', '=', 'available') = condition 1: book is available
        ('borrower_id', '=', user.partner_id.id) = condition 2: user borrowed this book

        Result: Users see books that are available OR books they borrowed
    -->
    <record id="book_user_rule" model="ir.rule">
        <field name="name">Book: User Access</field>
        <field name="model_id" ref="model_book_management"/>
        <field name="domain_force">
            ['|', ('state', '=', 'available'), ('borrower_id', '=', user.partner_id.id)]
        </field>
        <field name="groups" eval="[(4, ref('base.group_user'))]"/>
    </record>

    <!--
        Manager Rule: Managers see ALL books
        - domain_force: [(1, '=', 1)] is always true
        - This is like "WHERE 1=1" in SQL - returns all records
        - Managers bypass the user rule because this rule applies to them
    -->
    <record id="book_manager_rule" model="ir.rule">
        <field name="name">Book: Manager Access</field>
        <field name="model_id" ref="model_book_management"/>
        <field name="domain_force">[(1, '=', 1)]</field>
        <field name="groups" eval="[(4, ref('group_book_manager'))]"/>
    </record>

    <!--
        Multi-Company Rule: Users only see books from their company
        - '|' = OR
        - ('company_id', '=', False) = books with no company (global/shared)
        - ('company_id', 'in', user.company_ids.ids) = books from user's allowed companies

        Result: See global books OR books from user's companies
    -->
    <record id="book_company_rule" model="ir.rule">
        <field name="name">Book: Multi-Company</field>
        <field name="model_id" ref="model_book_management"/>
        <field name="domain_force">
            ['|', ('company_id', '=', False), ('company_id', 'in', user.company_ids.ids)]
        </field>
    </record>
</odoo>
```

### Step 4: Update Manifest

`__manifest__.py`

```python
{
    'name': 'Book Management',
    'data': [
        # ORDER MATTERS! Load in this sequence:

        'security/security_groups.xml',   # 1. Create groups first
        'security/ir.model.access.csv',   # 2. Then access rights (need groups)
        'security/book_security.xml',     # 3. Then record rules (need groups + model)
        'views/book_views.xml',           # 4. Views last (might reference groups)
    ],
}
```

### Step 5: Field-Level Security (Optional)

In model - hide field from non-managers:
```python
from odoo import models, fields

class BookManagement(models.Model):
    _name = 'book.management'

    # Only managers can see the cost_price field
    # Other users won't see this field in forms or search
    cost_price = fields.Float(
        string='Cost Price',
        groups="book_management.group_book_manager"  # group_xml_id
    )

    # Multiple groups (comma-separated)
    # Only managers AND admins can see this
    admin_notes = fields.Text(
        string='Admin Notes',
        groups="book_management.group_book_manager,base.group_system"
    )
```

In views - hide field in specific views:
```xml
<form string="Book">
    <group>
        <field name="name"/>

        <!-- Only visible to users in group_book_manager -->
        <field name="cost_price" groups="book_management.group_book_manager"/>
    </group>
</form>
```

---

## Reference Tables

### Access Rights CSV Columns

| Column | Description | Example |
|--------|-------------|---------|
| `id` | Unique identifier | `access_book_user` |
| `name` | Display name | `book.user` |
| `model_id:id` | `model_` + model name (`.` → `_`) | `model_book_management` |
| `group_id:id` | Group XML ID | `base.group_user` |
| `perm_read` | Can view records | `1` = yes, `0` = no |
| `perm_write` | Can edit records | `1` = yes, `0` = no |
| `perm_create` | Can create records | `1` = yes, `0` = no |
| `perm_unlink` | Can delete records | `1` = yes, `0` = no |

### Model ID Conversion Examples

| Model Name | Model ID in CSV |
|------------|-----------------|
| `book.management` | `model_book_management` |
| `sale.order` | `model_sale_order` |
| `res.partner` | `model_res_partner` |
| `hr.employee` | `model_hr_employee` |

### Record Rule Fields

| Field | Description | Example |
|-------|-------------|---------|
| `name` | Rule name (shown in UI) | `"Book: User Access"` |
| `model_id` | Model this rule applies to | `ref="model_book_management"` |
| `domain_force` | Filter applied to all queries | `[('user_id', '=', user.id)]` |
| `groups` | Which groups this rule affects | `eval="[(4, ref('base.group_user'))]"` |
| `perm_read` | Apply to read operations | `True` / `False` |
| `perm_write` | Apply to write operations | `True` / `False` |

### Domain Operators

| Operator | Meaning | Example |
|----------|---------|---------|
| `=` | Equals | `[('state', '=', 'done')]` |
| `!=` | Not equals | `[('state', '!=', 'cancel')]` |
| `>` | Greater than | `[('amount', '>', 100)]` |
| `<` | Less than | `[('amount', '<', 100)]` |
| `in` | In list | `[('id', 'in', [1, 2, 3])]` |
| `not in` | Not in list | `[('state', 'not in', ['draft', 'cancel'])]` |
| `like` | Pattern match | `[('name', 'like', 'John%')]` |
| `ilike` | Case-insensitive match | `[('name', 'ilike', 'john')]` |
| `child_of` | Is child of | `[('parent_id', 'child_of', parent_id)]` |

### Domain Logic Operators

| Operator | Meaning | Usage |
|----------|---------|-------|
| `'|'` | OR | `['\|', cond1, cond2]` |
| `'&'` | AND (default) | `['&', cond1, cond2]` |
| `'!'` | NOT | `['!', condition]` |

### Common Domains

```python
# Single condition: state equals 'done'
[('state', '=', 'done')]

# AND (implicit - conditions in same list)
# state = 'done' AND amount > 100
[('state', '=', 'done'), ('amount', '>', 100)]

# OR (explicit with '|')
# state = 'done' OR state = 'cancel'
['|', ('state', '=', 'done'), ('state', '=', 'cancel')]

# Current user's records
[('user_id', '=', user.id)]

# Current company
[('company_id', '=', user.company_id.id)]

# Multi-company safe (global OR user's companies)
['|', ('company_id', '=', False), ('company_id', 'in', user.company_ids.ids)]

# Active records only
[('active', '=', True)]

# Complex: (A AND B) OR C
['|', '&', ('a', '=', 1), ('b', '=', 2), ('c', '=', 3)]
```

### Built-in Groups

| Group XML ID | Description | Use Case |
|--------------|-------------|----------|
| `base.group_user` | All internal users | Default for employees |
| `base.group_system` | Administrators | Full system access |
| `base.group_portal` | Portal users | External customers |
| `base.group_public` | Public users | Not logged in |
| `base.group_no_one` | Nobody | Hide field from all |

---

## Python Security Methods

```python
from odoo.exceptions import AccessError

# ============================================
# sudo() - Run as superuser (bypasses ALL security)
# ============================================
# WARNING: Use sparingly - bypasses record rules AND access rights
# Use when system needs to access data regardless of user permissions

# Example: System cron job needs to process all records
books = self.env['book.management'].sudo().search([])

# Example: Auto-assign regardless of user's permissions
self.sudo().write({'assigned_to': user.id})


# ============================================
# has_group() - Check if user is in a group
# ============================================
# Returns True/False

if self.env.user.has_group('book_management.group_book_manager'):
    # User is a manager - allow sensitive operation
    return self._do_sensitive_action()
else:
    raise AccessError("Only managers can do this")


# ============================================
# check_access() - Verify specific permission
# ============================================
# Raises exception if user doesn't have access
# Available: 'read', 'write', 'create', 'unlink'

try:
    self.check_access('write')
    # User can write - proceed
    self.write({'field': 'value'})
except:
    return {'error': 'No write permission'}


# ============================================
# AccessError - Raise access denied error
# ============================================
from odoo.exceptions import AccessError
from odoo.tools.translate import _

if not self.env.user.has_group('book_management.group_book_manager'):
    raise AccessError(_("You don't have permission to access this feature."))
```

---

## Directory Structure

```
custom_addons/book_management/
├── __init__.py
├── __manifest__.py
├── models/
│   └── book.py
├── views/
│   └── book_views.xml
├── security/
│   ├── security_groups.xml      # Step 1: Custom groups
│   ├── ir.model.access.csv      # Step 2: CRUD permissions
│   └── book_security.xml        # Step 3: Record rules
└── controllers/
    └── main.py
```

---

## Debugging

| Issue | Check | Location |
|-------|-------|----------|
| Records not showing | Record rules domain | Settings > Technical > Security > Record Rules |
| Access denied error | Access rights CSV | Settings > Technical > Security > Access Rights |
| Field not visible | groups attribute | Check model and view |
| Can't delete | perm_unlink = 1 | Access rights CSV |

### Debug in Python

```python
# Print all rules for a model
rules = request.env['ir.rule'].search([
    ('model_id.model', '=', 'book.management')
])
for rule in rules:
    print(f"Rule: {rule.name}")
    print(f"  Domain: {rule.domain_force}")
    print(f"  Groups: {rule.groups.mapped('name')}")

# Check user's groups
user = request.env.user
print(f"User: {user.name}")
print(f"Groups: {user.groups_id.mapped('name')}")
```

### Debug in SQL

```sql
-- Check access rights for a model
SELECT name, perm_read, perm_write, perm_create, perm_unlink
FROM ir_model_access
WHERE model_id = (SELECT id FROM ir_model WHERE model = 'book.management');

-- Check record rules
SELECT name, domain_force, groups
FROM ir_rule
WHERE model_id = (SELECT id FROM ir_model WHERE model = 'book.management');
```
