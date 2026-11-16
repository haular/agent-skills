# Odoo 20 Development Guide

## Coding Conventions

Use singular, descriptive module names and model-oriented filenames. Keep
Python, XML, JS, CSS, and SCSS concise and purpose-specific; Odoo 20 runtime
behavior and established addon style take precedence over a style-only rewrite.

Guide for developing Odoo 20 modules: creating modules, manifest, structure, and common patterns.
Runtime: Python ≥ 3.12 (≤ 3.14), PostgreSQL ≥ 16 (`odoo/release.py`); web client on OWL 3.

## Table of Contents
- [Module Structure](#module-structure)
- [Creating a Module](#creating-a-module)
- [Manifest File](#manifest-file)
- [Models](#models)
- [Views](#views)
- [Security](#security)
- [Data Files](#data-files)
- [Assets](#assets)
- [Wizards](#wizards)
- [Command Line](#command-line)

---

## Module Structure

### Standard Structure

```
my_module/
├── __init__.py
├── __manifest__.py
├── models/
│   ├── __init__.py
│   └── my_model.py
├── views/
│   └── my_model_views.xml
├── security/
│   ├── my_module_groups.xml      # res.groups / res.groups.privilege
│   └── ir.access.csv             # ir.access (20: replaces ir.model.access.csv + ir.rule)
├── data/
│   └── my_model_data.xml
├── demo/
│   └── demo_data.xml
├── populate/                     # optional: populate.blueprint records (+ generators)
│   └── my_module.xml
├── migrations/
│   └── 20.0.1.0/
│       └── post-migrate.py
├── tests/
│   ├── __init__.py
│   └── test_my_model.py
├── wizard/
│   ├── __init__.py
│   └── make_my_model.py
├── controllers/
│   ├── __init__.py
│   └── my_model.py
├── i18n/                         # <lang>.po; also holds manifest name/summary/description terms
├── static/
│   ├── src/
│   │   ├── js/
│   │   ├── xml/
│   │   └── scss/
│   └── description/
│       └── icon.png
└── report/
    ├── my_model_templates.xml
    └── my_model_reports.xml
```

---

## Creating a Module

### Step 1: Scaffold

```bash
odoo-bin scaffold my_module /path/to/addons     # -t/--template <name|path>, default "default"
```

⚠️ The built-in `default` template (`odoo/cli/templates/default/`) still generates
`security/ir.model.access.csv` with `ir.model.access` columns: delete it and write
`security/ir.access.csv` instead (see [Security](#security)).

### Step 2: __init__.py

```python
# __init__.py
from . import models
from . import controllers
```

```python
# models/__init__.py
from . import my_model
```

### Step 3: Model

```python
# models/my_model.py
from odoo import fields, models


class MyModel(models.Model):
    _name = 'my.model'
    _description = 'My Model'

    name = fields.Char(string="Name", required=True)
    description = fields.Text(string="Description")
    active = fields.Boolean(string="Active", default=True)
```

### Step 4: Manifest

```python
# __manifest__.py
{
    'name': 'My Module',
    'version': '1.0.0',
    'category': 'Tools',
    'summary': 'My awesome module',
    'author': 'Your Name',
    'website': 'https://github.com/yourname/my_module',
    'license': 'LGPL-3',
    'depends': ['base'],
    'data': [
        'security/my_module_groups.xml',
        'data/my_model_data.xml',
        'views/my_model_views.xml',
        'report/my_model_templates.xml',
        'report/my_model_reports.xml',
        'views/my_module_menus.xml',
        'security/ir.access.csv',
    ],
    'demo': [
        'demo/demo_data.xml',
    ],
    'assets': {
        'web.assets_backend': [
            'my_module/static/src/**/*',
        ],
    },
    'application': False,
}
```

---

## Manifest File

See `odoo-20-manifest-guide.md` for every key. Essentials:

- `name` is the only key without a default; a missing `author` or `license` logs a warning
  (license defaults to `LGPL-3`).
- `version`: `x.y`, `x.y.z`, `20.0.x.y` or `20.0.x.y.z`; another series makes the module
  non-installable.
- Empty `depends` becomes `['base']`.
- ❌ `init_xml`, `update_xml`, `demo_xml` (removed in 20); ✅ `data`, `demo`, `other_files`.
- Demo data is only loaded with `--with-demo`.
- `name`/`summary`/`description` translations go in the module's own `i18n/*.po` (20.0).

---

## Models

### Basic Model

```python
from odoo import fields, models


class MyModel(models.Model):
    _name = 'my.model'
    _description = 'My Model'
    _order = 'name'
    _rec_names_search = ('name', 'code')    # tuple since 19.5 (❌ ['name', 'code'])

    name = fields.Char(string="Name", required=True)
    code = fields.Char(string="Code")
    description = fields.Text(string="Description")
    active = fields.Boolean(string="Active", default=True)
```

### Model Inheritance (Extension)

```python
class ResPartner(models.Model):
    _inherit = 'res.partner'

    my_field = fields.Char(string="My Field")
```

### Model Inheritance (Prototype)

```python
class NewModel(models.Model):
    _name = 'new.model'
    _inherit = 'base.model'

    # Copies fields and methods of base.model into a new table
    my_field = fields.Char(string="My Field")
```

---

## Views

### Create Views

```xml
<?xml version="1.0" encoding="UTF-8"?>
<odoo>
    <record id="my_model_view_list" model="ir.ui.view">
        <field name="name">my.model.list</field>
        <field name="model">my.model</field>
        <field name="arch" type="xml">
            <list string="My Models">
                <field name="name"/>
                <field name="code"/>
                <field name="active"/>
            </list>
        </field>
    </record>

    <record id="my_model_view_form" model="ir.ui.view">
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
                        </group>
                    </group>
                    <notebook>
                        <page string="Description">
                            <field name="description"/>
                        </page>
                    </notebook>
                </sheet>
            </form>
        </field>
    </record>

    <record id="my_model_view_search" model="ir.ui.view">
        <field name="name">my.model.search</field>
        <field name="model">my.model</field>
        <field name="arch" type="xml">
            <search string="Search My Models">
                <field name="name"/>
                <field name="code"/>
                <filter string="Archived" name="inactive" domain="[('active', '=', False)]"/>
            </search>
        </field>
    </record>

    <record id="action_my_model" model="ir.actions.act_window">
        <field name="name">My Models</field>
        <field name="res_model">my.model</field>
        <field name="view_mode">list,form</field>
        <field name="help" type="html">
            <p class="o_view_nocontent_smiling_face">
                Create your first my.model!
            </p>
        </field>
    </record>

    <menuitem id="menu_my_model_root" name="My Module" web_icon="my_module,static/description/icon.png"/>
    <menuitem id="menu_my_model" name="My Models" parent="menu_my_model_root" action="action_my_model"/>
</odoo>
```

### Icons in Views (Material Symbols, 19.5)

Font Awesome is removed; use Material Symbols names.

```xml
<!-- ❌ --> <button name="action_open" type="object" icon="fa-star"/>
<!-- ✅ --> <button name="action_open" type="object" icon="star" icon_class="oi-filled"/>
<!-- ❌ --> <i class="fa fa-shopping-cart fa-fw"/>
<!-- ✅ --> <i class="oi oi-fw" data-icon="shopping_cart" role="img" aria-label="Orders"/>
```

---

## Security

Since 19.4, `ir.access` replaces both `ir.model.access` and `ir.rule` (neither model exists
in 20). One CSV, listed last in `data` (see `odoo-20-security-guide.md` for details):

File: `security/ir.access.csv`

```csv
id,name,model_id,group_id/id,operation,domain
access_my_model_user,my.model user,my.model,base.group_user,cru,"[('create_uid', '=', user.id)]"
access_my_model_manager,my.model manager,my.model,my_module.group_my_module_manager,crud,
my_model_comp_rule,My Model multi-company,my.model,,crud,"[('company_id', 'in', company_ids)]"
```

- `model_id`: the model `_name` (❌ `model_my_model` / `model_id:id`).
- `operation`: subset of `c`, `r`, `u`, `d` (❌ `perm_read,perm_write,perm_create,perm_unlink`).
- `domain`: with a group, grants access to matching records; with an empty group, it
  **restricts** all users to matching records (former global rule).
- Transient models (wizards) need access rows too.

---

## Data Files

### Create Records

```xml
<odoo>
    <record id="my_model_1" model="my.model">
        <field name="name">Record 1</field>
        <field name="code">R001</field>
    </record>
</odoo>
```

### noupdate Flag

```xml
<odoo>
    <data noupdate="1">
        <!-- Created on install, not overwritten on update -->
        <record id="default_record" model="my.model">
            <field name="name">Default</field>
        </record>
    </data>

    <!-- Loaded on install and update -->
    <record id="core_data" model="my.model">
        <field name="name">Core</field>
    </record>
</odoo>
```

---

## Assets

### JavaScript, CSS, and SCSS Conventions

- Put addon source in `static/src/` (`js`, `xml`, `scss`, `css` or per-feature folders);
  reserve `static/lib` for bundled third-party code.
- One component or concern per file, explicit ES module imports/exports
  (`@odoo/owl`, `@web/...`, `@my_module/...`). Do not load remote third-party assets from a manifest.
- Write OWL 3 code: `props = useProps(...)`, `signal`/`proxy`, `usePlugin`, `this.` in
  templates, `t-call-slot` (see `odoo-20-owl-guide.md`). Do not add new imports from
  `@web/owl2/utils` (temporary compatibility layer).
- Icons: `<i class="oi" data-icon="name"/>`; SCSS selectors `[data-icon='name']`
  (❌ `.fa-*`).
- Prefer SCSS for themed styles. Scope selectors beneath a module/component class, use
  variables and nesting sparingly, and avoid `!important`.
- Keep CSS/SCSS presentation-only and put behavior in JS; list assets in the
  appropriate bundle in dependency order.

### Bundles

```python
'assets': {
    'web.assets_backend': [
        'my_module/static/src/**/*',
        ('remove', 'my_module/static/src/dashboard/**/*'),
    ],
    'my_module.dashboard': [                  # lazy-loaded bundle (LazyComponent)
        'my_module/static/src/dashboard/**/*',
    ],
    'web.assets_frontend': [
        'my_module/static/src/js/frontend.js',
        'my_module/static/src/scss/frontend.scss',
    ],
    'web.assets_unit_tests': [
        'my_module/static/tests/**/*',
    ],
},
```

---

## Wizards

### TransientModel

```python
from odoo import fields, models


class MyModelMake(models.TransientModel):
    _name = 'my.model.make'
    _description = 'Make My Model'

    date = fields.Date(string="Date", required=True, default=fields.Date.context_today)
    note = fields.Text(string="Note")

    def action_confirm(self):
        # Do something
        return {'type': 'ir.actions.act_window_close'}
```

### Wizard View

```xml
<record id="my_model_make_view_form" model="ir.ui.view">
    <field name="name">my.model.make.form</field>
    <field name="model">my.model.make</field>
    <field name="arch" type="xml">
        <form string="Make My Model">
            <group>
                <field name="date"/>
                <field name="note"/>
            </group>
            <footer>
                <button name="action_confirm" string="Confirm" type="object" class="btn-primary"/>
                <button string="Cancel" class="btn-secondary" special="cancel"/>
            </footer>
        </form>
    </field>
</record>
```

### Action to Open Wizard

```python
def action_open_wizard(self):
    return {
        'type': 'ir.actions.act_window',
        'name': self.env._("Make My Model"),
        'res_model': 'my.model.make',
        'view_mode': 'form',
        'target': 'new',
        'context': {
            'default_date': fields.Date.context_today(self),
        },
    }
```

---

## Common Patterns

### State Field

```python
state = fields.Selection([
    ('draft', 'Draft'),
    ('confirmed', 'Confirmed'),
    ('done', 'Done'),
], string='State', default='draft', tracking=True)   # tracking requires mail.thread
```

### Default from Context

```python
@api.model
def default_get(self, fields):
    defaults = super().default_get(fields)
    if 'field' in fields:
        defaults['field'] = self.env.context.get('default_field', 'default')
    return defaults
```

### Name Search

✅ Declare searchable fields instead of overriding `name_search`:

```python
_rec_names_search = ('name', 'code')
```

If you must override, the 20.0 signature is
`name_search(self, name='', domain=None, operator='ilike', limit=100)`
(❌ `args=`); prefer overriding `_search_display_name(self, operator, value)`.

---

## Command Line

`odoo-bin <command>` (built-ins in `odoo/cli/`, addon commands in `<addon>/cli/<command>.py`):

| Command        | Purpose                                                                 |
| -------------- | ----------------------------------------------------------------------- |
| `server`       | Default: run the server                                                 |
| `scaffold`     | `odoo-bin scaffold [-t TEMPLATE] NAME [DEST]`                           |
| `shell`        | Interactive shell on a database                                         |
| `db`           | `init`, `load`, `dump`, `duplicate`, `rename`, `drop`                   |
| `module`       | `install`, `upgrade`, `uninstall`, `force-demo`, `list` (`-d DB`)       |
| `i18n`         | `export -l fr MODULE...`, `import -l fr FILE...`, `loadlang -l fr`      |
| `upgrade_code` | Source rewrites: `--script NAME`, `--from VERSION`, `--dry-run`         |
| `duplicate`    | **20: former `populate`** — copy existing records: `--models`, `--factors` |
| `populate`     | **20, `populate` addon** — synthetic data from a `populate.blueprint`: `-b BLUEPRINT`, `--scale`, `--seed`, `-j`, `--resume`, `--profile` |
| `cloc`, `neutralize`, `obfuscate`, `deploy`, `start`, `help` | as in previous versions          |

```bash
odoo-bin populate -d mydb -b sale.fake_sale_demo --scale 0.1   # requires the populate module installed
odoo-bin duplicate -d mydb --models res.partner --factors 10
```

---

## References

- `odoo/modules/module.py`, `odoo/modules/loading.py`, `odoo/release.py`
- `odoo/cli/`, `addons/populate/cli/populate.py`
- `odoo/addons/base/models/ir_access.py`
- `odoo/upgrade_code/` (19.1 … 20.0 scripts, `owl3-migration.py`)
