# Create API Endpoints in Odoo

Odoo provides HTTP controllers to create REST/JSON-RPC APIs for external integrations.

## API Types

| Type | Route Attribute | Use Case |
|------|-----------------|----------|
| `jsonrpc` | `type='jsonrpc'` | JSON-RPC protocol (recommended) |
| `http` | `type='http'` | REST-style HTTP endpoints |

---

## Step-by-Step: Create Employee API

### Step 1: Create Controller File

Scaffold (Quick Start)
```bash
python odoo-bin scaffold employees_api custom_addons
```

### Go To
`controllers/main.py` # could also be `api.py`

Add endpoint and a method that will be executed if that endpoint is hit

```python
from odoo import http
from odoo.http import request

class EmployeeAPI(http.Controller):
    
    # This is the endpoint
    @http.route('/api/employees', type='jsonrpc', auth='user', methods=['POST']) 
    def list_employees(self, **kwargs): #this is the method
        """List all employees"""
        employees = request.env['hr.employee'].sudo().search([])

        data = [
            {
                'id': emp.id,
                'name': emp.name,
                'work_email': emp.work_email,
            }
            for emp in employees
        ]

        return {'employees': data}
```

### Step 2: Update Controller __init__.py

`controllers/__init__.py`

```python
from . import main # if we had created api.py we would import api
```

### Step 3: Update Module __init__.py

`__init__.py`

```python
from . import models
from . import controllers
# These should be there by default if used scaffold
```

### Step 4: Update Manifest

`__manifest__.py`

```python
{
    'name': 'HR Extension',
    'version': '1.0',
    'depends': ['hr'], # this api depends on hr module
    'data': [],
}
```

### Step 5: Restart Odoo & Update Module

```bash
python odoo-bin -c odoo.conf 
```

---

## Complete CRUD Example (inside Controller/main.py)

### Note: Do Not Use ```sudo()``` for production unless required
### List Employees 

```python
@http.route('/api/employees', type='jsonrpc', auth='user', methods=['POST'])
def list_employees(self, **kwargs):
    employees = request.env['hr.employee'].sudo().search([])
    return {
        'employees': [{'id': e.id, 'name': e.name} for e in employees]
    }
```

### Get Single Employee

```python
@http.route('/api/employees/<int:employee_id>', type='jsonrpc', auth='user', methods=['POST'])
def get_employee(self, employee_id, **kwargs):
    employee = request.env['hr.employee'].sudo().browse(employee_id)
    if not employee.exists():
        return {'error': 'Not found'}

    return {
        'id': employee.id,
        'name': employee.name,
        'work_email': employee.work_email,
    }
```

### Create Employee

```python
@http.route('/api/employees/create', type='jsonrpc', auth='user', methods=['POST'])
def create_employee(self, **kwargs):
    # Required field
    if not kwargs.get('name'):
        return {'error': 'Name is required'}
    
    # mandatory field
    values = {
        'name': kwargs['name'],
        'company_id': request.env.company.id,
    }

    # Optional fields
    if kwargs.get('work_email'):
        values['work_email'] = kwargs['work_email']

    try:
        employee = request.env['hr.employee'].sudo().create(values)
        return {
            'success': True,
            'id': employee.id,
            'name': employee.name,
        }
    except Exception as e:
        return {'error': str(e)}
```

### Update Employee

```python
@http.route('/api/employees/<int:employee_id>/update', type='jsonrpc', auth='user', methods=['POST'])
def update_employee(self, employee_id, **kwargs):
    employee = request.env['hr.employee'].sudo().browse(employee_id)

    if not employee.exists():
        return {'error': 'Employee not found'}

    # Optional fields, will update if value passed
    values = {}
    if kwargs.get('name'):
        values['name'] = kwargs['name']
    if kwargs.get('work_email'):
        values['work_email'] = kwargs['work_email']

    try:
        if values:
            employee.write(values)
        return {'success': True, 'id': employee.id}
    except Exception as e:
        return {'error': str(e)}
```

### Delete Employee

```python
@http.route('/api/employees/<int:employee_id>/delete', type='jsonrpc', auth='user', methods=['POST'])
def delete_employee(self, employee_id, **kwargs):
    employee = request.env['hr.employee'].sudo().browse(employee_id)

    if not employee.exists():
        return {'error': 'Employee not found'}

    try:
        employee.unlink()
        return {'success': True, 'id': employee_id}
    except Exception as e:
        return {'error': str(e)}
```

---

## Directory Structure

```
custom_addons/hr_extension/
├── __init__.py
├── __manifest__.py
|
├── controllers/
│   ├── __init__.py
│   └── main.py
|
└── models/
    ├── __init__.py
    └── models.py
```
This is enough for API. No need of views.

---

## Route Parameters

| Parameter | Values | Description |
|-----------|--------|-------------|
| `type` | `'jsonrpc'`, `'http'` | API protocol type |
| `auth` | `'user'`, `'public'`, `'none'` | Authentication method |
| `methods` | `['GET']`, `['POST']`, `['GET', 'POST']` | HTTP methods allowed |
| `csrf` | `True`, `False` | CSRF protection (use False for APIs) |

### Auth Options

| Value | Description |
|-------|-------------|
| `'user'` | Requires authenticated Odoo user login|
| `'public'` | Anyone can access (anyone can use `.sudo()` in code) |
| `'none'` | No authentication |

---

## Testing with Postman

### URL Format
```
POST http://localhost:8069/api/employees
```

### Headers
```
Content-Type: application/json
```

### Body (JSON-RPC)
```json
{
    "jsonrpc": "2.0",
    "method": "call",
    "params": {
        "db": "your_database",
        "login": "admin",
        "password": "admin",
        "name": "John Doe"
    },
    "id": 1
}
```

### Response
```json
{
    "jsonrpc": "2.0",
    "id": 1,
    "result": {
        "success": true,
        "id": 5,
        "name": "John Doe"
    }
}
```

---

## Key Concepts

### request.env
Access Odoo environment and models:
```python
employees = request.env['hr.employee'].sudo().search([])
```

### .sudo()
Run as superuser (bypasses security rules):
```python
# With sudo() - bypasses record rules
request.env['hr.employee'].sudo().create(values)

# Without sudo() - respects user permissions
request.env['hr.employee'].create(values)
```

### browse() vs search()
```python
# browse - get by ID
employee = request.env['hr.employee'].browse(1)

# search - get by domain filter
employees = request.env['hr.employee'].search([('active', '=', True)])
```

### URL Parameters
```python
# Dynamic URL parameter
@http.route('/api/employees/<int:employee_id>', ...)
def get_employee(self, employee_id, **kwargs):
    # employee_id comes from URL

# String parameter
@http.route('/api/employees/<name>', ...)
def get_by_name(self, name, **kwargs):
    # name comes from URL
```

---

## Common Patterns

### Check Record Exists
```python
employee = request.env['hr.employee'].sudo().browse(employee_id)
if not employee.exists():
    return {'error': 'Not found'}
```

### Handle Required Fields
```python
if not kwargs.get('name'):
    return {'error': 'Name is required', 'code': 400}
```

### Error Handling
```python
try:
    employee.unlink()
    return {'success': True}
except Exception as e:
    return {'error': str(e)}
```

---

## Quick Reference

| Operation | Method | URL Pattern |
|-----------|--------|-------------|
| List | `search([])` | `/api/employees` |
| Get | `browse(id)` | `/api/employees/<id>` |
| Create | `create(values)` | `/api/employees/create` |
| Update | `write(values)` | `/api/employees/<id>/update` |
| Delete | `unlink()` | `/api/employees/<id>/delete` |

| ORM Method | Description |
|------------|-------------|
| `search(domain)` | Find records matching domain |
| `browse(id)` | Get record by ID |
| `create(values)` | Create new record |
| `write(values)` | Update record |
| `unlink()` | Delete record |
| `exists()` | Check if record exists |
| `search_count(domain)` | Count matching records |
