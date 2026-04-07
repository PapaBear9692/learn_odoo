# Odoo ORM Database Relationships

This guide explains how database relationships work in Odoo with code examples.

---

## Relationship Types Overview

| Relationship | Field Type | Example |
|--------------|------------|---------|
| Many-to-One | `Many2one` | Employee → Department |
| One-to-Many | `One2many` | Department → Employees |
| Many-to-Many | `Many2many` | Student ↔ Courses |
| One-to-One | `Many2one` + unique | User → Profile |

---

## 1. Many-to-One (N:1)

### Meaning
- **Many** employees belong to **one** department
- Each employee has ONE department
- Department can have MANY employees

**Example:** Employee → Department

```
┌─────────────┐         ┌─────────────┐
│  Employee   │         │ Department  │
├─────────────┤         ├─────────────┤
│ John        │────────►│ Sales       │
│ Jane        │────────►│ Sales       │
│ Bob         │────────►│ IT          │
└─────────────┘         └─────────────┘
```

### Code

```python
from odoo import models, fields

class Employee(models.Model):
    _name = 'my.employee'
    _description = 'Employee'

    name = fields.Char(
        string='Name',           # Label shown in UI
        required=True,           # Must be filled
    )

    # ═══════════════════════════════════════════════════════════════
    # MANY2ONE - Employee belongs to ONE Department
    # ═══════════════════════════════════════════════════════════════
    # This creates a foreign key column in the database
    # The field stores the ID of the related department

    # Here 'Many' is for this class (Employee), One is for target class (department)
    department_id = fields.Many2one(    
        'my.department',          # Target model (what we're linking to)
                                   # This MUST be the _name of the other model

        string='Department',      # Label shown in UI

        ondelete='set null',      # What happens when department is deleted:
                                   # 'set null' = this field becomes empty
                                   # 'cascade' = delete employee too
                                   # 'restrict' = prevent department deletion

        help='Select the department this employee belongs to',
    )
```

### How to Use

```python
# Create employee with department
employee = self.env['my.employee'].create({
    'name': 'John Doe',
    'department_id': 1,  # ID of the department
})

# Get department info from employee
print(employee.department_id.name)      # "Sales"
print(employee.department_id.id)        # 1

# Search employees in a department
employees = self.env['my.employee'].search([
    ('department_id', '=', 1)
])
```

---

## ⚠️ Important: Extending Existing Modules

When you add a relationship to an **existing module's model**, you should also **extend that target model** to add the inverse relationship.

### Scenario: Adding Custom Field to hr.employee

**Your custom module:** `custom_addons/hr_extension`

```python
# ═══════════════════════════════════════════════════════════════
# models/hr_employee.py - Add Many2one to Employee
# ═══════════════════════════════════════════════════════════════

from odoo import models, fields

class HrEmployee(models.Model):
    _inherit = 'hr.employee'  # Extend existing Odoo model

    # Add new Many2one field
    custom_department_id = fields.Many2one(
        'hr.department',
        string='Custom Department',
    )
```

**You should also add the inverse (optional but recommended):**

```python
# ═══════════════════════════════════════════════════════════════
# models/hr_department.py - Add One2many to Department
# ═══════════════════════════════════════════════════════════════

from odoo import models, fields

class HrDepartment(models.Model):
    _inherit = 'hr.department'  # Extend existing Odoo model

    # Add the inverse One2many field
    custom_employee_ids = fields.One2many(
        'hr.employee',              # Target model
        'custom_department_id',     # The field we added in hr.employee
        string='Custom Employees',
    )
```

### Why Extend Both Sides?

| Without Inverse | With Inverse |
|-----------------|--------------|
| `employee.custom_department_id.name` ✅ | Same ✅ |
| Cannot access: `dept.custom_employee_ids` ❌ | Can access: `dept.custom_employee_ids` ✅ |
| Search only from employee side | Search from both sides ✅ |
| UI: Can't show employees in department form | UI: Can show employees in department form ✅ |

### Directory Structure for Extension

```
custom_addons/hr_extension/
├── __init__.py
├── __manifest__.py
├── models/
│   ├── __init__.py
│   ├── hr_employee.py       # Extend employee
│   └── hr_department.py     # Extend department (inverse)
└── views/
    ├── hr_employee_views.xml
    └── hr_department_views.xml
```

### __init__.py for Models

```python
# models/__init__.py
from . import hr_employee
from . import hr_department  # Don't forget to import!
```

### Manifest Dependencies

```python
# __manifest__.py
{
    'name': 'HR Extension',
    'version': '1.0',
    'depends': ['hr'],  # Must depend on the module you're extending
    'data': [
        'views/hr_employee_views.xml',
        'views/hr_department_views.xml',
    ],
}
```

---

## 2. One-to-Many (1:N)

### Meaning
- **One** department has **many** employees
- This is the INVERSE of Many2one
- Both sides need to be defined

**Example:** Department → Employees

```
┌─────────────┐
│ Department  │
│   Sales     │
├─────────────┤
│ Employees:  │
│  - John     │
│  - Jane     │
│  - Bob      │
└─────────────┘
```

### Code

```python
from odoo import models, fields

class Department(models.Model):
    _name = 'my.department'
    _description = 'Department'

    name = fields.Char(
        string='Department Name',
        required=True,
    )

    # ═══════════════════════════════════════════════════════════════
    # ONE2MANY - One Department has MANY Employees
    # ═══════════════════════════════════════════════════════════════
    # This does NOT create a new column in database
    # It's a VIRTUAL field that queries the related model

    # Here 'One' refers to this class (Department), 'Many' refers to target class (Employee)
    employee_ids = fields.One2many(
        'my.employee',             # Target model (Employee)

        'department_id',           # The Many2one field IN THE TARGET model
                                   # This MUST match the field name in my.employee
                                   # that points back to this model

        string='Employees',        # Label in UI

        help='List of employees in this department',
    )


class Employee(models.Model):
    _name = 'my.employee'
    _description = 'Employee'

    name = fields.Char(string='Name', required=True)

    # This is the "other side" of the relationship
    # One2many in Department points to THIS field
    department_id = fields.Many2one(
        'my.department',
        string='Department',
        ondelete='set null',
    )
```

### How to Use

```python
# Get all employees in a department
department = self.env['my.department'].browse(1)
for emp in department.employee_ids:
    print(emp.name)  # John, Jane, Bob

# Count employees
count = len(department.employee_ids)

# Add employee to department (via employee creation)
self.env['my.employee'].create({
    'name': 'New Employee',
    'department_id': department.id,
})

# Add employee using One2many (from department side)
department.write({
    'employee_ids': [(0, 0, {'name': 'New Employee'})]  # Create new
})

# Remove employee from department
employee.department_id = False  # Set to empty
```

### One2many Write Operations

```python
# (0, 0, {values}) - CREATE new record and link it
department.write({
    'employee_ids': [(0, 0, {'name': 'New Guy'})]
})

# (1, id, {values}) - UPDATE existing record
department.write({
    'employee_ids': [(1, employee_id, {'name': 'Updated Name'})]
})

# (2, id, 0) - UNLINK (remove from relation, don't delete)
department.write({
    'employee_ids': [(2, employee_id, 0)]
})

# (3, id, 0) - UNLINK (same as above)
department.write({
    'employee_ids': [(3, employee_id, 0)]
})

# (4, id, 0) - LINK existing record
department.write({
    'employee_ids': [(4, existing_employee_id, 0)]
})

# (5, 0, 0) - UNLINK ALL records
department.write({
    'employee_ids': [(5, 0, 0)]
})

# (6, 0, [ids]) - REPLACE ALL with these ids
department.write({
    'employee_ids': [(6, 0, [1, 2, 3])]  # Only these 3 employees
})
```

---

## 3. Many-to-Many (N:N)

### Meaning
- **Many** students can enroll in **many** courses
- **Many** courses can have **many** students
- Creates a third "junction" table automatically

**Example:** Students ↔ Courses

```
Students          Junction Table           Courses
┌────────┐       ┌──────────────┐        ┌────────┐
│ John   │       │ student │course│        │ Math   │
│ Jane   │──────►│    1   │  1   │◄───────│ English│
│ Bob    │       │    1   │  2   │        │ Science│
└────────┘       │    2   │  1   │        └────────┘
                 │    2   │  3   │
                 │    3   │  2   │
                 └──────────────┘
```

### Code

```python
from odoo import models, fields

class Student(models.Model):
    _name = 'my.student'
    _description = 'Student'

    name = fields.Char(string='Name', required=True)

    # ═══════════════════════════════════════════════════════════════
    # MANY2MANY - Student can enroll in MANY Courses
    # ═══════════════════════════════════════════════════════════════
    # This creates:
    # 1. A junction table in database (my_student_course_rel)
    # 2. Links students to courses via this table

    course_ids = fields.Many2many(
        'my.course',               # Target model

        'my_student_course_rel',   # Junction table name (optional)
                                   # If omitted: auto-generated as model1_model2_rel
                                   # Convention: put both model names alphabetically

        'student_id',              # Column name for THIS model in junction table
        'course_id',               # Column name for TARGET model in junction table

        string='Courses',          # Label in UI

        help='Courses this student is enrolled in',
    )


class Course(models.Model):
    _name = 'my.course'
    _description = 'Course'

    name = fields.Char(string='Course Name', required=True)

    # ═══════════════════════════════════════════════════════════════
    # MANY2MANY - Course can have MANY Students (inverse side)
    # ═══════════════════════════════════════════════════════════════
    # Same junction table, but columns reversed

    student_ids = fields.Many2many(
        'my.student',              # Target model (back to student)

        'my_student_course_rel',   # SAME junction table name as above!

        'course_id',               # This model's column (reversed)
        'student_id',              # Target model's column (reversed)

        string='Students',
    )
```

### Simplified Many2many (Auto-names)

```python
# If you don't care about table/column names, use this:

class Student(models.Model):
    _name = 'my.student'

    # Odoo auto-generates:
    # - Table: my_student_my_course_rel
    # - Columns: my_student_id, my_course_id
    course_ids = fields.Many2many(
        'my.course',
        string='Courses',
    )


class Course(models.Model):
    _name = 'my.course'

    student_ids = fields.Many2many(
        'my.student',
        string='Students',
    )
```

### How to Use

```python
# Create student with courses
student = self.env['my.student'].create({
    'name': 'John',
    'course_ids': [(4, course_id) for course_id in [1, 2, 3]],
})

# Get all courses for a student
for course in student.course_ids:
    print(course.name)  # Math, English, Science

# Get all students in a course
course = self.env['my.course'].browse(1)
for student in course.student_ids:
    print(student.name)

# Check if student is enrolled in course
if course in student.course_ids:
    print("Enrolled!")

# Add course to student
student.write({
    'course_ids': [(4, new_course_id)]  # Add one course
})

# Remove course from student
student.write({
    'course_ids': [(3, course_id)]  # Remove one course
})

# Replace all courses
student.write({
    'course_ids': [(6, 0, [1, 2, 3])]  # Only these courses
})
```

---

## 4. One-to-One (1:1)

### Meaning
- **One** user has **one** profile
- **One** profile belongs to **one** user
- This is a Many2one with `copy=False` to ensure uniqueness

**Example:** User ↔ Profile

```
┌─────────┐         ┌─────────┐
│  User   │◄────────►│ Profile │
│  John   │    1:1   │  John's │
└─────────┘         │ Profile │
                    └─────────┘
```

### Code

```python
from odoo import models, fields

class UserProfile(models.Model):
    _name = 'my.user.profile'
    _description = 'User Profile'

    name = fields.Char(string='Profile Name')

    # ═══════════════════════════════════════════════════════════════
    # ONE-TO-ONE using Many2one
    # ═══════════════════════════════════════════════════════════════
    # copy=False prevents duplication (ensures uniqueness)
    # index=True improves search performance

    user_id = fields.Many2one(
        'res.users',              # Link to Odoo's built-in User model

        string='User',

        ondelete='cascade',       # Delete profile when user is deleted

        copy=False,               # Don't copy this when duplicating profile
                                   # This helps prevent two profiles for same user

        index=True,               # Add database index for faster lookups
    )

    bio = fields.Text(string='Biography')
    avatar = fields.Binary(string='Avatar Image')


class Users(models.Model):
    _inherit = 'res.users'  # Extend Odoo's built-in User model

    # ═══════════════════════════════════════════════════════════════
    # ONE2ONE inverse - using One2many but with singular naming
    # ═══════════════════════════════════════════════════════════════

    profile_id = fields.One2many(
        'my.user.profile',        # Target model

        'user_id',                # The field in UserProfile that links here

        string='Profile',

        limit=1,                  # Only fetch one record
    )
```

### Better One-to-One Approach

```python
from odoo import models, fields, api
from odoo.exceptions import ValidationError

class UserProfile(models.Model):
    _name = 'my.user.profile'

    user_id = fields.Many2one(
        'res.users',
        string='User',
        ondelete='cascade',
        copy=False,
        index=True,
    )

    # SQL constraint to ensure ONE profile per user
    _sql_constraints = [
        (
            'user_unique',               # Constraint name
            'UNIQUE(user_id)',           # SQL constraint
            'Each user can only have one profile!'  # Error message
        ),
    ]
```

### How to Use

```python
# Create profile for user
profile = self.env['my.user.profile'].create({
    'user_id': user.id,
    'bio': 'Software developer',
})

# Get profile from user
user = self.env['res.users'].browse(1)
if user.profile_id:
    print(user.profile_id.bio)

# Get user from profile
print(profile.user_id.name)
```

---

## 5. Self-Referencing (Hierarchical)

### Meaning
- A record links to another record of the **same model**
- Used for trees/hierarchies: Categories, Employees (manager), etc.

**Example:** Employee → Manager (also an Employee)

```
            CEO (John)
               │
       ┌───────┴───────┐
    Manager        Manager
    (Jane)         (Bob)
       │               │
    Employee      Employee
    (Alice)       (Charlie)
```

### Code

```python
from odoo import models, fields

class Employee(models.Model):
    _name = 'my.employee'
    _description = 'Employee'

    name = fields.Char(string='Name', required=True)

    # ═══════════════════════════════════════════════════════════════
    # SELF-REFERENCING Many2one - Employee has a Manager
    # ═══════════════════════════════════════════════════════════════
    # Points to the SAME model (my.employee)

    manager_id = fields.Many2one(
        'my.employee',            # Same model! Self-reference

        string='Manager',

        ondelete='set null',      # If manager deleted, this becomes empty

        help='Direct manager of this employee',
    )

    # ═══════════════════════════════════════════════════════════════
    # SELF-REFERENCING One2many - Manager has Subordinates
    # ═══════════════════════════════════════════════════════════════

    subordinate_ids = fields.One2many(
        'my.employee',            # Same model

        'manager_id',             # The field above (points back)

        string='Subordinates',

        help='Employees reporting to this person',
    )


class ProductCategory(models.Model):
    _name = 'my.category'
    _description = 'Product Category'

    name = fields.Char(string='Name', required=True)

    # Parent category (this is a subcategory of...)
    parent_id = fields.Many2one(
        'my.category',
        string='Parent Category',
        ondelete='cascade',
    )

    # Child categories (this category contains...)
    child_ids = fields.One2many(
        'my.category',
        'parent_id',
        string='Subcategories',
    )

    # Get all parent IDs as a list (for breadcrumbs)
    parent_path = fields.Char(
        string='Parent Path',
        index=True,
        # Stores path like "1/5/12" for category hierarchy
    )
```

### How to Use

```python
# Create hierarchy
ceo = self.env['my.employee'].create({'name': 'John (CEO)'})

manager1 = self.env['my.employee'].create({
    'name': 'Jane (Manager)',
    'manager_id': ceo.id,
})

employee1 = self.env['my.employee'].create({
    'name': 'Alice (Employee)',
    'manager_id': manager1.id,
})

# Get manager of employee
print(employee1.manager_id.name)  # Jane (Manager)

# Get subordinates of manager
for sub in manager1.subordinate_ids:
    print(sub.name)  # Alice (Employee)

# Get all subordinates recursively (CEO sees everyone)
def get_all_subordinates(employee):
    result = employee.subordinate_ids
    for sub in employee.subordinate_ids:
        result += get_all_subordinates(sub)
    return result

all_employees = get_all_subordinates(ceo)
```

---

## 6. Related Fields

### Meaning
- Shortcut to access fields from related records
- No database column (computed automatically)
- Useful to show related data without extra queries

### Code

```python
from odoo import models, fields

class Employee(models.Model):
    _name = 'my.employee'

    name = fields.Char(string='Name')

    department_id = fields.Many2one(
        'my.department',
        string='Department',
    )

    # ═══════════════════════════════════════════════════════════════
    # RELATED FIELD - Access department info directly from employee
    # ═══════════════════════════════════════════════════════════════
    # This is like: employee.department_id.name
    # But you can use it like: employee.department_name

    department_name = fields.Char(
        related='department_id.name',  # Path to the field
        string='Department Name',
        store=True,                     # If True: saves to DB (faster reads)
                                        # If False: computed each time
        readonly=True,                  # Usually read-only
    )

    department_code = fields.Char(
        related='department_id.code',
        string='Dept Code',
        store=True,
    )

    # Related field from related field (multi-level)
    # employee.department_id.company_id.name
    company_name = fields.Char(
        related='department_id.company_id.name',
        string='Company',
        store=True,
    )
```

### How to Use

```python
# Without related field
employee = self.env['my.employee'].browse(1)
print(employee.department_id.name)      # Two database queries

# With related field (store=True)
print(employee.department_name)          # One database query

# Search by related field (needs store=True)
employees = self.env['my.employee'].search([
    ('department_name', '=', 'Sales')   # Can search!
])
```

---

## 7. Computed Fields with Relations

### Code

```python
from odoo import models, fields, api

class Department(models.Model):
    _name = 'my.department'

    name = fields.Char(string='Name')

    employee_ids = fields.One2many(
        'my.employee',
        'department_id',
        string='Employees',
    )

    # ═══════════════════════════════════════════════════════════════
    # COMPUTED FIELD - Count employees automatically
    # ═══════════════════════════════════════════════════════════════

    employee_count = fields.Integer(
        string='Employee Count',
        compute='_compute_employee_count',  # Method name
        store=True,                          # Save to DB
    )

    @api.depends('employee_ids')             # Recompute when employees change
    def _compute_employee_count(self):
        for dept in self:
            dept.employee_count = len(dept.employee_ids)
```

---

## Quick Reference

### Field Type Summary

| Type | Creates DB Column | Use Case |
|------|-------------------|----------|
| `Many2one` | Yes (foreign key) | Child → Parent |
| `One2many` | No (virtual) | Parent → Children |
| `Many2many` | Yes (junction table) | Peer ↔ Peer |
| `related` | Optional | Shortcut to related field |

### Common Parameters

| Parameter | Description |
|-----------|-------------|
| `string` | UI label |
| `required` | Must be filled |
| `readonly` | Cannot edit |
| `ondelete` | Action when related record deleted |
| `domain` | Filter available options |
| `context` | Pass data to related views |
| `help` | Tooltip text |

### ondelete Options

| Value | Action |
|-------|--------|
| `'cascade'` | Delete this record when parent deleted |
| `'set null'` | Set field to empty when parent deleted |
| `'restrict'` | Prevent parent deletion if children exist |

---

## Complete Example: Library System

```python
from odoo import models, fields, api

class Library(models.Model):
    _name = 'my.library'
    _description = 'Library'

    name = fields.Char(string='Library Name')

    # One library has many books
    book_ids = fields.One2many(
        'my.book',
        'library_id',
        string='Books',
    )

    # Computed: count books
    book_count = fields.Integer(
        compute='_compute_book_count',
        store=True,
    )

    @api.depends('book_ids')
    def _compute_book_count(self):
        for library in self:
            library.book_count = len(library.book_ids)


class Book(models.Model):
    _name = 'my.book'
    _description = 'Book'

    name = fields.Char(string='Title', required=True)
    isbn = fields.Char(string='ISBN')

    # Book belongs to one library
    library_id = fields.Many2one(
        'my.library',
        string='Library',
        ondelete='cascade',
    )

    # Related field for quick access
    library_name = fields.Char(
        related='library_id.name',
        string='Library Name',
        store=True,
    )

    # Book has one author
    author_id = fields.Many2one(
        'my.author',
        string='Author',
    )

    # Book can have many categories
    category_ids = fields.Many2many(
        'my.category',
        'my_book_category_rel',
        'book_id',
        'category_id',
        string='Categories',
    )


class Author(models.Model):
    _name = 'my.author'
    _description = 'Author'

    name = fields.Char(string='Name', required=True)

    # Author has many books
    book_ids = fields.One2many(
        'my.book',
        'author_id',
        string='Books',
    )


class Category(models.Model):
    _name = 'my.category'
    _description = 'Category'

    name = fields.Char(string='Name', required=True)

    # Self-referencing for subcategories
    parent_id = fields.Many2one(
        'my.category',
        string='Parent Category',
    )

    child_ids = fields.One2many(
        'my.category',
        'parent_id',
        string='Subcategories',
    )

    # Many books can be in many categories
    book_ids = fields.Many2many(
        'my.book',
        'my_book_category_rel',
        'category_id',
        'book_id',
        string='Books',
    )
```
