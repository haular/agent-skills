# Odoo 20 Development Guide

This file provides guidance to AI agents when working with Odoo 20 code in this repository.

> **For setup instructions with different AI IDEs, see [AGENTS.md](./AGENTS.md)**

## Documentation Structure

The `skills/odoo-20.0/references/` directory contains modular guides for Odoo 20 development:

```
skills/odoo-20.0/
├── SKILL.md                       # Master index
├── references/                    # Development guides (18 files)
│   ├── odoo-20-actions-guide.md     # ir.actions.*, cron, bindings
│   ├── odoo-20-controller-guide.md  # HTTP, routing, controllers
│   ├── odoo-20-data-guide.md        # XML/CSV data files, records
│   ├── odoo-20-decorator-guide.md   # @api decorators
│   ├── odoo-20-development-guide.md # Manifest, wizards (overview)
│   ├── odoo-20-field-guide.md       # Field types, parameters
│   ├── odoo-20-manifest-guide.md    # __manifest__.py reference
│   ├── odoo-20-mixins-guide.md      # mail.thread, activities, etc.
│   ├── odoo-20-model-guide.md       # ORM, CRUD, search, domain
│   ├── odoo-20-migration-guide.md   # Migration scripts, hooks
│   ├── odoo-20-owl-guide.md         # OWL components, services
│   ├── odoo-20-performance-guide.md # N+1 prevention, optimization
│   ├── odoo-20-reports-guide.md     # QWeb reports, PDF/HTML
│   ├── odoo-20-security-guide.md    # ACL, record rules, security
│   ├── odoo-20-testing-guide.md     # Test classes, decorators
│   ├── odoo-20-transaction-guide.md # Savepoints, errors
│   ├── odoo-20-translation-guide.md # Translations, i18n
│   └── odoo-20-view-guide.md        # XML views, QWeb
├── CLAUDE.md                      # This file
└── AGENTS.md                      # AI agents setup
```

## Which Guide to Use

| Task                                  | Guide                                     |
| ------------------------------------- | ----------------------------------------- |
| Creating actions, menus, cron jobs    | `references/odoo-20-actions-guide.md`     |
| Creating a new module                 | `references/odoo-20-development-guide.md` |
| Configuring **manifest**.py           | `references/odoo-20-manifest-guide.md`    |
| Creating XML/CSV data files           | `references/odoo-20-data-guide.md`        |
| Writing ORM queries/search            | `references/odoo-20-model-guide.md`       |
| Defining model fields                 | `references/odoo-20-field-guide.md`       |
| Using @api decorators                 | `references/odoo-20-decorator-guide.md`   |
| Writing XML views                     | `references/odoo-20-view-guide.md`        |
| Fixing slow code/N+1 queries          | `references/odoo-20-performance-guide.md` |
| Handling database errors              | `references/odoo-20-transaction-guide.md` |
| Creating HTTP endpoints               | `references/odoo-20-controller-guide.md`  |
| Building OWL components               | `references/odoo-20-owl-guide.md`         |
| Upgrading modules/migrating data      | `references/odoo-20-migration-guide.md`   |
| Using mail.thread, activities, mixins | `references/odoo-20-mixins-guide.md`      |
| Creating QWeb reports                 | `references/odoo-20-reports-guide.md`     |
| Configuring security (ACL, rules)     | `references/odoo-20-security-guide.md`    |
| Writing tests                         | `references/odoo-20-testing-guide.md`     |
| Adding translations/localization      | `references/odoo-20-translation-guide.md` |

## Key Odoo 20 Changes

Odoo 20 bundles the Odoo Online 19.1-19.5 changes. Full list with evidence: `references/api-highlights.md`.

| Change               | Old (Odoo 19)                                   | New (Odoo 20)                                              |
| -------------------- | ----------------------------------------------- | ---------------------------------------------------------- |
| Access control       | `ir.model.access.csv` + `ir.rule`               | `security/ir.access.csv` (`ir.access`, `operation=crud`)   |
| Access check API     | `check_access_rights()` / `check_access_rule()` | `check_access()` / `has_access()`; override `_access_domain` |
| Binary fields        | base64 strings, `ir.attachment.datas`           | `BinaryValue` / `BinaryBytes`, `.content`, `raw`           |
| XML binary data      | `type="base64" file="..."`                      | `type="bytes" file="..."`                                  |
| Config parameters    | `get_param()` / `set_param()`                   | `get_str/get_int/get_bool/get_float`, `set_*`              |
| SQL view models      | `_table_query = "SELECT ..."`                   | `_table_sql` property returning `SQL`                      |
| Domain helpers       | `from odoo.osv import expression`               | `from odoo.fields import Domain` (`odoo.osv` removed)      |
| ormcache             | `tools.ormcache`, `registry.clear_cache()`      | `api.ormcache`, `env.transaction.invalidate_ormcache()`    |
| Search on computes   | `store=True` just to search/group               | `compute_sql=`                                             |
| Icons                | `fa fa-*`                                       | `<i class="oi" data-icon="..."/>` (Material Symbols)       |
| QWeb output          | `t-esc` / `t-raw` (deprecated)                  | `t-out` (`t-esc` / `t-raw` removed)                        |
| t-call parameters    | `<t t-set>` inside `<t t-call>`                 | attributes: `<t t-call="x" title="expr"/>`                 |
| HTTP imports         | `from odoo.http import Stream`                  | `from odoo.http.stream import Stream` (`odoo/http/` package) |
| External API         | XML-RPC / JSON-RPC                              | `POST /json/2/<model>/<method>` (RPC deprecated)           |
| Frontend             | OWL 2                                           | OWL 3 (`upgrade_code --script owl3-migration`)             |
| PDF engine           | wkhtmltopdf hard-wired                          | engine modules (`base_report_wkhtmltox`, paper-muncher)    |
| Default test tag     | `at_install`                                    | `post_install`                                             |
| Tracking hooks       | `_track_subtype`, `_track_template`             | `_track_log_get_default_subtype`, `_track_template_parameters` |

Still valid from 19: `<list>`, direct-expression attrs, `@api.ondelete`, `aggregator=`, `models.Constraint` / `Index`, `privilege_id` groups, `@api.private`, `_read_group()`, explicit `_name`.

## Critical Anti-Patterns

| Anti-Pattern                                                   | Why Bad                         | Correct Approach                                   |
| -------------------------------------------------------------- | ------------------------------- | -------------------------------------------------- |
| `security/ir.model.access.csv` / `ir.rule` records             | Models removed in Odoo 20       | `security/ir.access.csv`                           |
| `check_access_rights()` / `check_access_rule()`                | Removed in Odoo 20              | `check_access()` / `has_access()`                  |
| `base64.b64encode(...)` into a Binary field                    | Binary fields hold `BinaryValue` | `BinaryBytes(data, filename=...)`                 |
| `ir.config_parameter.get_param(...)`                           | Removed                         | `get_str(...)` / `get_int(...)` / `get_bool(...)`  |
| `@api.depends('partner_id')` then accessing `partner_id.email` | N queries per record            | Add `@api.depends('partner_id.email')`             |
| `search()` inside loop                                         | N+1 queries                     | Use `search()` with `IN` domain or `_read_group()` |
| `create()` in loop                                             | N INSERT statements             | Batch: `create([{...}, {...}])`                    |
| Overriding `unlink()` for validation                           | Breaks module uninstall         | Use `@api.ondelete(at_uninstall=False)`            |
| Using `_sql_constraints`                                       | Ignored since Odoo 19           | Use `models.Constraint(...)`                       |
| Using `t-esc` / `t-raw` in server QWeb                         | Removed: renders nothing        | Use `t-out` instead                                |
| `fa fa-*` icon classes                                         | Font Awesome not bundled        | `oi` + `data-icon`                                 |
| `read_group(domain, fields, groupby)`                          | Signature changed in Odoo 20    | `_read_group()` or `formatted_read_group()`        |

## Coding Conventions

Before writing code, run the `odoo-workflow` skill if it is installed (trace the real
source, Context Brief with `file:line`, definition of done); install it alongside this pack.

Read `references/api-highlights.md` before writing code and apply its Quick review
checks: the ❌ items are what not to use in Odoo 20.

Apply the Coding Conventions section in the guide you open. Odoo 20 runtime
behavior takes precedence, followed by the surrounding stable-addon style;
keep convention-only changes focused. The common source is the [Odoo Coding
Guidelines](https://raw.githubusercontent.com/odoo/documentation/17.0/content/contributing/development/coding_guidelines.rst).

- Use the development guide for module layout, assets, JavaScript, CSS, and
  SCSS conventions.
- Use data, actions, reports, security, and views guides for XML formatting
  and external IDs.
- Use model, field, decorator, performance, transaction, and translation
  guides for Python, recordsets, context, database work, and i18n.

## @api Decorator Decision Tree

```
Need to define field behavior?
├── Field computed from other fields → @api.depends
│   └── CAN use dotted paths: `@api.depends('partner_id.email')`
├── Validate data → @api.constrains
│   └── CANNOT use dotted paths: only simple field names
├── Prevent record deletion → @api.ondelete
└── Update form UI → @api.onchange
    └── NO CRUD operations allowed

Need to define method behavior?
├── Method-level, doesn't depend on self → @api.model
├── Mark method as non-RPC callable → @api.private
├── Cache a pure lookup → @api.ormcache
└── Normal record method → no decorator needed
```

## Common Patterns Reference

### N+1 Query Prevention

```python
# BAD: search in loop
for order in orders:
    payments = self.env['payment'].search([('order_id', '=', order.id)])

# GOOD: single query
payments = self.env['payment'].search_read([('order_id', 'in', orders.ids)])
```

### List View (Odoo 20)

```xml
<list string="Records" editable="bottom" multi_edit="1">
    <field name="state" decoration-success="state == 'done'"/>
    <field name="phone" optional="show"/>
</list>
```

### Delete Validation (Odoo 20)

```python
@api.ondelete(at_uninstall=False)
def _unlink_if_not_draft(self):
    if any(rec.state != 'draft' for rec in self):
        raise UserError("Cannot delete non-draft records")
```

## Module Structure

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
│   ├── ir.access.csv
│   ├── my_module_groups.xml
│   └── ir_access.xml
├── data/
│   └── my_model_data.xml
├── migrations/
│   └── 20.0.1.0/
│       └── post-migration.py
├── tests/
│   ├── __init__.py
│   └── test_my_model.py
├── wizard/
│   ├── __init__.py
│   └── make_my_model.py
├── controllers/
│   ├── __init__.py
│   └── my_model.py
└── static/
    └── src/
        ├── js/
        │   └── my_component.js
        ├── xml/
        │   └── my_component.xml
        └── scss/
            └── my_style.scss
```

## Base Code Reference

The guides are based on Odoo 20 source code. Reference these files in your Odoo installation:

- `odoo/orm/models.py` - ORM implementation
- `odoo/orm/fields.py` - Field types
- `odoo/orm/decorators.py` - Decorators
- `odoo/http/` - HTTP layer (package: `routing_map.py`, `dispatcher.py`, `requestlib.py`, `stream.py`)
- `odoo/orm/cache.py` - ormcache
- `odoo/orm/domains.py` - Domain
- `odoo/addons/base/models/ir_access.py` - Access control
- `odoo/exceptions.py` - Exception types
- `odoo/tools/translate.py` - Translation system
- `odoo/addons/base/models/res_lang.py` - Language model
- `addons/web/static/src/core/l10n/translation.js` - JS translations
