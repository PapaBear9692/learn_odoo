# Modifying the HR Module

> **Golden Rule**: Never edit core Odoo source files. Always create a **new module** that inherits the existing one.

## Step 1 — Create `hr_custom/__manifest__.py`

```python
{
    'name': 'HR Customization',
    'version': '17.0.1.0.0',
    'depends': ['hr'],
    # MUST list 'hr' as a dependency.
    'data': [
        'security/ir.model.access.csv',
        'views/hr_employee_view.xml',
    ],
    'installable': True,
}
```

## Step 2 — Extend the Employee Model

```python
# hr_custom/models/hr_employee.py
from odoo import models, fields, api


class HrEmployee(models.Model):
    """
    HR Employee Extension

    This class extends the built-in hr.employee model
    to add custom fields and custom functionality.
    """

    # ===========================================
    # CLASS ATTRIBUTES
    # ===========================================

    # _inherit: KEY - inherit (extend) existing model instead of creating new one.
    # Fields are added to the same PostgreSQL table (hr_employee)
    _inherit = 'hr.employee'

    # ===========================================
    # FIELDS - Each maps to a database column
    # ===========================================

    # NEW CUSTOM FIELDS
    employee_code = fields.Char(
        string='Employee Code',           # Label shown in UI
        copy=False,                   # Don't copy when duplicating record
    )

    date_confirmed = fields.Date(string='Confirmation Date')

    is_remote = fields.Boolean(
        string='Remote Worker',
        default=False,
    )

    # SELECTION FIELD: Dropdown list
    skill_level = fields.Selection(
        selection=[
            ('junior', 'Junior'),         # (stored_value, displayed_text)
            ('mid',    'Mid-Level'),
            ('senior', 'Senior'),
        ],
        string='Skill Level',
    )

    years_experience = fields.Integer(
        string='Years of Experience',
        default=0,
    )

    # ===========================================
    # COMPUTED FIELDS - Calculated on demand
    # ===========================================

    seniority_label = fields.Char(
        string='Seniority',
        compute='_compute_seniority_label',  # Method that calculates value
    )

    @api.depends('skill_level', 'years_experience')
    # Re-compute when skill_level OR years_experience changes
    def _compute_seniority_label(self):
        # Always loop over self - may contain multiple records
        for emp in self:
            # Get the display text for the current selection value
            label = dict(self._fields['skill_level'].selection).get(
                emp.skill_level, ''
            )
            # Build the seniority label string
            emp.seniority_label = f"{label} ({emp.years_experience} yrs)"

    # ===========================================
    # METHOD OVERRIDES - Customize existing behavior
    # ===========================================

    def _get_employee_name(self):
        """
        Override the employee name display to include employee code.
        """
        result = super()._get_employee_name()
        if self.employee_code:
            return f"[{self.employee_code}] {result}"
        return result
```

## Step 3 — Extend the Form View (XML)

```xml
<!-- hr_custom/views/hr_employee_view.xml -->
<odoo>
    <!--
    ===========================================
    INHERITED FORM VIEW - extends existing employee form
    ===========================================
    This record modifies the standard HR Employee form view
    to add our custom fields and tab.
    -->
    <record id="view_hr_employee_custom_form" model="ir.ui.view">

        <!-- Internal name of the view (for reference/debugging) -->
        <field name="name">hr.employee.form.custom</field>

        <!-- Model this view is for (must match _name in Python model) -->
        <field name="model">hr.employee</field>

        <!--
        VERY IMPORTANT: we are inheriting the existing employee form
        - ref="hr.view_employee_form" points to the core hr module's form view
        - This is how Odoo knows to modify instead of replace
        -->
        <field name="inherit_id" ref="hr.view_employee_form"/>

        <!-- XML structure modifications go inside 'arch' (architecture) -->
        <field name="arch" type="xml">

            <!--
            ===========================================
            XPATH INJECTION 1: Add fields after 'department_id'
            ===========================================
            xpath = where to insert your changes
            //field[@name='department_id'] = find the field named 'department_id'
            position="after" = insert content AFTER that element

            Available positions:
            - "before"  = insert before the found element
            - "after"   = insert after the found element
            - "inside"  = insert inside the found element
            - "replace" = replace the found element entirely
            - "attributes" = modify attributes of the found element
            -->
            <xpath expr="//field[@name='department_id']" position="after">

                <!-- Your custom fields from Python model -->
                <field name="employee_code"/>
                <field name="skill_level"/>
                <field name="years_experience"/>
                <field name="is_remote"/>

            </xpath>

            <!--
            ===========================================
            XPATH INJECTION 2: Add a new tab (notebook page)
            ===========================================
            //notebook = find the notebook (tabs) inside the form
            position="inside" = insert content inside that notebook
            -->
            <xpath expr="//notebook" position="inside">

                <!-- Create a new tab/page -->
                <page string="Custom Info" name="custom_info_page">

                    <!--
                    Group = layout (fields arranged nicely in columns)
                    Nested group creates a labeled section.
                    -->
                    <group>

                        <!-- Labeled group for confirmation fields -->
                        <group string="Confirmation">
                            <field name="date_confirmed"/>
                            <field name="seniority_label"/>
                        </group>

                    </group>
                </page>

            </xpath>

            <!--
            ===========================================
            XPATH INJECTION 3: Modify attribute of existing field
            ===========================================
            Change the label of the 'name' field from "Name" to "Full Name"
            -->
            <xpath expr="//field[@name='name']" position="attributes">
                <attribute name="string">Full Name</attribute>
            </xpath>

        </field>
    </record>
</odoo>
```
