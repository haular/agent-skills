# Odoo 20 View Guide

## Coding Conventions

- Use model-first IDs such as `my_model_view_list` and matching dotted names
  such as `my.model.list`; name inherited views after the model and feature.
- Use `<list>` and `list,form` in normal Odoo 20 examples. Keep field order
  task-oriented and use direct modifier expressions rather than `attrs`.
- Icons are Material Symbols: `<i class="oi" data-icon="NAME"/>`, button
  `icon="NAME"`. Font Awesome (`fa fa-*`) is no longer loaded (since 19.5).

Guide for creating views in Odoo 20: list, form, search, kanban/card, calendar, graph, pivot, icons, and QWeb templates.

## Table of Contents

- [View Types](#view-types)
- [Common Root Attributes](#common-root-attributes)
- [List Views](#list-views)
- [Form Views](#form-views)
- [Icons](#icons)
- [Search Views](#search-views)
- [Kanban and Card Views](#kanban-and-card-views)
- [Calendar Views](#calendar-views)
- [Graph Views](#graph-views)
- [Pivot Views](#pivot-views)
- [View Inheritance](#view-inheritance)
- [QWeb Templates](#qweb-templates)

---

## View Types

| Type     | Tag          | Description                                         |
| -------- | ------------ | --------------------------------------------------- |
| List     | `<list>`     | Table view (formerly `<tree>`)                      |
| Form     | `<form>`     | Single record view                                  |
| Search   | `<search>`   | Search/filter view                                  |
| Kanban   | `<kanban>`   | Card-based view                                     |
| Card     | `<card>`     | Reusable card templates (new `ir.ui.view` type `card`) |
| Calendar | `<calendar>` | Calendar view                                       |
| Graph    | `<graph>`    | Chart view                                          |
| Pivot    | `<pivot>`    | Pivot table view                                    |
| Activity | `<activity>` | Activity view                                       |
| QWeb     | `<template>` | HTML/PDF template                                   |

Arch is validated on install by the RelaxNG schemas in `odoo/addons/base/rng/*.rng` (list,
search, calendar, graph, pivot, activity + `common.rng`) and by `_validate_tag_*` methods in
`odoo/addons/base/models/ir_ui_view.py`. Unknown attributes fail the RNG.

---

## Common Root Attributes

### dialog_size (new in 20)

Size of the dialog when the view is opened in one (action `target="new"`, many2one
"Search more"/"Create and edit", x2many form dialog). No effect otherwise.
Values: `sm`, `md`, `lg`, `xl`, `fs`, `fullscreen`.

```xml
<form dialog_size="lg">
    ...
</form>
<list dialog_size="xl">
    ...
</list>
```

---

## List Views

### Basic List View

```xml
<record id="my_model_view_list" model="ir.ui.view">
    <field name="name">my.model.list</field>
    <field name="model">my.model</field>
    <field name="arch" type="xml">
        <list>
            <field name="name"/>
            <field name="code"/>
            <field name="date"/>
        </list>
    </field>
</record>
```

### List View Options

```xml
<list editable="bottom" multi_edit="1" limit="80" default_order="date desc">
    <field name="name"/>
    <field name="code"/>
</list>
```

| Option          | Description                                        |
| --------------- | -------------------------------------------------- |
| `editable`      | `top` or `bottom`: inline editing                  |
| `multi_edit`    | Enable multi-row editing                           |
| `limit`         | Default records per page                           |
| `create`        | Show create button (default: true)                 |
| `default_order` | Default sort                                       |
| `dialog_size`   | Dialog size when opened in a dialog (new in 20)    |
| `string`        | Deprecated on `<list>`: has no effect (RNG comment) |

### Decoration

```xml
<list>
    <field name="state" decoration-success="state == 'done'"
                       decoration-warning="state == 'pending'"
                       decoration-danger="state == 'cancelled'"/>
    <field name="name"/>
</list>
```

| Decoration           | Description   |
| -------------------- | ------------- |
| `decoration-success` | Green         |
| `decoration-warning` | Yellow/Orange |
| `decoration-danger`  | Red           |
| `decoration-info`    | Blue          |
| `decoration-primary` | Primary color |
| `decoration-muted`   | Grey          |
| `decoration-bf`      | Bold font     |
| `decoration-it`      | Italic font   |

### Optional and Hidden Columns

```xml
<list>
    <field name="name"/>
    <field name="description" optional="show"/>
    <field name="notes" optional="hide"/>
    <field name="company_id" column_invisible="not context.get('show_company')"/>
</list>
```

`column_invisible` hides the whole column (evaluated without the row values); `invisible`
hides the cell per row. In 20 `column_invisible`/`invisible` are also accepted on `<widget>`.

### Stacked Column (new in 20)

`<column>` groups several fields in one cell, on stacked lines. Attributes: `name`, `string`
(defaults to the first field's label), `column_invisible`, `width`. `column_invisible` has no
effect on the child fields; put it on `<column>`.

```xml
<list>
    <field name="name"/>
    <column string="Contact">
        <field name="partner_id"/>
        <field name="email"/>
    </column>
</list>
```

### Column Width

`width` accepts a fixed width (`width="40px"`, or `width="40"`: always pixels), a minimum
(`width="[40]"`) or a min/max range (`width="[40,70]"`).

---

## Form Views

### Basic Form View

```xml
<record id="view_my_model_form" model="ir.ui.view">
    <field name="name">my.model.form</field>
    <field name="model">my.model</field>
    <field name="arch" type="xml">
        <form string="My Model">
            <sheet>
                <group>
                    <group>
                        <field name="name"/>
                        <field name="code"/>
                    </group>
                    <group>
                        <field name="active"/>
                        <field name="date"/>
                    </group>
                </group>
                <notebook>
                    <page string="Description">
                        <field name="description"/>
                    </page>
                    <page string="Notes">
                        <field name="notes"/>
                    </page>
                </notebook>
            </sheet>
        </form>
    </field>
</record>
```

### Form Attributes

```xml
<form string="My Model" create="false" edit="false" delete="false" dialog_size="lg">
    ...
</form>
```

| Attribute     | Description                          |
| ------------- | ------------------------------------ |
| `create`      | Show create button                   |
| `edit`        | Show edit button                     |
| `delete`      | Show delete button                   |
| `dialog_size` | Dialog size (new in 20)              |

### Chatter

`<chatter/>` must be a direct sibling of `<sheet>`, not a wrapper around it:

```xml
<form string="My Model">
    <sheet>
        <!-- Form content -->
    </sheet>
    <chatter/>
</form>
```

### Statusbar and Buttons

```xml
<form string="My Model">
    <header>
        <button name="action_confirm" string="Confirm" type="object" invisible="state != 'draft'"
                confirm="Confirm this record?" confirm-title="Confirmation"/>
        <button name="action_send" string="Send" type="object" icon="mail" icon_class="oi-filled"/>
        <field name="state" widget="statusbar"/>
    </header>
    <sheet>
        <!-- Form content -->
    </sheet>
</form>
```

| Button attribute | Description                                                      |
| ---------------- | ---------------------------------------------------------------- |
| `icon`           | Material Symbols name (e.g. `mail`, `check`, `settings`)         |
| `icon_class`     | Extra classes on the icon, e.g. `oi-filled`, `text-danger` (new) |
| `confirm`        | Confirmation dialog message                                      |
| `confirm-title`  | Title of the confirmation dialog (new in 20 RNG)                 |

---

## Icons

Since 19.5, Odoo uses **Material Symbols** (Outlined) plus Odoo UI icons, both via the `.oi`
class and a `data-icon` attribute. Font Awesome CSS is no longer in the asset bundles, so
`fa fa-*` classes render nothing.

```xml
<!-- ❌ Odoo ≤ 19.0 -->
<i class="fa fa-user" title="User"/>
<button name="action_send" type="object" icon="fa-envelope"/>

<!-- ✅ Odoo 20 -->
<i class="oi" data-icon="person" title="User"/>
<i class="oi oi-fw oi-filled me-1" data-icon="person"/>
<i class="oi" data-icon="oi_odoo"/>                       <!-- Odoo UI icons: oi_ prefix -->
<button name="action_send" type="object" icon="mail" icon_class="oi-filled" string="Send"/>
```

| Class                                         | Effect                               |
| --------------------------------------------- | ------------------------------------ |
| `oi-filled`                                   | Filled variant (`FILL 1`)            |
| `oi-fw`                                       | Fixed width                          |
| `oi-spin`, `oi-pulse`                         | Animation                            |
| `oi-rotate-90/180/270`, `oi-flip-horizontal`  | Transform                            |
| `oi-stack`, `oi-stack-1x`, `oi-stack-2x`      | Stacked icons                        |

- Names are Material Symbols ligatures (`check`, `settings`, `arrow_forward`, `edit_square`).
- Accessibility check: an icon `<i data-icon>` or a button with `icon` needs a `title`/
  `aria-label` (on itself, a parent or a child) or adjacent text, otherwise a view warning
  is logged (`_validate_data_icon_accessibility`).
- Migrate existing code with the best-effort rewriter (handles `fa-*` and old `oi-*` classes;
  dynamically built class names need a manual pass):

```bash
odoo-bin upgrade_code --script 19.5-00-material-symbols-icons --addons-path=/path/to/addons
```

---

## Search Views

### Basic Search View

```xml
<record id="view_my_model_search" model="ir.ui.view">
    <field name="name">my.model.search</field>
    <field name="model">my.model</field>
    <field name="arch" type="xml">
        <search string="Search My Models">
            <field name="name"/>
            <field name="code"/>
            <filter string="Active" name="active" domain="[('active', '=', True)]"/>
            <separator/>
            <filter name="filter_date" date="date" default_period="month"/>
            <group>
                <filter string="State" name="state" context="{'group_by': 'state'}"/>
            </group>
        </search>
    </field>
</record>
```

### Custom Filters

```xml
<search string="Search My Models">
    <field name="name"/>
    <field name="partner_id"/>

    <!-- Simple filter -->
    <filter string="Active" name="active" domain="[('active', '=', True)]"/>

    <!-- Advanced filter -->
    <filter string="My Partners" name="my_partners" domain="[('partner_id.user_id', '=', uid)]"/>

    <separator/>

    <!-- Group by -->
    <group>
        <filter string="State" name="state" context="{'group_by': 'state'}"/>
        <filter string="Partner" name="partner_id" context="{'group_by': 'partner_id'}"/>
    </group>
</search>
```

A standard `<filter>` requires `name`.

### Date Filters (changed in 20.0)

`start_month`, `end_month`, `start_year`, `end_year` were removed: they are not parsed and the
RNG rejects them. The dropdown always offers `month`, `month-1`, `month-2`, `year`, `year-1`,
`year-2`, quarters, and relative periods.

```xml
<!-- ❌ Odoo 19: fails RNG validation in 20 -->
<filter name="date" date="date" start_month="-2" end_month="0" start_year="-1" end_year="0"/>

<!-- ✅ Odoo 20 -->
<filter name="date" date="date" default_period="month"/>
<filter name="filter_this_week" date="date" default_period="this_week"/>

<!-- ✅ New: range overlap on two date fields (set string, it defaults to the `date` label) -->
<filter string="Period" name="filter_period" date="date_start" end_date="date_stop"/>
```

- `default_period`: `month`, `year`, `month±N`/`year±N`, `first_quarter`…`fourth_quarter`, or
  relative `today`, `this_week`, `this_month`, `this_quarter`, `this_year` (comma-separated).
- `date`/`end_date` must name an existing, searchable `date`/`datetime` field; `end_date`
  requires `date` (view error otherwise).
- `end_date` matches records whose `[date, end_date]` overlaps the period; empty end = start.
- Migrate: `odoo-bin upgrade_code --script 20.0-00-search-date-filters`.

### Grouped and Lazy-Loaded Filters (new in 20)

A `<filter>` carrying `string` (no `name` required) may contain either sub-filters or a single
`<field>`. With `<field>` (Selection, Many2one, Many2many only), options are fetched when the
user expands the filter; `domain` on the `<field>` limits the relational options, the filter
`domain` is combined with the chosen option. Lazy filters cannot be search defaults.

```xml
<filter string="Status">
    <filter string="To Approve" name="to_approve" domain="[('state', '=', 'confirm')]"/>
    <filter string="Approved" name="validated" domain="[('state', '=', 'validate')]"/>
</filter>

<filter string="My Sales Teams" name="sales_teams_switcher" domain="[('user_id', '=', uid)]">
    <field name="team_id" domain="[('user_id', '=', uid)]"/>
</filter>
```

---

## Kanban and Card Views

### Basic Kanban View

```xml
<record id="view_my_model_kanban" model="ir.ui.view">
    <field name="name">my.model.kanban</field>
    <field name="model">my.model</field>
    <field name="arch" type="xml">
        <kanban default_group_by="state" highlight_color="color">
            <field name="color"/>
            <templates>
                <t t-name="card">
                    <field name="name" class="fw-bold"/>
                    <field name="partner_id"/>
                    <footer>
                        <field name="activity_ids" widget="kanban_activity"/>
                        <field name="user_id" widget="many2one_avatar_user" class="ms-auto"/>
                    </footer>
                </t>
                <t t-name="menu">
                    <a t-if="widget.editable" role="menuitem" type="open" class="dropdown-item">Edit</a>
                    <a t-if="widget.deletable" role="menuitem" type="delete" class="dropdown-item">Delete</a>
                    <field name="color" widget="kanban_color_picker"/>
                </t>
            </templates>
        </kanban>
    </field>
</record>
```

- Template name is `card` (`kanban-box` is gone); `menu` renders in the ⋮ dropdown.
- `highlight_color` (integer field) colors the card border; the `kanban_color_picker` widget
  edits it. Drag-to-reorder uses the `handle` widget on a sequence field in the card.
- No `oe_kanban_card`/`oe_kanban_content` wrappers: use `<aside>`, `<main>`, `<footer>` and
  Bootstrap utilities.
- Kanban-only button/link `type`: `open`, `delete`, `archive`, `unarchive`, `set_cover`.

### Card View and `card_id` (new in 20)

A `card` view holds card templates that a kanban (or calendar popover) reuses via
`card_id`. The card arch is inlined at `get_views` time; the kanban needs no `<templates>`.

```xml
<record id="my_model_view_card" model="ir.ui.view">
    <field name="name">my.model.card</field>
    <field name="model">my.model</field>
    <field name="arch" type="xml">
        <card>
            <templates>
                <t t-name="card">
                    <field name="name"/>
                </t>
            </templates>
        </card>
    </field>
</record>

<record id="my_model_view_kanban" model="ir.ui.view">
    <field name="name">my.model.kanban</field>
    <field name="model">my.model</field>
    <field name="arch" type="xml">
        <kanban card_id="%(my_module.my_model_view_card)d" default_group_by="stage_id"/>
    </field>
</record>
```

The `aside` full-bleed class is `o_card_aside_full` (was `o_kanban_aside_full`).

---

## Calendar Views

### Calendar View

```xml
<record id="view_my_model_calendar" model="ir.ui.view">
    <field name="name">my.model.calendar</field>
    <field name="model">my.model</field>
    <field name="arch" type="xml">
        <calendar string="My Calendar" date_start="date_start" date_stop="date_stop"
                  color="user_id" mode="month" schedule="1">
            <field name="name"/>
            <field name="user_id"/>
        </calendar>
    </field>
</record>
```

| Attribute     | Description                                                        |
| ------------- | ------------------------------------------------------------------ |
| `date_start`  | Start date field                                                   |
| `date_stop`   | End date field (defaults to `date_start`)                          |
| `color`       | Field for color                                                    |
| `mode`        | Default scale: `day`, `week`, `month`, `year`                      |
| `schedule`    | Side panel of unscheduled records to drag into the calendar (new)  |
| `dialog_size` | Dialog size (new in 20)                                            |

- ❌ `date_delay` was removed in 20 (RNG and `_postprocess` no longer know it): store an
  end date and use `date_stop`.
- New `<popover>` child (optionally `card_id`, `<field>`s or `<templates>`) customizes the
  record popover; without it the root `<field>`s are shown.

---

## Graph Views

### Graph View

```xml
<record id="view_my_model_graph" model="ir.ui.view">
    <field name="name">my.model.graph</field>
    <field name="model">my.model</field>
    <field name="arch" type="xml">
        <graph string="Sales Analysis" type="bar">
            <field name="date" interval="month" type="row"/>
            <field name="product_id" type="col"/>
            <field name="price_total" type="measure"/>
        </graph>
    </field>
</record>
```

| Attribute   | Description           |
| ----------- | --------------------- |
| `type`      | `bar`, `pie`, `line`  |
| `stacked`   | Stack bars            |
| `cumulated` | Cumulative line chart |
| `order`     | `asc` / `desc`        |

---

## Pivot Views

### Pivot View

```xml
<record id="view_my_model_pivot" model="ir.ui.view">
    <field name="name">my.model.pivot</field>
    <field name="model">my.model</field>
    <field name="arch" type="xml">
        <pivot string="Sales Analysis">
            <field name="date" interval="month" type="row"/>
            <field name="product_id" type="col"/>
            <field name="price_total" type="measure"/>
        </pivot>
    </field>
</record>
```

---

## View Inheritance

### Extend View

```xml
<record id="view_my_model_form_inherit" model="ir.ui.view">
    <field name="name">my.model.form.inherit</field>
    <field name="model">my.model</field>
    <field name="inherit_id" ref="view_my_model_form"/>
    <field name="arch" type="xml">
        <xpath expr="//field[@name='name']" position="after">
            <field name="new_field"/>
        </xpath>
    </field>
</record>
```

### Position Options

| Position     | Description       |
| ------------ | ----------------- |
| `inside`     | Inside element    |
| `replace`    | Replace element   |
| `before`     | Before element    |
| `after`      | After element     |
| `attributes` | Modify attributes |

### Example: Add Field

```xml
<xpath expr="//field[@name='name']" position="after">
    <field name="new_field"/>
</xpath>
```

### Example: Replace Field

```xml
<xpath expr="//field[@name='name']" position="replace">
    <field name="name" widget="char" required="1"/>
</xpath>
```

### Example: Modify Attributes

```xml
<xpath expr="//field[@name='name']" position="attributes">
    <attribute name="required">True</attribute>
</xpath>
```

When upgrading views that targeted `fa-*` classes, `start_month`… or `t-set` inside a
`t-call`, update the xpaths too: the parent views changed.

---

## QWeb Templates

### Basic Template

```xml
<template id="my_template" name="My Template">
    <div class="my-class">
        <h1>Hello World</h1>
    </div>
</template>
```

### Template Inheritance

```xml
<template id="my_template_inherit" inherit_id="my_template" name="My Template Inherit">
    <xpath expr="//div[@class='my-class']" position="inside">
        <p>This is inherited content</p>
    </xpath>
</template>
```

### QWeb Directives

| Directive   | Description                                              |
| ----------- | -------------------------------------------------------- |
| `t-out`     | Escape and output (`Markup` values are not escaped)      |
| `t-field`   | Output formatted field                                   |
| `t-if`      | Conditional                                              |
| `t-elif`    | Else if                                                  |
| `t-else`    | Else                                                     |
| `t-foreach` | Loop                                                     |
| `t-as`      | Loop variable                                            |
| `t-set`     | Set variable (`t-value`, `t-valuef`, `t-valuef.translate`) |
| `t-call`    | Call template (only on `<t>`)                            |
| `t-att-*`   | Set attribute                                            |

❌ `t-esc` and `t-raw` are gone from server QWeb in 20 (no compile method; they are logged as
"Unknown directives or unused attributes" and output nothing). Use `t-out`
(`Markup(...)` for raw HTML).

### t-call Parameters (changed in 19.1)

Values for the called template are passed as attributes of the `t-call` element. A `t-set`
in the body of a `t-call` is now local to the body (rendered as `0`); it no longer reaches the
called template (the 19.0 compatibility path was removed).

```xml
<!-- ❌ Odoo 19: `title` no longer reaches the callee -->
<t t-call="portal.portal_searchbar">
    <t t-set="title">Events</t>
</t>

<!-- ✅ Odoo 20 -->
<t t-call="portal.portal_searchbar" title.translate="Events"/>
<t t-call="web.external_layout" company="doc.company_id.sudo()" address="address"/>
<t t-call="portal.placeholder_user_sign_in" _item_class.f="nav-item"/>
```

| Attribute form   | Value                                         |
| ---------------- | --------------------------------------------- |
| `name="expr"`    | Python expression                             |
| `name.f="..."`   | Format string (`{{ expr }}` interpolation)    |
| `name.translate` | Translatable format string                    |
| `t-args="expr"`  | Dict (or pairs) merged into the callee values |

The callee still sees the caller's values; body content is available as `t-out="0"`.
Migrate: `odoo-bin upgrade_code --script 19.1-00-t-call` (inherited `t-call` xpaths become
`position="attributes"`).

---

## References

- Odoo 20 documentation: `content/developer/reference/user_interface/view_architectures.rst`,
  `user_interface/icons.rst`, `frontend/qweb.rst`
- Source: `odoo/addons/base/rng/*.rng`, `odoo/addons/base/models/ir_ui_view.py`,
  `odoo/addons/base/models/ir_qweb.py`, `odoo/upgrade_code/`
