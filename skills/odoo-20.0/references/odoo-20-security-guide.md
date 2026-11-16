# Odoo 20 Security Guide

## Coding Conventions

- Keep groups in `<module>_groups.xml` (or `res_groups.xml`) and all access
  records in `security/ir.access.csv`; list the groups file before the CSV in the
  manifest. `ir.model.access.csv` and `ir.rule` records no longer load (models removed).
- Name permission IDs `access_<model>_<role>` and restriction IDs
  `<model>_<purpose>_rule`. Keep access domains narrow and never rely on a UI
  restriction as access control.
- Bind any recordset obtained through `sudo()` to a name ending in `_sudo`
  (`partner_sudo = self.partner_id.sudo()`,
  `orders_sudo = self.env['sale.order'].sudo().search(domain)`). A one-shot call
  (`record.sudo().write(vals)`) needs no variable. Never return a `_sudo` recordset
  from a public method or keep it on `self`.

Guide for Odoo 20 security: groups, `ir.access` (permissions and restrictions),
access-check API, field permissions, and security pitfalls.

## Table of Contents

- [Security Overview](#security-overview)
- [Groups](#groups)
- [Access Records (ir.access)](#access-records-iraccess)
- [Permissions vs Restrictions](#permissions-vs-restrictions)
- [Checking Access in Code](#checking-access-in-code)
- [Field Access](#field-access)
- [Security Pitfalls](#security-pitfalls)

---

## Security Overview

> **Breaking change (19.4)**: `ir.model.access` (ACL) and `ir.rule` (record rules)
> are **removed** and merged into a single model, `ir.access`. Each record grants
> (or restricts) a set of CRUD operations on a model, optionally limited by a
> domain. Convert old modules with `odoo-bin upgrade_code` (script
> `odoo/upgrade_code/19.4-00-ir-access.py`), which rewrites
> `ir.model.access.csv` + `ir.rule` XML into `ir.access.csv`.

`ir.access` records are linked to users through **groups**:

- **Permission** (`group_id` set): grants the operations on the records matching its domain.
- **Restriction** (`group_id` empty): applies to every user and limits all permissions.

---

## Groups

`res.groups` - Users belong to groups, security mechanisms are associated to groups.

> `category_id` was **removed** from `res.groups` in 19.0. Groups are attached to a
> `res.groups.privilege` through `privilege_id`.

### Key Attributes

| Attribute      | Description                                                         |
| -------------- | ------------------------------------------------------------------- |
| `name`         | User-readable identification (role/purpose)                         |
| `privilege_id` | Reference to `res.groups.privilege` (labelled "Scope")              |
| `implied_ids`  | Other groups to set on the user alongside this one                  |
| `access_ids`   | One2many to `ir.access` records of the group                        |
| `comment`      | Additional notes                                                    |

### 3-Tier Security Architecture

```
ir.module.category → res.groups.privilege → res.groups → res.users
```

`base.group_everyone` ("Role / Everyone") is implied by `group_user`, `group_portal`
and `group_public`: use it for a permission that every user should get.

### Example

```xml
<!-- Step 1: Define privilege -->
<record id="privilege_my_module" model="res.groups.privilege">
    <field name="name">My Module Access</field>
    <field name="category_id" ref="base.module_category_my_module"/>
</record>

<!-- Step 2: Define group with privilege -->
<record id="group_my_module_user" model="res.groups">
    <field name="name">My Module User</field>
    <field name="privilege_id" ref="privilege_my_module"/>
    <field name="implied_ids" eval="[(4, ref('base.group_user'))]"/>
    <field name="comment">Users can access my module features.</field>
</record>
```

---

## Access Records (ir.access)

`ir.access` - one model for model-level access and record-level conditions.

### Key Attributes

| Attribute   | Description                                                                   |
| ----------- | ----------------------------------------------------------------------------- |
| `name`      | Purpose or role (required)                                                    |
| `model_id`  | Model the access applies to (required)                                        |
| `group_id`  | Group granted access; **empty = restriction applied to every user**           |
| `operation` | Subsequence of `"crud"` (required): `crud`, `cru`, `r`, `ru`, `cd`, ...       |
| `domain`    | Python expression evaluating to a domain; empty = all records                 |
| `active`    | Inactive records are ignored                                                  |

`operation` letters: `c` = create, `r` = read, `u` = update (write), `d` = delete
(unlink). The order is fixed: write `cru`, never `rcu`. Computed booleans
`for_create`, `for_read`, `for_write`, `for_unlink` mirror it.

❌ Removed: `ir.model.access` (`perm_read`, `perm_write`, `perm_create`,
`perm_unlink`) and `ir.rule` (`groups`, `domain_force`, `perm_*`, `global`).

### Example CSV (standard format)

File: `security/ir.access.csv` (add it to the manifest `data` list)

```csv
id,name,model_id,group_id/id,operation,domain
access_my_model_user,my.model user,my.model,my_module.group_my_module_user,cru,"[('user_id', '=', user.id)]"
access_my_model_manager,my.model manager,my.model,my_module.group_my_module_manager,crud,
my_model_comp_rule,My Model: multi-company,my.model,,crud,"[('company_id', 'in', company_ids + [False])]"
```

- `model_id` holds the **model name** (`sale.order`), not `model_sale_order`
  (resolved by name; `ir.model._rec_names_search = ('name', 'model')`).
- `group_id/id` holds the group external ID; leave it empty for a restriction.
- Multi-line domains are allowed inside the quoted cell.

```csv
# ❌ Odoo 19 (file no longer loads)
id,name,model_id:id,group_id:id,perm_read,perm_write,perm_create,perm_unlink
access_my_model_user,my.model.user,model_my_model,group_my_module_user,1,1,0,0
```

### Example XML

```xml
<record id="project_task_rule_portal_project_sharing" model="ir.access">
    <field name="name">Project/Task: portal users can edit with project sharing</field>
    <field name="model_id" ref="project.model_project_task"/>
    <field name="group_id" ref="base.group_portal"/>
    <field name="operation">cu</field>
    <field name="domain">[('project_id.collaborator_ids.partner_id', '=', user.partner_id.id)]</field>
</record>

<!-- Disable another module's access record (former ACLs and rules keep their xmlids) -->
<record id="product.product_pricelist_comp_rule" model="ir.access">
    <field name="active" eval="False"/>
</record>
```

### Domain Variables

| Variable      | Description                                                         |
| ------------- | ------------------------------------------------------------------- |
| `user`        | Current user (singleton recordset, empty context)                   |
| `company_id`  | Current company (single id, `self.env.company.id`)                  |
| `company_ids` | Companies active in the company switcher (`self.env.companies.ids`) |
| `time`        | Python `time` module                                                |

This is `ir.access._eval_context()` in `base`; `website` adds `website`
(`self.env.website`). `uid`, `datetime` and `context` are not available. Use
`company_ids`, not `user.company_ids` (every company the user may access, ignoring
the switcher).

### The `access` Domain Operator (19.3)

`(field, 'access', operation)` is true when the current user may perform
`operation` (`'read'`, `'write'`, `'create'`, `'unlink'`) on the record that the
many2one `field` points to (or on the record itself with `'id'`). Use it to make a
line follow its parent's access instead of copying the parent's domain.

```csv
id,name,model_id,group_id/id,operation,domain
base_user_account_move_line_rule,Normal User Account Move Line,account.move.line,sales_team.group_sale_salesman,r,"[('move_id', 'access', 'read')]"
```

- Works only on many2one fields and `id`; not on properties.
- Usable in any domain (`search`, `filtered_domain`), not only in `ir.access`.

### Multi-Company Restriction

```csv
id,name,model_id,group_id/id,operation,domain
my_model_comp_rule,My Model: multi-company,my.model,,crud,"[('company_id', 'in', company_ids + [False])]"
```

The restriction only filters records. To stop a record from linking to another
company's records, set `_check_company_auto = True` on the model and
`check_company=True` on company-scoped relations (core pattern, e.g. `sale.order`):

```python
class MyModel(models.Model):
    _name = 'my.model'
    _check_company_auto = True  # create()/write() call _check_company()

    company_id = fields.Many2one('res.company', required=True, default=lambda self: self.env.company)
    partner_id = fields.Many2one('res.partner', check_company=True)  # also adds a company domain in the UI
```

---

## Permissions vs Restrictions

For a user and an operation, the security domain is
`OR(permissions of the user's groups) & AND(all restrictions)`
(`Model._access_domain()`).

| Record kind                  | Behavior                                                         |
| ---------------------------- | ---------------------------------------------------------------- |
| **Permission** (group set)   | **Unify** - any matching permission grants access                |
| **Restriction** (no group)   | **Intersect** - every restriction must be satisfied              |
| **Permission + Restriction** | **Intersect** - a restriction can remove access granted by a permission |

- No permission for the operation → no access at all (model-level error).
- A restriction never grants anything: a model with only restrictions is inaccessible.
- A permission's domain both grants the model operation and limits the records:
  there is no separate "ACL then rule" step any more.
- `_inherits` models also require access to the parent records.

> **Danger**: Creating multiple restrictions is risky - non-overlapping
> restrictions remove all access.

---

## Checking Access in Code

| API                                   | Result                                                       |
| ------------------------------------- | ------------------------------------------------------------ |
| `records.check_access(operation)`     | Raise `AccessError` if any record is forbidden (`@api.private`) |
| `records.has_access(operation)`       | Same check, returns a bool                                   |
| `records._filtered_access(operation)` | Subset of records the user may access                        |
| `Model._access_domain(operation)`     | Security `Domain` for the current user (ormcached)           |
| `Model.browse().check_access(op)`     | Empty recordset: checks model-level permission only          |

```python
# ✅ Odoo 20
order.check_access('write')
if not lines.has_access('read'):
    ...
visible = lines._filtered_access('read')

# ❌ Removed in 20.0 (deprecated since 18.0)
order.check_access_rights('write')
order.check_access_rule('write')
```

To customize access for a model, override `_access_domain` (core pattern,
`mail/models/mail_mail.py`):

```python
def _access_domain(self, operation):
    if not self.env['mail.message'].has_access(operation):
        return Domain.FALSE
    return super()._access_domain(operation)
```

> Overriding `_check_access` is deprecated: `@deprecated("Since 20.0, use
> Model._access_domain instead")`, and `check_access`/`has_access` no longer call it.

---

## Field Access

An ORM field can have a `groups` attribute providing a list of groups (comma-separated external identifiers).
Field access is independent of `ir.access`.

If the current user is not in one of the listed groups:

- Restricted fields are removed from views
- Restricted fields are removed from `fields_get()` responses
- Attempts to read/write restricted fields result in an access error

```python
notes = fields.Text('Internal Notes', groups='base.group_system')

# Only users in base.group_system can see/edit this field
```

Check or customize field access with `Model.has_field_access(field, operation)` /
`check_field_access(field, operation)` and override `_has_field_access` (operation
`'read'` or `'write'`). `_check_field_access` is deprecated:
`@deprecated("Since 20.0, use check_field_access()")`.

---

## Security Pitfalls

### Unsafe Public Methods

Any public method can be executed via RPC call. Methods starting with `_` are not callable from action buttons or external API.

> Use the `@api.private` decorator (since 18.2) to explicitly prevent RPC access on any method.

```python
# BAD: this method is public and arguments can not be trusted
def action_done(self):
    if self.state == "draft" and self.env.user.has_group('base.manager'):
        self._set_state("done")

# GOOD: use @api.private or underscore prefix
@api.private
def _set_state(self, new_state):
    self.sudo().write({"state": new_state})
```

### Bypassing the ORM

**Never** use the database cursor directly when the ORM can do the same thing!

#### Wrong (very bad)

```python
# SQL injection vulnerability
self.env.cr.execute('SELECT id FROM auction_lots WHERE auction_id in (' +
                    ','.join(map(str, ids)) + ') AND state=%s AND obj_price > 0',
                    ('draft',))
```

#### Better (no injection, but still wrong)

```python
self.env.cr.execute('SELECT id FROM auction_lots WHERE auction_id in %s '
                    'AND state=%s AND obj_price > 0',
                    (tuple(ids), 'draft',))
```

#### Best (use ORM)

```python
auction_lots_ids = self.search([
    ('auction_id', 'in', ids),
    ('state', '=', 'draft'),
    ('obj_price', '>', 0)
]).ids
```

#### Using SQL class (recommended)

```python
from odoo.tools import SQL

auction_lots_ids = self.env.execute_query(SQL("""
    SELECT id FROM auction_lots
    WHERE auction_id IN %s AND state = %s AND obj_price > 0
""", tuple(ids), 'draft'))
```

### SQL Injections

**Never** use Python string concatenation (+) or interpolation (%) for SQL queries.

Use psycopg2 parameter passing or `odoo.tools.SQL` wrapper.

```python
# BAD: SQL injection vulnerability
self.env.cr.execute('SELECT distinct child_id FROM account_account_consol_rel '
                    'WHERE parent_id IN ('+','.join(map(str, ids))+')')

# GOOD: use parameter passing
self.env.cr.execute('SELECT DISTINCT child_id '
                    'FROM account_account_consol_rel '
                    'WHERE parent_id IN %s',
                    (tuple(ids),))

# BETTER: use SQL wrapper
from odoo.tools import SQL
self.env.cr.execute(SQL("""
    SELECT DISTINCT child_id
    FROM account_account_consol_rel
    WHERE parent_id IN %s
""", tuple(ids)))
```

### Building Domains

Use `odoo.fields.Domain` to handle domain manipulation safely.

```python
# BAD: the user can pass ['|', ('id', '>', 0)] to access all
domain = ...  # passed by user
security_domain = [('user_id', '=', self.env.uid)]
domain += security_domain  # can have side effects
self.search(domain)

# GOOD: use Domain class
from odoo.fields import Domain
domain = Domain(...)
domain &= Domain('user_id', '=', self.env.uid)
self.search(domain)
```

### Unescaped Field Content

Never mark untrusted content as safe. `t-raw` (and `t-esc`) are no longer
compiled by server QWeb in 20.0 (only `t-out` exists); `t-out` escapes unless the
value is `Markup`.

#### Bad XML

```xml
<div t-name="insecure_template">
    <div id="information-bar"><t t-raw="info_message" /></div>  <!-- ❌ not a directive in 20 -->
</div>
```

#### Good XML

```xml
<div t-name="secure_template">
    <div class="info"><t t-out="message" /></div>
    <div class="subject"><t t-out="subject" /></div>
</div>
```

### Escaping vs Sanitizing

**Escaping** (always mandatory) converts TEXT to CODE.

```python
from markupsafe import Markup
from odoo.tools import html_escape

# data is TEXT
code = html_escape(data)  # Convert to CODE
self.website_description = Markup("<strong>%s</strong>") % code
```

**Sanitizing** converts CODE to SAFER CODE (but not necessarily safe).

```python
from odoo.tools import html_sanitize

code = Markup("<p class='text-warning'>Important</p>")
html_sanitize(code, strip_classes=True)  # => Markup('<p>Important</p>')
```

### Evaluating Content

Avoid `eval()` - use `ast.literal_eval()` for parsing.

```python
# BAD: very bad
domain = eval(self.filter_domain)

# BETTER: still not recommended
from odoo.tools.safe_eval import safe_eval
domain = safe_eval(self.filter_domain)

# GOOD: use literal_eval
from ast import literal_eval
domain = literal_eval(self.filter_domain)
```

### Accessing Object Attributes

Use `__getitem__` instead of `getattr()` for dynamic field access. `record[state_field]`
blocks arbitrary method access, but it still returns the value of *any* field on the
model. If `state_field` comes from an untrusted caller, also validate it against a
fixed allowlist and avoid `sudo()` unless the caller's access to that specific field
is already established.

```python
# BAD: unsafe, can access any attribute (including methods) and any field
def _get_state_value(self, res_id, state_field):
    record = self.sudo().browse(res_id)
    return getattr(record, state_field, False)

# GOOD: blocks method access, restricts fields to an allowlist, no unconditional sudo()
ALLOWED_STATE_FIELDS = {'state', 'stage_id'}

def _get_state_value(self, res_id, state_field):
    if state_field not in ALLOWED_STATE_FIELDS:
        raise ValueError(f"Field {state_field!r} is not allowed")
    record = self.browse(res_id)
    return record[state_field]
```

---

## References

- Source: Odoo 20 documentation `/doc/developer/reference/backend/security.rst`
- Code: `odoo/addons/base/models/ir_access.py`, `odoo/orm/models.py` (`check_access`,
  `has_access`, `_access_domain`), `odoo/orm/domains.py` (`access` operator),
  `odoo/upgrade_code/19.4-00-ir-access.py`
