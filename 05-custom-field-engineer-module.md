# Custom Field Engineer Module (Example)

## File: `field_engineer/__manifest__.py`

```python
{
    'name': 'Field Engineer',
    'version': '17.0.1.0.0',
    'summary': 'Manage field engineer job orders and assignments',
    'author': 'Your Company',
    'category': 'Field Service',
    'depends': ['base', 'hr', 'mail'],
    'data': [
        'security/ir.model.access.csv',
        'security/security.xml',
        'data/sequence.xml',
        'views/job_order_view.xml',
        'views/assignment_view.xml',
        'views/menu.xml',
    ],
    'installable': True,
    'application': True,
    # True = appears as main App with icon in home screen.
    'license': 'LGPL-3',
}
```

## File: `field_engineer/models/__init__.py`

```python
from . import job_order
from . import assignment
```

## File: `field_engineer/models/job_order.py`

```python
# Import Odoo's core modules
# - models: base class for all Odoo models
# - fields: field types (Char, Integer, Many2one, etc.)
# - api: decorators (@api.depends, @api.onchange, etc.)
from odoo import models, fields, api

# Import exception classes for error handling
# - ValidationError: for validation failures (shows red error message)
# - UserError: for user-facing errors that stop the action
from odoo.exceptions import ValidationError, UserError


class FieldJobOrder(models.Model):
    """
    Field Job Order Model
    
    This model represents a job order for field engineers.
    It tracks the job from creation through completion.
    """
    
    # ===========================================
    # CLASS ATTRIBUTES
    # ===========================================
    
    # _name: REQUIRED - Technical name of the model
    # This becomes the database table name: field_job_order
    _name = 'field.job.order'
    
    # _description: Human-readable description (shown in debug mode)
    _description = 'Field Job Order'

    # _inherit: Add functionality from other models/classes
    # - mail.thread: Adds chatter (messages, followers, tracking)
    # - mail.activity.mixin: Adds activities/tasks functionality
    _inherit = ['mail.thread', 'mail.activity.mixin']

    # _order: Default sort order when listing records
    # Format: 'field1 asc/desc, field2 asc/desc'
    _order = 'date_scheduled desc, name asc'

    # ===========================================
    # FIELDS - Each maps to a database column
    # ===========================================
    
    # CHAR FIELD: Short text (single line)
    name = fields.Char(
        string='Reference',           # Label shown in UI
        required=True,                # Must have a value
        copy=False,                   # Don't copy when duplicating record
        default='New',                # Default value for new records
        readonly=True,                # Not editable in UI
        # Make editable only in draft state:
        states={'draft': [('readonly', False)]},
    )

    # DATETIME FIELD: Date and time
    date_scheduled = fields.Datetime(
        string='Scheduled Date',
        required=True,
        tracking=True,                # Log changes in chatter (requires mail.thread)
    )

    # Simple char fields
    client_name = fields.Char(string='Client Name')
    client_phone = fields.Char(string='Client Phone')
    location = fields.Char(string='Site Location')
    
    # TEXT FIELD: Multi-line text
    description = fields.Text(string='Job Description')

    # SELECTION FIELD: Dropdown list
    priority = fields.Selection(
        selection=[
            ('0', 'Normal'),         # (stored_value, displayed_text)
            ('1', 'Urgent'),
            ('2', 'Very Urgent'),
        ],
        string='Priority',
        default='0',                 # Default to 'Normal'
    )

    # SELECTION FIELD for state/workflow
    state = fields.Selection(
        selection=[
            ('draft',        'Draft'),
            ('assigned',     'Assigned'),
            ('in_progress',  'In Progress'),
            ('done',         'Done'),
            ('cancelled',    'Cancelled'),
        ],
        string='Status',
        default='draft',
        tracking=True,                # Log state changes in chatter
        required=True,
    )

    # MANY2ONE: Foreign key to another model (like a dropdown)
    engineer_id = fields.Many2one(
        comodel_name='hr.employee',   # Related model
        string='Assigned Engineer',
        tracking=True,
        # domain: filter available options
        # Only show active employees
        domain=[('active', '=', True)],
    )

    # ONETOMANY: List of child records
    # The inverse_name is the field on the child model pointing back
    assignment_ids = fields.One2many(
        comodel_name='field.assignment',
        inverse_name='job_order_id',  # Field in assignment.py
        string='Assignment History',
    )

    # HTML FIELD: Rich text editor
    notes = fields.Html(string='Internal Notes')

    # ===========================================
    # COMPUTED FIELDS - Calculated on demand
    # ===========================================
    
    assignment_count = fields.Integer(
        string='Assignments',
        compute='_compute_assignment_count',  # Method that calculates value
    )

    @api.depends('assignment_ids')
    # Re-compute when assignment_ids changes
    def _compute_assignment_count(self):
        # Always loop over self - it may contain multiple records
        for order in self:
            order.assignment_count = len(order.assignment_ids)

    # ===========================================
    # CRUD OVERRIDES - Customize create/write/unlink
    # ===========================================
    
    @api.model_create_multi
    # Use this decorator (Odoo 16+) for create() method
    def create(self, vals_list):
        """
        Auto-generate reference number when creating new job orders.
        Uses ir.sequence to get the next number.
        """
        # vals_list: list of dictionaries, one per record being created
        for vals in vals_list:
            # Only generate if name is still 'New' (default)
            if vals.get('name', 'New') == 'New':
                # Get next number from sequence
                # 'field.job.order' must match code in sequence.xml
                vals['name'] = (
                    self.env['ir.sequence'].next_by_code('field.job.order')
                    or 'New'  # Fallback if sequence not found
                )
        # Always call super() to complete the database insert
        return super().create(vals_list)

    # ===========================================
    # STATE TRANSITION METHODS - Called by buttons
    # ===========================================
    
    def action_assign(self):
        """Transition from draft to assigned state."""
        self.write({'state': 'assigned'})

    def action_start(self):
        """Transition from assigned to in_progress. Requires engineer."""
        for order in self:
            # Validation: must have an engineer assigned
            if not order.engineer_id:
                raise ValidationError(
                    f"Job Order '{order.name}' has no engineer assigned!"
                )
        self.write({'state': 'in_progress'})

    def action_done(self):
        """Transition to done state."""
        self.write({'state': 'done'})

    def action_cancel(self):
        """Cancel the job order. Cannot cancel completed jobs."""
        # Check if any jobs are already done
        if any(order.state == 'done' for order in self):
            raise UserError("Completed jobs cannot be cancelled.")
        self.write({'state': 'cancelled'})

    def action_reset_draft(self):
        """Reset cancelled job back to draft."""
        self.write({'state': 'draft'})

    # ===========================================
    # SMART BUTTON ACTION - Opens related records
    # ===========================================
    
    def action_view_assignments(self):
        """
        Return an action dictionary to open the assignments view.
        Called when clicking the smart button on the form.
        """
        return {
            'type': 'ir.actions.act_window',  # Window action type
            'name': 'Assignments',            # Window title
            'res_model': 'field.assignment',  # Model to display
            'view_mode': 'tree,form',         # Available views
            'domain': [('job_order_id', '=', self.id)],  # Filter to this job
            'context': {'default_job_order_id': self.id},  # Pre-fill job order
        }
```

## File: `field_engineer/models/assignment.py`

```python
from odoo import models, fields, api
from odoo.exceptions import ValidationError


class FieldAssignment(models.Model):
    """
    Field Assignment Model
    
    Tracks which engineer is assigned to which job order.
    A job order can have multiple assignments (history).
    """
    
    # ===========================================
    # CLASS ATTRIBUTES
    # ===========================================
    _name = 'field.assignment'
    _description = 'Field Engineer Assignment'
    
    # Sort by date_assigned, newest first
    _order = 'date_assigned desc'

    # ===========================================
    # FIELDS
    # ===========================================
    
    # MANY2ONE: This assignment belongs to a job order
    job_order_id = fields.Many2one(
        comodel_name='field.job.order',
        string='Job Order',
        required=True,
        # ondelete='cascade': Delete assignments when job order is deleted
        ondelete='cascade',
    )

    # MANY2ONE: The engineer assigned
    engineer_id = fields.Many2one(
        comodel_name='hr.employee',
        string='Engineer',
        required=True,
        # Only show active employees in dropdown
        domain=[('active', '=', True)],
    )

    # DATETIME: When the assignment was made
    date_assigned = fields.Datetime(
        string='Assigned On',
        default=fields.Datetime.now,  # Auto-fill with current time
        required=True,
    )

    # DATETIME: When the work was completed
    date_completed = fields.Datetime(string='Completed On')

    # SELECTION: Assignment status
    status = fields.Selection(
        selection=[
            ('assigned',  'Assigned'),
            ('on_site',   'On Site'),
            ('completed', 'Completed'),
        ],
        string='Status',
        default='assigned',
    )

    notes = fields.Text(string='Notes')

    # ===========================================
    # COMPUTED FIELD - Duration calculation
    # ===========================================
    
    # FLOAT: Duration in hours (computed from dates)
    duration = fields.Float(
        string='Duration (hours)',
        compute='_compute_duration',
        store=True,  # Save to database (enables filtering/sorting)
    )

    @api.depends('date_assigned', 'date_completed')
    # Re-compute when either date changes
    def _compute_duration(self):
        for rec in self:
            if rec.date_assigned and rec.date_completed:
                # Calculate time difference
                delta = rec.date_completed - rec.date_assigned
                # Convert seconds to hours
                rec.duration = delta.total_seconds() / 3600
            else:
                rec.duration = 0.0

    # ===========================================
    # CONSTRAINTS - Database-level validation
    # ===========================================
    
    @api.constrains('engineer_id', 'job_order_id')
    # Runs validation when these fields change
    def _check_no_duplicate_assignment(self):
        """
        Prevent the same engineer from being assigned twice
        to the same job order.
        """
        for rec in self:
            # Search for duplicate assignments
            # Note: ('id', '!=', rec.id) excludes current record
            # This is important when editing existing records
            duplicate = self.search([
                ('engineer_id',  '=', rec.engineer_id.id),
                ('job_order_id', '=', rec.job_order_id.id),
                ('id', '!=', rec.id),  # Exclude self
            ])
            if duplicate:
                raise ValidationError(
                    f"Engineer '{rec.engineer_id.name}' is already assigned "
                    f"to job order '{rec.job_order_id.name}'."
                )
```

## File: `field_engineer/views/job_order_view.xml`

```xml
<odoo>
    <!-- 
    ===========================================
    FORM VIEW — single record detail page
    ===========================================
    This creates the detailed form when you open a job order record.
    -->
    <record id="view_field_job_order_form" model="ir.ui.view">
        
        <!-- Internal name of the view (for reference/debugging) -->
        <field name="name">field.job.order.form</field>
        
        <!-- Model this view is for (must match _name in Python model) -->
        <field name="model">field.job.order</field>
        
        <!-- XML structure of the view goes inside 'arch' (architecture) -->
        <field name="arch" type="xml">
            
            <!-- 
            <form> = the main container for a single record view
            string="Job Order" = title shown in the view
            -->
            <form string="Job Order">
                
                <!-- 
                ===========================================
                HEADER — action buttons and status bar
                ===========================================
                The header appears at the top of the form and contains:
                - Action buttons (visible based on state)
                - Status bar showing workflow progress
                -->
                <header>
                    
                    <!-- 
                    BUTTON: Assign
                    - name="action_assign" = calls the Python method with this name
                    - type="object" = calls a method on the model (not a server action)
                    - class="btn-primary" = blue highlighted button
                    - states="draft" = only visible when state is 'draft'
                    -->
                    <button name="action_assign" type="object" string="Assign"
                            class="btn-primary" states="draft"/>
                    
                    <!-- 
                    BUTTON: Start Job
                    - No class = default gray button
                    - states="assigned" = only visible when state is 'assigned'
                    -->
                    <button name="action_start" type="object" string="Start Job"
                            states="assigned"/>
                    
                    <button name="action_done" type="object" string="Mark as Done"
                            class="btn-primary" states="in_progress"/>
                    
                    <button name="action_cancel" type="object" string="Cancel"
                            states="draft,assigned,in_progress"/>
                    
                    <button name="action_reset_draft" type="object" string="Reset to Draft"
                            states="cancelled"/>

                    <!-- 
                    STATUS BAR:
                    - widget="statusbar" = renders as clickable workflow steps
                    - statusbar_visible = which states to show as circles
                    -->
                    <field name="state" widget="statusbar"
                           statusbar_visible="draft,assigned,in_progress,done"/>
                </header>

                <!-- 
                ===========================================
                SHEET — main body of the form
                ===========================================
                <sheet> = white card-like container for form content
                This is where most of your fields go.
                -->
                <sheet>
                    
                    <!-- 
                    SMART BUTTON BOX:
                    - class="oe_button_box" = special Odoo class for stat buttons
                    - These appear as clickable boxes at the top right
                    -->
                    <div class="oe_button_box" name="button_box">
                        
                        <!-- 
                        STAT BUTTON:
                        - class="oe_stat_button" = makes it look like a stat box
                        - icon="fa-users" = FontAwesome icon
                        - widget="statinfo" = shows number + label
                        -->
                        <button name="action_view_assignments" type="object"
                                class="oe_stat_button" icon="fa-users">
                            <field name="assignment_count" widget="statinfo" string="Assignments"/>
                        </button>
                    </div>

                    <!-- 
                    TITLE AREA:
                    - class="oe_title" = special styling for main title
                    - Usually contains the record name/reference
                    -->
                    <div class="oe_title">
                        <h1><field name="name" placeholder="Reference..."/></h1>
                    </div>

                    <!-- 
                    ===========================================
                    GROUP — organizes fields in columns
                    ===========================================
                    - <group> = container that arranges fields in 2 columns
                    - Nested <group string="..."> = creates a labeled section
                    -->
                    <group>
                        
                        <!-- First column: Job Information -->
                        <group string="Job Information">
                            <!-- Each <field> = one field from your Python model -->
                            <field name="client_name"/>
                            <field name="client_phone"/>
                            <field name="location"/>
                            <field name="date_scheduled"/>
                            <!-- widget="priority" = renders as star icons -->
                            <field name="priority" widget="priority"/>
                        </group>
                        
                        <!-- Second column: Assignment info -->
                        <group string="Assignment">
                            <!-- 
                            Many2one field options:
                            - options="{'no_create': True}" = hide "Create" button
                            - Prevents creating new employees from dropdown
                            -->
                            <field name="engineer_id" options="{'no_create': True}"/>
                            <!-- readonly="1" = field visible but not editable -->
                            <field name="state" readonly="1"/>
                        </group>
                    </group>

                    <!-- 
                    Single field spanning full width:
                    - nolabel="1" = don't show the field label
                    - placeholder = gray hint text when empty
                    -->
                    <group>
                        <field name="description" placeholder="Describe the job..."
                               nolabel="1"/>
                    </group>

                    <!-- 
                    ===========================================
                    NOTEBOOK — tab container
                    ===========================================
                    <notebook> = creates tabs (like browser tabs)
                    Each <page> = one tab
                    -->
                    <notebook>
                        
                        <!-- 
                        TAB 1: Assignment History
                        - name="assignments_tab" = internal identifier
                        -->
                        <page string="Assignment History" name="assignments_tab">
                            
                            <!-- 
                            One2many field with inline tree view:
                            - Shows related records in a table
                            - editable="bottom" = add new rows at bottom
                            -->
                            <field name="assignment_ids">
                                <tree editable="bottom">
                                    <field name="engineer_id"/>
                                    <field name="date_assigned"/>
                                    <field name="status"/>
                                    <field name="notes"/>
                                </tree>
                            </field>
                        </page>
                        
                        <!-- TAB 2: Internal Notes -->
                        <page string="Internal Notes" name="notes_tab">
                            <field name="notes" nolabel="1"/>
                        </page>
                    </notebook>
                </sheet>

                <!-- 
                ===========================================
                CHATTER — messaging and activity section
                ===========================================
                - Appears at the bottom of the form
                - Requires mail.thread and mail.activity.mixin in Python
                -->
                <div class="oe_chatter">
                    <!-- Followers list -->
                    <field name="message_follower_ids"/>
                    <!-- Activities/tasks -->
                    <field name="activity_ids"/>
                    <!-- Message history/log -->
                    <field name="message_ids"/>
                </div>
            </form>
        </field>
  </record>

    <!-- 
    ===========================================
    TREE / LIST VIEW
    ===========================================
    Shows multiple records in a table format (the list view)
    -->
    <record id="view_field_job_order_tree" model="ir.ui.view">
        
        <field name="name">field.job.order.tree</field>
        <field name="model">field.job.order</field>
        <field name="arch" type="xml">
            
            <!-- 
            <tree> = list/table view
            decoration-* = conditional row colors based on field values:
            - decoration-success = green (state is done)
            - decoration-info = blue (in progress)
            - decoration-warning = orange/yellow (urgent priority)
            - decoration-muted = gray (cancelled)
            -->
            <tree string="Job Orders"
                  decoration-success="state == 'done'"
                  decoration-info="state == 'in_progress'"
                  decoration-warning="priority == '2'"
                  decoration-muted="state == 'cancelled'">
                
                <!-- Columns in the list -->
                <field name="name" string="Reference"/>
                <field name="client_name"/>
                <field name="engineer_id"/>
                <field name="date_scheduled"/>
                <!-- optional="show" = shown by default, user can hide -->
                <field name="priority" widget="priority" optional="show"/>
                
                <!-- 
                Badge widget for status:
                - widget="badge" = colored label
                - decoration-* = badge color based on state
                -->
                <field name="state" widget="badge"
                       decoration-success="state == 'done'"
                       decoration-info="state == 'in_progress'"
                       decoration-warning="state == 'assigned'"
                       decoration-muted="state == 'cancelled'"/>
                
                <!-- 
                invisible="1" = hidden but needed for decorations
                (must be in tree to use in decoration-warning above)
                -->
                <field name="priority" invisible="1"/>
            </tree>
        </field>
  </record>

    <!-- 
    ===========================================
    SEARCH VIEW
    ===========================================
    Defines search options in the list view:
    - Searchable fields
    - Filter buttons
    - Group by options
    -->
    <record id="view_field_job_order_search" model="ir.ui.view">
        
        <field name="name">field.job.order.search</field>
        <field name="model">field.job.order</field>
        <field name="arch" type="xml">
            
            <search string="Search Job Orders">
                
                <!-- 
                SEARCHABLE FIELDS:
                These fields appear in the search bar dropdown
                -->
                <field name="name"/>
                <field name="client_name"/>
                <field name="engineer_id"/>

                <!-- 
                ===========================================
                FILTERS
                ===========================================
                Predefined search buttons
                - domain = Odoo ORM filter expression
                - uid = current logged-in user ID
                -->
                <!-- Show only jobs assigned to current user -->
                <filter string="My Jobs" name="my_jobs"
                        domain="[('engineer_id.user_id', '=', uid)]"/>
                
                <filter string="In Progress" name="in_progress"
                        domain="[('state', '=', 'in_progress')]"/>
                
                <filter string="Done" name="done"
                        domain="[('state', '=', 'done')]"/>
                
                <!-- Priority in list = OR condition -->
                <filter string="Urgent" name="urgent"
                        domain="[('priority', 'in', ['1', '2'])]"/>
                
                <!-- Visual separator between filter groups -->
                <separator/>
                
                <!-- Date-based filter (today's jobs) -->
                <filter string="Today's Jobs" name="today"
                        domain="[('date_scheduled', '&gt;=', datetime.datetime.now().strftime('%Y-%m-%d 00:00:00')),
                                 ('date_scheduled', '&lt;=', datetime.datetime.now().strftime('%Y-%m-%d 23:59:59'))]"/>

                <!-- 
                ===========================================
                GROUP BY
                ===========================================
                Allows grouping records in the list view
                - expand="0" = collapsed by default
                - context={'group_by': 'field_name'} = which field to group by
                -->
                <group expand="0" string="Group By">
                    <filter name="group_by_engineer" string="Engineer"
                            context="{'group_by': 'engineer_id'}"/>
                    <filter name="group_by_state" string="Status"
                            context="{'group_by': 'state'}"/>
                    <!-- :week = group by week instead of exact date -->
                    <filter name="group_by_date" string="Scheduled Date"
                            context="{'group_by': 'date_scheduled:week'}"/>
                </group>
            </search>
        </field>
  </record>

    <!-- 
    ===========================================
    WINDOW ACTION
    ===========================================
    Defines what happens when user clicks a menu item
    - Which model to show
    - Which views are available
    - Default filters/domain
    -->
    <record id="action_field_job_order" model="ir.actions.act_window">
        
        <!-- Title shown in the view header -->
        <field name="name">Job Orders</field>
        
        <!-- Model this action opens -->
        <field name="res_model">field.job.order</field>
        
        <!-- Available view modes (order matters - first is default) -->
        <field name="view_mode">tree,form,kanban</field>
        
        <!-- Default domain filter (empty = show all) -->
        <field name="domain">[]</field>
        
        <!-- Context passed to views (can set default values) -->
        <field name="context">{}</field>
        
        <!-- 
        Help message shown when list is empty
        - type="html" = allows HTML formatting
        -->
        <field name="help" type="html">
            <p class="o_view_nocontent_smiling_face">Create your first job order!</p>
            <p>Track field engineering jobs, assign engineers, and monitor progress.</p>
        </field>
  </record>
</odoo>
```

## File: `field_engineer/views/menu.xml`

```xml
<odoo>
    <!-- 
    ===========================================
    ROOT MENU — top-level menu in navigation bar
    ===========================================
    This creates the main "Field Engineer" menu entry
    that appears in Odoo's top navigation.
    -->
    
    <!-- 
    ROOT MENUITEM attributes:
    - id = unique identifier (used as parent for sub-menus)
    - name = text shown in the menu
    - sequence = order in menu bar (lower = more to the left)
    - web_icon = icon shown in app drawer (module_name,path/to/icon.png)
    -->
    <menuitem id="menu_field_engineer_root"
              name="Field Engineer"
              sequence="50"
              web_icon="field_engineer,static/description/icon.png"/>

    <!-- 
    ===========================================
    SECOND-LEVEL MENUS — submenus under root
    ===========================================
    These appear as dropdown items when clicking the root menu.
    -->
    
    <!-- 
    JOB ORDERS SUBMENU:
    - parent = which menu this belongs under (the root menu's id)
    - action = which window action to open when clicked
    - sequence = order within the dropdown (10 = first)
    -->
    <menuitem id="menu_field_engineer_job_orders"
              name="Job Orders"
              parent="menu_field_engineer_root"
              action="action_field_job_order"
              sequence="10"/>

    <!-- 
    ASSIGNMENTS SUBMENU:
    - Opens a different window action for assignments
    - sequence="20" = appears after Job Orders
    -->
    <menuitem id="menu_field_engineer_assignments"
              name="Assignments"
              parent="menu_field_engineer_root"
              action="action_field_assignment"
              sequence="20"/>

    <!-- 
    CONFIGURATION SUBMENU:
    - groups = restrict to specific user groups
    - base.group_system = only administrators can see this
    - sequence="100" = appears at the bottom (config usually last)
    -->
    <menuitem id="menu_field_engineer_config"
              name="Configuration"
              parent="menu_field_engineer_root"
              sequence="100"
              groups="base.group_system"/>
</odoo>
```

## File: `field_engineer/data/sequence.xml`

```xml
<odoo>
    <!-- 
    ===========================================
    SEQUENCE — auto-incrementing reference numbers
    ===========================================
    Sequences generate unique reference numbers like FE/2024/0001
    
    <data noupdate="1">:
    - noupdate="1" = load once on install, don't overwrite on module upgrade
    - Prevents resetting the sequence counter when you update the module
    - IMPORTANT: keeps your numbering continuous across upgrades
    -->
    <data noupdate="1">
        
        <!-- 
        SEQUENCE RECORD:
        model="ir.sequence" = Odoo's built-in sequence model
        -->
        <record id="seq_field_job_order" model="ir.sequence">
            
            <!-- Human-readable name (shown in Settings > Technical > Sequences) -->
            <field name="name">Field Job Order Sequence</field>
            
            <!-- 
            CODE: MUST match what you use in Python
            In job_order.py: self.env['ir.sequence'].next_by_code('field.job.order')
            This code is how Python finds this sequence definition.
            -->
            <field name="code">field.job.order</field>
            
            <!-- 
            PREFIX: Text that appears before the number
            - %(year)s = 4-digit year (2024)
            - Result: FE/2024/0001, FE/2024/0002, etc.
            Other placeholders: %(month)s, %(day)s, %(y)s (2-digit year)
            -->
            <field name="prefix">FE/%(year)s/</field>
            
            <!-- 
            PADDING: Number of digits for the counter
            padding="4" = 0001, 0002, ... 9999
            padding="5" = 00001, 00002, etc.
            -->
            <field name="padding">4</field>
            
            <!-- How much to increment each time (usually 1) -->
            <field name="number_increment">1</field>
            
            <!-- Starting number for new installations -->
            <field name="number_next_actual">1</field>
        </record>
    </data>
</odoo>
```

## File: `field_engineer/security/security.xml`

```xml
<odoo>
    <!-- 
    ===========================================
    SECURITY GROUPS
    ===========================================
    Groups define user roles. Users are assigned to groups,
    and permissions are granted to groups (not individual users).
    -->
    
    <!-- 
    USER GROUP:
    - Basic users who work with job orders
    - Can read/write but limited create/delete
    -->
    <record id="group_field_engineer_user" model="res.groups">
        
        <!-- Display name shown in user settings -->
        <field name="name">Field Engineer / User</field>
        
        <!-- 
        category_id: where this group appears in Settings > Users > Groups
        - base.module_category_hidden = not shown in the main list
        - You can create custom categories if needed
        -->
        <field name="category_id" ref="base.module_category_hidden"/>
        
        <!-- 
        implied_ids: groups that are automatically granted
        - eval="[(4, ref('base.group_user'))]" = add base.group_user
        - (4, ID) = Odoo's "add to list" command
        - Effect: User group members also get basic Odoo user rights
        -->
        <field name="implied_ids" eval="[(4, ref('base.group_user'))]"/>
    </record>

    <!-- 
    MANAGER GROUP:
    - Administrators with full access
    - Inherits all User permissions + additional rights
    -->
    <record id="group_field_engineer_manager" model="res.groups">
        <field name="name">Field Engineer / Manager</field>
        <field name="category_id" ref="base.module_category_hidden"/>
        
        <!-- 
        Managers inherit User group:
        - Managers get all User permissions automatically
        - Plus any additional manager-only permissions
        -->
        <field name="implied_ids" eval="[(4, ref('group_field_engineer_user'))]"/>
    </record>

    <!-- 
    ===========================================
    RECORD RULES — row-level access control
    ===========================================
    Record rules filter what records a user can see/modify.
    They add a WHERE clause to every database query.
    -->
    
    <!-- 
    USER RULE: Own jobs only
    Regular users can only see:
    - Jobs assigned to them
    - Jobs with no engineer assigned (unassigned)
    -->
    <record id="rule_job_order_user_own" model="ir.rule">
        
        <!-- Descriptive name for debugging -->
        <field name="name">Field Job Order: own jobs only (user)</field>
        
        <!-- 
        model_id: which model this rule applies to
        - ref="model_field_job_order" = auto-generated when model is created
        - Format: model_<model_name_with_underscores>
        -->
        <field name="model_id" ref="model_field_job_order"/>
        
        <!-- 
        domain_force: the filter expression
        - '|' = OR operator (prefix notation)
        - Condition 1: engineer's linked user = current user
        - Condition 2: no engineer assigned (False/None)
        
        Odoo domain syntax:
        - ['|', A, B] = A OR B
        - ['&', A, B] = A AND B (default, can omit)
        - ['!', A] = NOT A
        -->
        <field name="domain_force">[
            '|',
            ('engineer_id.user_id', '=', user.id),
            ('engineer_id', '=', False)
        ]</field>
        
        <!-- Which group this rule applies to -->
        <field name="groups" eval="[(4, ref('group_field_engineer_user'))]"/>
        
        <!-- 
        Permission types this rule affects:
        - perm_read = can read matching records
        - perm_write = can edit matching records
        - perm_create = can create new records (False = no restriction)
        - perm_unlink = can delete matching records
        -->
        <field name="perm_read"   eval="True"/>
        <field name="perm_write"  eval="True"/>
        <field name="perm_create" eval="False"/>
        <field name="perm_unlink" eval="True"/>
    </record>

    <!-- 
    MANAGER RULE: See all records
    Managers have no restrictions - they see everything.
    -->
    <record id="rule_job_order_manager_all" model="ir.rule">
        <field name="name">Field Job Order: see all (manager)</field>
        <field name="model_id" ref="model_field_job_order"/>
        
        <!-- 
        domain_force="[(1, '=', 1)]" = always TRUE
        This effectively disables filtering for managers
        -->
        <field name="domain_force">[(1, '=', 1)]</field>
        
        <field name="groups" eval="[(4, ref('group_field_engineer_manager'))]"/>
        
        <!-- Managers get all permissions -->
        <field name="perm_read"   eval="True"/>
        <field name="perm_write"  eval="True"/>
        <field name="perm_create" eval="True"/>
        <field name="perm_unlink" eval="True"/>
    </record>
</odoo>
```

## File: `field_engineer/security/ir.model.access.csv`

```csv
# ===========================================
# MODEL ACCESS RIGHTS (CSV FORMAT)
# ===========================================
# This file defines WHO can do WHAT to WHICH model.
# It's like a permissions table: [Group] can [Action] on [Model]
#
# COLUMNS:
# - id: Unique identifier for this access rule
# - name: Human-readable description
# - model_id:id: The model (format: model_<model_name>)
# - group_id:id: The user group (module_name.group_id)
# - perm_read: Can view records? (1=yes, 0=no)
# - perm_write: Can edit records? (1=yes, 0=no)
# - perm_create: Can create new records? (1=yes, 0=no)
# - perm_unlink: Can delete records? (1=yes, 0=no)
#
# NOTE: Record rules (security.xml) further filter which records
# a user can access. This CSV just defines basic model-level access.
# ===========================================

id,name,model_id:id,group_id:id,perm_read,perm_write,perm_create,perm_unlink

# Job Order access rules
# User: read/write/create, but NO delete
access_job_order_user,field.job.order user,model_field_job_order,field_engineer.group_field_engineer_user,1,1,1,0

# Manager: full access (all CRUD operations)
access_job_order_manager,field.job.order manager,model_field_job_order,field_engineer.group_field_engineer_manager,1,1,1,1

# Public (any logged-in user): read-only
access_job_order_public,field.job.order public,model_field_job_order,base.group_user,1,0,0,0

# Assignment access rules
# User: read/write/create, but NO delete
access_assignment_user,field.assignment user,model_field_assignment,field_engineer.group_field_engineer_user,1,1,1,0

# Manager: full access
access_assignment_manager,field.assignment manager,model_field_assignment,field_engineer.group_field_engineer_manager,1,1,1,1
```
