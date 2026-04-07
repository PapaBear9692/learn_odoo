# Odoo ORM Query Optimization (N+1 Problem)

## What is the N+1 Problem?

**N+1** means you run **1 query to get N records**, then **N extra queries** to get related data for each record.

```
You want:     1 query
You get:      1 + N queries  ← This is SLOW
```

---

## The Problem Explained

### ❌ Bad Code (N+1 Queries)

```python
# Get 100 employees
employees = self.env['hr.employee'].search([])

for emp in employees:
    # This queries the database EVERY time!
    # 100 employees = 100 extra queries!
    print(emp.department_id.name)
```

**What happens in database:**
```sql
-- Query 1: Get all employees
SELECT * FROM hr_employee;

-- Then for EACH employee (100 times!):
SELECT * FROM hr_department WHERE id = 1;  -- Employee 1's department
SELECT * FROM hr_department WHERE id = 2;  -- Employee 2's department
SELECT * FROM hr_department WHERE id = 3;  -- Employee 3's department
-- ... 97 more queries!
```

**Total: 101 queries** (1 + 100)

---

## Eager Loading vs Lazy Loading

### What is Lazy Loading? (Default in Odoo)

**Lazy Loading** = Load related data **only when you access it**

```python
employees = self.env['hr.employee'].search([])

# At this point: Only employee data is loaded (1 query)
# Department data is NOT loaded yet

for emp in employees:
    # NOW it queries department (lazy - loads when accessed)
    print(emp.department_id.name)  # Triggers query each time!
```

**Problem:** Triggers N+1 queries because each access = new query

### What is Eager Loading?

**Eager Loading** = Load related data **upfront, before you need it**

```python
employees = self.env['hr.employee'].search([])

# Prefetch ALL departments at once (eager loading)
employees.mapped('department_id.name')  # 1 query for all departments

for emp in employees:
    print(emp.department_id.name)  # No query! Already loaded
```

**Benefit:** Only 2 queries total (1 for employees, 1 for all departments)

---

## Comparison Table

| Aspect | Lazy Loading | Eager Loading |
|--------|--------------|---------------|
| **When loaded** | On access | Upfront |
| **Queries** | N+1 | 2 |
| **Best for** | Single record access | List iterations |
| **Odoo default** | ✅ Yes | ❌ No (must trigger) |
| **Memory usage** | Low initially | Higher upfront |

### When to Use Each

| Use Lazy Loading When | Use Eager Loading When |
|-----------------------|------------------------|
| Accessing single record | Iterating over many records |
| Not sure if data needed | You KNOW you need related data |
| Building interactive UI | API responses, reports |
| Memory is limited | Performance is critical |

---

## Solution Categories

| Method | Loading Type | How it Works |
|--------|--------------|--------------|
| `mapped()` | **Eager** | Prefetches all related records in one query |
| `search_read()` | **Eager** | Fetches all fields in single query |
| `read_group()` | **Eager** | Aggregates in single query |
| `with_prefetch()` | **Eager** | Marks records for batch prefetching |
| Direct access (`emp.department_id`) | **Lazy** | Queries on each access |

---

## Solutions

### ✅ Solution 1: Use `mapped()` — 🔴 EAGER LOADING

**How it works:** Prefetches ALL related records in ONE query upfront

```python
# Get all employees
employees = self.env['hr.employee'].search([])

# mapped() prefetches related data in ONE query
department_names = employees.mapped('department_id.name')

# Now iterate - no extra queries!
for emp in employees:
    print(emp.department_id.name)  # Already loaded!
```

**Total: 2 queries** (1 for employees, 1 for departments)

#### 🎯 Best Use Case for `mapped()`

| ✅ Use When | ❌ Don't Use When |
|------------|-------------------|
| You need to iterate over records AND access related fields | You only need a flat list of values |
| You need the recordset for further operations | You only need aggregated data (count, sum) |
| Working with nested relations (chain multiple mapped) | Building JSON for API (use `search_read` instead) |
| You need to modify records after reading | |

**Perfect for:**
- Business logic that processes records
- Reports that need full record access
- Nested relations like `employee.department_id.company_id.name`

---

### ✅ Solution 2: Use `read_group()` for Aggregations — 🔴 EAGER LOADING

**How it works:** Fetches aggregated data in ONE query with GROUP BY

```python
# ❌ BAD: Count employees per department (N+1)
departments = self.env['hr.department'].search([])
for dept in departments:
    count = self.env['hr.employee'].search_count([('department_id', '=', dept.id)])

# ✅ GOOD: One query with GROUP BY
results = self.env['hr.employee'].read_group(
    [('department_id', '!=', False)],  # Domain filter
    ['department_id'],                  # Fields to group by
    ['department_id'],                  # Groupby fields
)

for result in results:
    print(f"Department: {result['department_id'][1]}, Count: {result['department_id_count']}")
```

**Output:**
```
Department: Sales, Count: 15
Department: IT, Count: 8
Department: HR, Count: 5
```

#### 🎯 Best Use Case for `read_group()`

| ✅ Use When | ❌ Don't Use When |
|------------|-------------------|
| You need COUNT, SUM, AVG, MIN, MAX per group | You need individual record details |
| Building dashboards with statistics | You need to modify the records |
| Creating pivot tables / reports | You need all fields (use `search_read`) |
| Grouping data by category, status, date, etc. | |

**Perfect for:**
- Dashboard statistics
- Summary reports
- Counting records by group
- Calculating totals per category

**Aggregations supported:**
```python
# Count
results = model.read_group(domain, ['field'], ['group_field'])
count = results[0]['field_count']  # _count suffix

# Sum
results = model.read_group(domain, ['amount'], ['category'])
total = results[0]['amount']  # Sum is default for numeric fields

# Average (use field:avg)
results = model.read_group(domain, ['amount:avg'], ['category'])
avg = results[0]['amount']
```

---

### ✅ Solution 3: Prefetch with `with_prefetch()` — 🔴 EAGER LOADING

**How it works:** Marks records for batch prefetching (Odoo's internal optimization)

```python
# Tell Odoo to prefetch these records together
employees = self.env['hr.employee'].with_prefetch().search([])

# All department lookups are batched
for emp in employees:
    print(emp.department_id.name)
```

#### 🎯 Best Use Case for `with_prefetch()`

| ✅ Use When | ❌ Don't Use When |
|------------|-------------------|
| You want automatic prefetching without `mapped()` | You need explicit control (use `mapped`) |
| Complex queries with multiple related fields | Simple single-field access |
| Working with large recordsets | Small recordsets (overhead not worth it) |

**Note:** Odoo 14+ does this automatically in most cases. Less commonly needed now.

---

### ✅ Solution 4: Use `search_read()` for API Responses — 🔴 EAGER LOADING

**How it works:** Fetches all specified fields (including Many2one display names) in ONE query

```python
# ❌ BAD: Multiple queries
employees = self.env['hr.employee'].search([])
data = []
for emp in employees:
    data.append({
        'id': emp.id,
        'name': emp.name,
        'department': emp.department_id.name,  # Extra query each time!
    })

# ✅ GOOD: One query, specify all fields
employees = self.env['hr.employee'].search_read(
    [],  # domain
    ['name', 'department_id'],  # fields to fetch
)

# Result already includes department_id
# department_id is returned as [id, name] pair
for emp in employees:
    print(emp['name'], emp['department_id'])  # [5, 'Sales']
```

#### 🎯 Best Use Case for `search_read()`

| ✅ Use When | ❌ Don't Use When |
|------------|-------------------|
| Building JSON/XML API responses | You need recordset methods |
| Frontend data fetching | You need to call methods on records |
| You only need specific fields | You need computed fields without `store=True` |
| Exporting data | You need to write/update records |

**Perfect for:**
- REST/JSON-RPC API endpoints
- JavaScript frontend data
- Data export functionality
- Read-only list views

**Return format:**
```python
# Returns list of dicts (not recordset!)
[
    {'id': 1, 'name': 'John', 'department_id': [5, 'Sales']},
    {'id': 2, 'name': 'Jane', 'department_id': [3, 'IT']},
]
# Many2one returns [id, display_name]
# One2many returns [id1, id2, id3]
# Many2many returns [id1, id2, id3]
```

---

## Common Usecases

### Usecase 1: Display List with Related Fields

```python
# ═══════════════════════════════════════════════════════════════
# GOAL: Show employee list with department name
# ═══════════════════════════════════════════════════════════════

# ❌ BAD: N+1 problem
def get_employee_list_bad(self):
    employees = self.env['hr.employee'].search([])
    result = []
    for emp in employees:
        result.append({
            'id': emp.id,
            'name': emp.name,
            'department': emp.department_id.name,  # Query each time!
            'manager': emp.parent_id.name,          # Another query!
        })
    return result

# ✅ GOOD: Use mapped() to prefetch
def get_employee_list_good(self):
    employees = self.env['hr.employee'].search([])

    # Prefetch related data
    employees.mapped('department_id.name')
    employees.mapped('parent_id.name')

    result = []
    for emp in employees:
        result.append({
            'id': emp.id,
            'name': emp.name,
            'department': emp.department_id.name,  # No extra query
            'manager': emp.parent_id.name,          # No extra query
        })
    return result

# ✅ BEST: Use search_read()
def get_employee_list_best(self):
    return self.env['hr.employee'].search_read(
        [],
        ['name', 'department_id', 'parent_id']
    )
```

---

### Usecase 2: Calculate Totals by Group

```python
# ═══════════════════════════════════════════════════════════════
# GOAL: Get total salary per department
# ═══════════════════════════════════════════════════════════════

# ❌ BAD: N+1 queries
def get_salary_by_department_bad(self):
    departments = self.env['hr.department'].search([])
    result = {}
    for dept in departments:
        employees = self.env['hr.employee'].search([('department_id', '=', dept.id)])
        total = sum(emp.wage for emp in employees)  # Another query per dept!
        result[dept.name] = total
    return result

# ✅ GOOD: Single read_group query
def get_salary_by_department_good(self):
    results = self.env['hr.employee'].read_group(
        [('department_id', '!=', False)],
        ['wage', 'department_id'],
        ['department_id'],
    )

    return {
        r['department_id'][1]: r['wage']  # [1] is name, 'wage' is sum
        for r in results
    }
```

---

### Usecase 3: Get Nested Relations

```python
# ═══════════════════════════════════════════════════════════════
# GOAL: Employee → Department → Company
# ═══════════════════════════════════════════════════════════════

# ❌ BAD: 3 levels of queries
employees = self.env['hr.employee'].search([])
for emp in employees:
    # Each line = potential query!
    company = emp.department_id.company_id.name

# ✅ GOOD: Prefetch all levels
employees = self.env['hr.employee'].search([])

# Prefetch nested relations
employees.mapped('department_id.company_id.name')

for emp in employees:
    print(emp.department_id.company_id.name)  # No queries!
```

---

### Usecase 4: API Endpoint Optimization

```python
from odoo import http
from odoo.http import request

class EmployeeAPI(http.Controller):

    @http.route('/api/employees', type='jsonrpc', auth='user', methods=['POST'])
    def list_employees(self, **kwargs):
        # ═══════════════════════════════════════════════════════════════
        # Optimized API - minimal queries
        # ═══════════════════════════════════════════════════════════════

        employees = self.env['hr.employee'].search([])

        # Prefetch all related fields we need
        employees.mapped('department_id.name')
        employees.mapped('parent_id.name')

        # Now build response - no extra queries
        return {
            'employees': [
                {
                    'id': emp.id,
                    'name': emp.name,
                    'department': emp.department_id.name or '',
                    'manager': emp.parent_id.name or '',
                }
                for emp in employees
            ]
        }
```

---

### Usecase 5: Batch Operations

```python
# ═══════════════════════════════════════════════════════════════
# GOAL: Update related records efficiently
# ═══════════════════════════════════════════════════════════════

# ❌ BAD: Update one by one
employees = self.env['hr.employee'].search([('active', '=', True)])
for emp in employees:
    emp.write({'notes': 'Updated'})  # N queries!

# ✅ GOOD: Batch update in ONE query
employees = self.env['hr.employee'].search([('active', '=', True)])
employees.write({'notes': 'Updated'})  # 1 query!

# ✅ EVEN BETTER: Direct SQL for mass updates (advanced)
self.env.cr.execute(
    "UPDATE hr_employee SET notes = %s WHERE active = %s",
    ('Updated', True)
)
```

---

## Quick Reference

### Decision Matrix: Which Solution to Use?

```
What do you need?
│
├─► Count, Sum, Average per group?
│   └─► Use read_group()
│
├─► Data for API/JSON response?
│   └─► Use search_read()
│
├─► Full records + related fields + more processing?
│   └─► Use mapped()
│
├─► Nested relations (A.B.C)?
│   └─► Use mapped() with chaining
│
└─► Mass update records?
    └─► Use write() on full recordset
```

### When to Use What

| Situation | Best Solution | Why |
|-----------|---------------|-----|
| Count employees per department | `read_group()` | Single GROUP BY query |
| Sum salary per department | `read_group()` | Single aggregation query |
| API: Return employee list | `search_read()` | Returns dict, includes related IDs |
| Process records + access relations | `mapped()` | Prefetches, keeps recordset |
| Nested: employee.dept.company | `mapped()` chain | Handles multi-level prefetch |
| Update 1000 records | `recordset.write()` | Single UPDATE query |
| Get just the names list | `mapped()` | Returns flat list |

### Comparison Table

| Method | Loading | Returns | Queries | Best For |
|--------|---------|---------|---------|----------|
| Loop + direct access | 🟡 Lazy | - | 1 + N | ❌ Never use |
| `mapped()` | 🔴 Eager | Recordset/list | 2 | Processing with relations |
| `search_read()` | 🔴 Eager | List[dict] | 1 | API responses |
| `read_group()` | 🔴 Eager | List[dict] | 1 | Statistics/aggregations |
| `with_prefetch()` | 🔴 Eager | Recordset | 2 | Advanced prefetching |

### Loading Type Legend

| Symbol | Meaning | Behavior |
|--------|---------|----------|
| 🟡 Lazy | Load on access | Triggers query EACH time you access |
| 🔴 Eager | Load upfront | Triggers query ONCE, then cached |

---

## Debug Your Queries

### Enable SQL Logging

```python
# In your code
import logging
_logger = logging.getLogger('odoo.sql_db')
_logger.setLevel(logging.DEBUG)
```

### Check Query Count

```python
from odoo.tools import profiler

# Count queries in a code block
with profiler.Counter() as counter:
    employees = self.env['hr.employee'].search([])
    for emp in employees:
        print(emp.department_id.name)

print(f"Total queries: {counter.count}")
```

### Log Queries in Shell

```python
# Enable in odoo.conf
log_handler = odoo.sql_db:DEBUG

# Or in Python
import logging
logging.getLogger('odoo.sql_db').setLevel(logging.DEBUG)
```

---

## Summary Checklist

- [ ] Use `mapped()` when iterating over related fields
- [ ] Use `search_read()` for API responses
- [ ] Use `read_group()` for counts and sums
- [ ] Avoid queries inside loops
- [ ] Batch write operations on recordsets
- [ ] Check query count during development

---

## Decision Flow: Which Method to Use?

```
                    What do you need?
                          │
          ┌───────────────┼───────────────┐
          │               │               │
      Statistics      Data List      Modify Records
     (count, sum)     for API/UI     (create/write)
          │               │               │
          ▼               ▼               ▼
     read_group()   search_read()     mapped()
                                        │
                                        ▼
                                   recordset.write()
```

### Quick Decision Guide

```
Q: Do you need COUNT, SUM, or AVG by group?
│
├─ YES → read_group()
│
└─ NO ↓

Q: Is this for an API response or frontend?
│
├─ YES → search_read()
│
└─ NO ↓

Q: Do you need to process/modify records?
│
├─ YES → mapped() + iterate
│
└─ NO ↓

Q: Just need a list of values?
│
├─ YES → mapped() returns flat list
│
└─ NO → Use regular search() + mapped() for prefetch
```

### Examples by Task

| Task | Method | Why |
|------|--------|-----|
| Get all employee names | `mapped('name')` | Returns `['John', 'Jane']` |
| Get employees with dept names | `mapped()` + iterate | Need both records and relations |
| API: List employees | `search_read()` | Returns dicts, perfect for JSON |
| Count employees per dept | `read_group()` | Single aggregation query |
| Sum salary per dept | `read_group()` | Single aggregation query |
| Update 1000 records | `recordset.write()` | One UPDATE query |
| Export to CSV | `search_read()` | Returns simple dicts |
