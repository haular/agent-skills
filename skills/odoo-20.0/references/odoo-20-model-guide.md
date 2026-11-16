# Odoo 20 Model Guide

## Python and Odoo Conventions

- Order imports into standard/external libraries, `odoo` imports, then imports
  from Odoo addons; alphabetize within each group.
- Prefer readable built-ins, comprehensions, and recordset helpers such as
  `filtered`, `mapped`, and `sorted` over clever abstractions. Avoid custom
  generators and decorators in addon business code.
- Use singular dotted model names, CamelCase classes, and `snake_case`
  variables. Reserve `_id`/`_ids` for integer IDs rather than recordsets.
- Order declarations as private attributes, defaults, fields, compute/inverse/
  search methods, selection methods, constraints/onchanges, CRUD overrides,
  actions, then business methods.
- Make overrides extendable: call `super()`, do not assume a singleton without
  `ensure_one()`, and never mutate context directly; use `with_context()` for a
  derived environment.
- Context is immutable and propagated automatically. Prefix custom behavior
  keys with the module name; avoid generic `default_<field>` keys because they
  can leak into nested creates on unrelated models.

Guide for working with Odoo 20 ORM, recordsets, CRUD operations, and domain filters.
Odoo 20 requires Python >= 3.12 (`odoo/release.py` `MIN_PY_VERSION`).

## Table of Contents

- [Models](#models)
- [Fields](#fields)
- [SQL Constraints](#sql-constraints)
- [Database Indexes](#database-indexes)
- [Recordsets](#recordsets)
- [CRUD Operations](#crud-operations)
- [Search Domains](#search-domains)
- [Environment](#environment)
- [Configuration Parameters](#configuration-parameters)
- [ORM Cache](#orm-cache)
- [SQL Execution](#sql-execution)
- [Inheritance](#inheritance)

---

## Models

### Defining a Model

> **Since 19.0**: when `_name` is missing, Odoo derives it from the CamelCase class name (each capital letter → `.` separator, e.g. `ResPartner` → `res.partner`, `SaleOrder` → `sale.order`) and logs the warning `Class ... has no _name, please make it explicit`. Keep `_name` explicit on new models. A string `_inherit` without `_name` still sets `_name = _inherit`; a list `_inherit` does not, so `class Partner` with `_inherit = ['res.partner']` and no `_name` creates a new model `partner`.

```python
from odoo import models, fields

class MyModel(models.Model):
    _name = 'my.model'  # omitting it works (derived from the class name) but logs a warning
    _description = 'My Model'

    field1 = fields.Char()
    field2 = fields.Integer(string="Field Label")
```

```python
# When _name differs from class name convention, specify explicitly:
class CustomNameModel(models.Model):
    _name = 'custom.different.name'
    _description = 'Custom Named Model'

    name = fields.Char()
```

### Model Attributes

| Attribute                | Description                                                                         |
| ------------------------ | ----------------------------------------------------------------------------------- |
| `_name`                  | Model name (derived from the CamelCase class name with a warning if missing)        |
| `_description`           | Model description                                                                   |
| `_explanation`           | Verbose explanation of the model's purpose (new in 20, used by stock/project)       |
| `_order`                 | Default sort order                                                                  |
| `_rec_name`              | Field to use as name representation                                                |
| `_rec_names_search`      | Fields searched by `name_search`/`display_name` — declare as a **tuple** (19.5)     |
| `_inherit`               | Model(s) to inherit from                                                            |
| `_inherits`              | Delegation inheritance                                                              |
| `_table`                 | Database table name (cannot be redefined when extending, see Inheritance)           |
| `_table_sql`             | Property returning the `SQL` table/subquery (replaces `_table_query`, removed 20.0) |
| `_log_access`            | Enable create_date, write_date, create_uid, write_uid                               |
| `_auto`                  | Auto-create database table                                                          |
| `_abstract`              | Abstract model                                                                      |
| `_transient`             | Transient model                                                                     |
| `_parent_store`          | Enable parent_path field                                                            |
| `_fold_name`             | Field for kanban fold                                                               |
| `_clear_cache_name`      | ormcache name invalidated on create/write/unlink (e.g. `'default'`, `'stable'`)     |
| `_clear_cache_on_fields` | Fields whose write triggers that invalidation (`None` = all fields)                 |

```python
# ✅ Odoo 20 (19.5 upgrade script 19.5-00-tuple-rec_names_search.py rewrites lists)
_rec_names_search = ('name', 'model')
# ❌ list form, rewritten by the upgrade tool
_rec_names_search = ['name', 'model']
```

### Model Types

| Class                   | Description                                                   |
| ----------------------- | ------------------------------------------------------------- |
| `models.Model`          | Regular database model                                        |
| `models.TransientModel` | Temporary/wizard model                                        |
| `models.AbstractModel`  | Abstract model (no database table)                            |
| `models.CachedModel`    | Python-only base (20) caching `_cached_data_fields` per model |

### SQL-View Models: `_table_query` Removed (20.0)

`Model._table_query` no longer exists. Report/SQL-view models override the `_table_sql`
property and return an `SQL` object (`odoo/orm/models.py` `_table_sql`).

```python
# ❌ Odoo 19
@property
def _table_query(self):
    return SQL("SELECT ... FROM sale_order_line ...")

# ✅ Odoo 20 (addons/sale/report/sale_report.py)
@property
def _table_sql(self) -> SQL:
    query = self.env['sale.order.line'].sudo()._search(self._order_line_domain())
    query.groupby = SQL(", ").join(self._groupby_list(query.table))
    return query.subselect(*self._select_dict_to_list(self._select_dict(query.table)))

# Table-less model (odoo/addons/base/models/res_users.py)
_auto = False
_table_sql = SQL('(0)')
```

---

## Fields

### Field Definition

Fields are defined as class attributes on the model. See the field guide for details.

```python
from odoo import models, fields

class MyModel(models.Model):
    _name = 'my.model'

    name = fields.Char(required=True)
    description = fields.Text()
    active = fields.Boolean(default=True)
    count = fields.Integer()
    price = fields.Float(digits='Product Price')
```

### Default Values

```python
# Value
name = fields.Char(default="A value")

# Function
def _default_name(self):
    return self.get_value()

name = fields.Char(default=lambda self: self._default_name())
```

### Field Types

| Type              | Class                        | Description                              |
| ----------------- | ---------------------------- | ---------------------------------------- |
| **Basic**         |                              |                                          |
| Boolean           | `fields.Boolean()`           | True/False                               |
| Char              | `fields.Char()`              | String (limited length)                  |
| Float             | `fields.Float()`             | Floating-point number                    |
| Integer           | `fields.Integer()`           | Integer                                  |
| **Advanced**      |                              |                                          |
| Binary            | `fields.Binary()`            | Binary data; value is a `BinaryValue`    |
| Html              | `fields.Html()`              | HTML content                             |
| Image             | `fields.Image()`             | Image (enhanced Binary)                  |
| Json              | `fields.Json()`              | JSON value                               |
| Monetary          | `fields.Monetary()`          | Monetary amount                          |
| Selection         | `fields.Selection()`         | Selection from list                      |
| Text              | `fields.Text()`              | Long text                                |
| **Date**          |                              |                                          |
| Date              | `fields.Date()`              | Date (no time)                           |
| Datetime          | `fields.Datetime()`          | Date and time                            |
| **Relational**    |                              |                                          |
| Many2one          | `fields.Many2one()`          | Many-to-one                              |
| One2many          | `fields.One2many()`          | One-to-many                              |
| Many2many         | `fields.Many2many()`         | Many-to-many                             |
| **Pseudo**        |                              |                                          |
| Reference         | `fields.Reference()`         | Reference to any model                   |
| Many2oneReference | `fields.Many2oneReference()` | Many2one with dynamic model              |
| **Properties**    |                              |                                          |
| Properties        | `fields.Properties()`        | Dynamic properties                       |

### Computed Fields

```python
from odoo import api

total = fields.Float(compute='_compute_total', store=True)

@api.depends('value', 'tax')
def _compute_total(self):
    for record in self:
        record.total = record.value + record.value * record.tax
```

Non-stored computed fields can be made searchable, sortable and groupable with
`compute_sql=` (19.1) — see the field guide.

### Related Fields

```python
nickname = fields.Char(related='partner_id.name', store=True)
```

### Automatic Fields

| Field          | Type     | Description           |
| -------------- | -------- | --------------------- |
| `id`           | int      | Identifier            |
| `display_name` | char     | Display name          |
| `create_date`  | datetime | Creation timestamp    |
| `create_uid`   | Many2one | Creator               |
| `write_date`   | datetime | Last update timestamp |
| `write_uid`    | Many2one | Last modifier         |

### Reserved Field Names

| Name          | Type      | Purpose                                         |
| ------------- | --------- | ----------------------------------------------- |
| `name`        | Char      | Default `rec_name`; copied with " (copy)" (20)  |
| `active`      | Boolean   | Toggles global visibility                       |
| `state`       | Selection | Lifecycle stages; `copy=False` by default       |
| `parent_id`   | Many2one  | Tree structure parent                           |
| `parent_path` | Char      | Tree structure path                             |
| `company_id`  | Many2one  | Multi-company field                             |

---

## SQL Constraints

> `_sql_constraints` is **no longer supported** (since 19.0): the model only logs
> `Model attribute '_sql_constraints' is no longer supported` (`odoo/orm/model_classes.py`).
> Use `models.Constraint`.

### models.Constraint

```python
from odoo import models, fields

class MyModel(models.Model):
    _name = 'my.model'
    _description = 'My Model'

    name = fields.Char(required=True)
    code = fields.Char()
    quantity = fields.Integer()

    # UNIQUE constraint
    _unique_name = models.Constraint(
        'UNIQUE(name)',
        'Name must be unique!',
    )

    # UNIQUE on multiple fields
    _unique_name_code = models.Constraint(
        'UNIQUE(name, code)',
        'The combination of name and code must be unique!',
    )

    # CHECK constraint
    _check_quantity = models.Constraint(
        'CHECK(quantity > 0)',
        'Quantity must be positive!',
    )
```

### Constraint Naming

The attribute name must start with `_`; the SQL name is `<table>_<attribute name without the leading _>`
(`_unique_name` on `my_model` → `my_model_unique_name`). There is no separate name argument.
For a unique index with a message, use `models.UniqueIndex('(name)', 'Name must be unique!')`.

### Migration from `_sql_constraints`

```python
# ❌ OLD (Odoo 18 and earlier) — ignored since 19.0
class MyModel(models.Model):
    _name = 'my.model'
    _sql_constraints = [
        ('name_uniq', 'UNIQUE(name)', 'Name must be unique!'),
        ('check_qty', 'CHECK(quantity > 0)', 'Quantity must be positive!'),
    ]

# ✅ NEW
class MyModel(models.Model):
    _name = 'my.model'

    _name_uniq = models.Constraint(
        'UNIQUE(name)',
        'Name must be unique!',
    )

    _check_qty = models.Constraint(
        'CHECK(quantity > 0)',
        'Quantity must be positive!',
    )
```

### Overriding Constraints in Inherited Models

```python
class ExtendedModel(models.Model):
    _inherit = 'my.model'

    # Override constraint with different check
    _check_qty = models.Constraint(
        'CHECK(quantity >= 0)',
        'Quantity cannot be negative!',
    )
```

---

## Database Indexes

### Field-Level Index

```python
name = fields.Char(index=True)  # btree; also 'btree_not_null', 'trigram'
```

### Declarative Index

For composite or custom indexes, use `models.Index`:

```python
class MyModel(models.Model):
    _name = 'my.model'

    name = fields.Char()
    code = fields.Char()
    date = fields.Date()

    # Composite index on multiple fields
    _name_code_idx = models.Index('(name, code)')

    # Index with specific ordering
    _date_idx = models.Index('(date DESC)')
```

> **Warning**: Don't over-index — indexes consume space and impact INSERT/UPDATE/DELETE performance.

---

## Recordsets

### Active Record Interface

```python
# Read field
record.name
record.company_id.name

# Write field
record.name = "Bob"

# Dynamic field access
field = "name"
record[field]
```

### Iteration

```python
def do_operation(self):
    for record in self:
        # record is a single record
        print(record.name)
```

### Record Cache and Prefetching

Odoo caches values and prefetches fields for all records of the recordset being iterated:
the loop below reads `name` and `lang` for all partners in one query, not one per record.

```python
for partner in partners:
    print(partner.name)
    print(partner.lang)
```

### x2many Values Contain Only Accessible Records (19.3)

Reading a One2many/Many2many returns only the comodel records the current user can read
(`_RelationalMulti.convert_to_record` filters with `_filtered_access('read')`, unless the field
has `bypass_search_access=True` or the env is sudo). Do not assume
`len(order.line_ids)` equals the database count for a restricted user; use `sudo()` explicitly
when you need all lines.

### Combining Recordsets: `concat` / `union` (19.3)

Call them on an **empty** recordset with **one iterable**. Other forms still work but warn
`"Since 20.0, use concat() with only one iterable"` / `"Since 20.0, use concat() on an empty recordset"`
(same for `union`).

```python
# ❌ deprecated forms
records = rec1.concat(rec2, rec3)
records = rec1.union(rec2)

# ✅ Odoo 20 (addons/stock/models/stock_package.py, stock_location.py)
quants = self.env['stock.quant'].concat(group)          # keeps duplicates
products = Product.union(p for p in product_list)       # deduplicated, order kept
records = rec1 | rec2                                   # operators unchanged
```

---

## CRUD Operations

### Create

```python
# Single record
record = self.env['model.name'].create({'field': 'value'})

# Multiple records (batch)
records = self.env['model.name'].create([
    {'field': 'value1'},
    {'field': 'value2'},
])
```

Field values and inverses are applied in `(field.write_sequence, field index)` order in both
`create()` and `write()` (19.4). x2many fields have `write_sequence = 20`, Monetary and
Properties `10`, others `0`; set `write_sequence=` on a field to force an earlier/later inverse
(e.g. `stock.move.quantity` uses `write_sequence=25`).

### Read

```python
# Browse
record = self.env['model.name'].browse(record_id)
records = self.env['model.name'].browse([id1, id2, id3])

# Read
data = records.read(['field1', 'field2'])
```

### Write

```python
# Single record
record.write({'field': 'value'})

# Multiple records
records.write({'field': 'value'})
```

### Copy

`copy_data()` uses each field's `copy` attribute; a callable `copy` produces the copied value.
A Char field named `name` (or `x_name`) defaults to `copy=mark_as_copy('name')`, appending
" (copy)" in every stored language (20.0). Remove `copy()`/`copy_data()` overrides that only
add " (copy)" to the name.

### Unlink (Delete)

```python
# Single record
record.unlink()

# Multiple records
records.unlink()
```

### Archiving

`toggle_active()` was removed in 20; use `action_archive()` / `action_unarchive()`.
`_check_recursion()` / `_check_m2m_recursion()` were removed; use `_has_cycle(field_name=None)`.

---

## Search Domains

A search domain is a first-order logical predicate for filtering.

### Domain Condition

```python
# Simple condition
domain = [('name', '=', 'ABC')]

# Multiple conditions
domain = [('name', '=', 'ABC'), ('phone', 'like', '7620')]
```

### Operators

| Operator                             | Description                                     |
| ------------------------------------ | ----------------------------------------------- |
| `=`                                  | equals                                          |
| `!=`                                 | not equals                                      |
| `>`, `>=`, `<`, `<=`                 | comparison                                      |
| `=?`                                 | unset or equals                                 |
| `=like`, `like`, `ilike`, `=ilike`   | pattern matching (and `not ...` variants)       |
| `in`, `not in`                       | in list                                         |
| `child_of`, `parent_of`              | tree traversal                                  |
| `any`, `any!`, `not any`, `not any!` | relation traversal (`!` = bypass access rules)  |
| `access`                             | record/comodel access check (19.3), see below   |

> **Removed in 20**: the `'<>'` and `'=='` operators (deprecated in 19) and upper-case
> operators (`'IN'`, `'ILIKE'`): 19 lower-cased them with a warning, 20 raises `Invalid operator`.
> Use `'!='`, `'='` and lower-case operators.

### `access` Operator (19.3)

`(field, 'access', operation)` keeps records whose many2one target (or the record itself
with `id`) is accessible for `'read'`, `'write'`, `'create'` or `'unlink'`. It works only on
many2one and `id` fields (`odoo/orm/domains.py` `_operator_access_rule_domain`). Mostly used in
`ir.access` domains:

```python
# addons/purchase/security/ir.access.csv
[('move_id', 'access', 'read')]

# in code
lines = self.env['account.move.line'].search([('move_id', 'access', 'write')])
```

### Logical Operators

```python
# AND (implicit)
domain = [('name', '=', 'ABC'), ('state', '=', 'draft')]

# OR (prefix notation)
domain = ['|', ('name', '=', 'ABC'), ('name', '=', 'XYZ')]

# NOT
domain = ['!', ('state', '=', 'draft')]
```

### Domain Class

`odoo.osv` was removed in 20 (deprecated in 19): `from odoo.osv import expression` fails.
Use `Domain` (`expression.AND/OR` → `Domain.AND/OR`).

```python
from odoo.fields import Domain

# Create domain
d1 = Domain('name', '=', 'abc')
d2 = Domain('phone', 'like', '7620')

# Combine
d3 = d1 & d2  # AND
d4 = d1 | d2  # OR
d5 = ~d1      # NOT
d6 = Domain.AND([d1, d2, Domain.TRUE])

# Parse from list
domain = Domain([('name', '=', 'abc'), ('phone', 'like', '7620')])

# Serialize to list
domain_list = list(domain)

# Inspect a simple condition (new in 20)
d1.is_condition('name', ('=', 'in'))
```

### Search Methods

```python
# Search
records = self.env['model'].search(domain)

# Search with limit / offset / order
records = self.env['model'].search(domain, limit=10, offset=20, order='name ASC')

# Search count
count = self.env['model'].search_count(domain)

# Search and read
records = self.env['model'].search_read(domain, ['field1', 'field2'])

# Search and fetch fields into the cache
records = self.env['model'].search_fetch(domain, ['field1', 'field2'])

# Name search
results = self.env['model'].name_search('keyword', operator='ilike')

# Grouping: backend
groups = self.env['model']._read_group(domain, ['partner_id'], ['amount:sum'])
```

> **`read_group` changed in 20**: the old `read_group(domain, fields, groupby, ..., lazy=True)`
> (deprecated in 19) is gone. `read_group(domain, groupby=(), aggregates=(), having=(), offset, limit, order)`
> is now a final public method returning `_read_group` tuples with records turned into ids.
> Use `_read_group()` in backend code and `formatted_read_group()` (web) for formatted dicts.

---

## Environment

The environment holds:

- Database cursor (`cr`)
- Current user (`user`, `uid`)
- Context (`context`)
- Record cache
- Transaction (`env.transaction`, holds the ormcache layers in 20)

### Accessing Environment

```python
# From recordset
env = record.env

# Create new recordset in another model
model = env['another.model']

# Access properties
env.uid         # Current user id
env.user        # Current user recordset
env.company     # Current company
env.companies   # Allowed companies
env.lang        # Current language
```

`record._cr`, `record._uid`, `record._context` still work but warn
`"Deprecated since 19.0, use self.env.cr directly"` (resp. `env.uid`, `env.context`).

### Altering Environment

Each of these returns a **new** recordset bound to the modified environment; the
original recordset is unchanged. Always use the return value.

```python
records_fr = records.with_context(lang='fr_FR')
records_as_user = records.with_user(user_id)
records_for_company = records.with_company(company_id)
records_new_env = records.with_env(new_env)
records_sudo = records.sudo()
```

---

## Configuration Parameters

The `ir.config_parameter` API is typed since 19.1: `get_param()`/`set_param()` no longer exist.

```python
ICP = self.env['ir.config_parameter'].sudo()

# ❌ Odoo 19
ICP.get_param('reveal.view_weeks_valid', 4)
ICP.set_param('my_module.enabled', True)

# ✅ Odoo 20
weeks = ICP.get_int('reveal.view_weeks_valid') or 4     # default 0
enabled = ICP.get_bool('my_module.enabled')             # default False
endpoint = ICP.get_str('reveal.endpoint', 'https://')   # default ''
rate = ICP.get_float('my_module.rate')                  # default 0.0
ICP.set_bool('my_module.enabled', True)                 # also set_int/set_float/set_str
```

`set_*()` returns the previous value; passing `None` (or `False` for int/float/str) clears the
value. An unparsable stored value logs a warning and returns the default.

---

## ORM Cache

The ormcache is thread-safe and transactional since 19.4 (`odoo/orm/cache.py`).

```python
from odoo import api, models

class MyModel(models.Model):
    _name = 'my.model'
    _clear_cache_name = 'default'            # invalidated on create/write/unlink
    _clear_cache_on_fields = {'active', 'code'}

    @api.ormcache('code', cache='default')   # decorator now in odoo.api
    def _get_id_by_code(self, code):
        return self.search([('code', '=', code)], limit=1).id   # never return records

    def _reset(self):
        self.env.transaction.invalidate_ormcache('default')
```

| ❌ Odoo 19                                       | ✅ Odoo 20                                          |
| ----------------------------------------------- | -------------------------------------------------- |
| `from odoo.tools import ormcache` (warns)       | `@api.ormcache(...)`                               |
| `self.env.registry.clear_cache('default')`      | `self.env.transaction.invalidate_ormcache('default')` |
| `tools.ormcache_context(...)`                   | not in 20; put context values in the key           |

`from odoo.tools import ormcache` still works but warns `"Since 20.0 import ormcache from odoo.api"`.
`registry.clear_cache` was removed (upgrade script `19.4-00-ormcache-on-transaction.py`).
Cache names: `'default'`, `'assets'`, `'stable'`, `'templates.cached_values'`, …
(`odoo/orm/registry.py` `_CACHES_BY_KEY`).

---

## SQL Execution

### SQL Class and `execute_query`

```python
from odoo.tools import SQL

query = SQL("SELECT id FROM %s WHERE state = %s", SQL.identifier(self._table), 'done')
rows = self.env.execute_query(query)          # flushes fields in SQL metadata, returns tuples
dicts = self.env.execute_query_dict(query)    # list of dicts

# Raw cursor still works
self.env.cr.execute(query)
```

### Query Builder (19.1)

`Query` is exported by `odoo.models` (it moved out of `odoo.tools` to `odoo/orm/query.py`).
`query.table` is a `TableSQL`: attribute access gives the field's SQL, follows many2one joins,
checks field access and records fields to flush.

```python
from odoo.models import Query
from odoo.tools import SQL

query = Query(self.env['mailing.trace'].sudo())
t = query.table
query.add_where(SQL("%s IS NOT NULL", t.email))
query.add_where(SQL("%s = %s", t.mass_mailing_id, self.id))
emails = {row[0] for row in self.env.execute_query(query.select(t.email))}

# subqueries in domains (addons/sale/models/product_product.py)
subquery = self.env['sale.order.line']._search([...])
return [('id', operator, subquery.subselect(subquery.table.product_id))]
```

| ❌ Odoo 19                                         | ✅ Odoo 20                                  |
| ------------------------------------------------- | ------------------------------------------ |
| `from odoo.tools import Query`                    | `from odoo.models import Query`            |
| `self._field_to_sql(alias, 'name', query)`        | `query.table.name` (`_field_to_sql` kept)  |
| `query.select('id')` / `query.order = 'id'` (str) | pass `SQL` objects (str warns "Since 20.0") |
| `_order_to_sql(order, query, alias)`              | `_order_to_sql(table, order, reverse=False)` |
| `_read_group_groupby(alias, spec, query)`         | `_read_group_groupby(table, spec)` (same for `_select`/`_having`/`_orderby`) |

### Flush and Invalidate

`execute_query` flushes automatically from SQL metadata. With `cr.execute`, flush first:

```python
self.env['model'].flush_model(['field1', 'field2'])
records.flush_recordset(['field1', 'field2'])
```

After SQL modifications, invalidate cache:

```python
self.env['model'].invalidate_model(['field1', 'field2'])
records.invalidate_recordset(['field1', 'field2'])
records.modified(['field1', 'field2'])
```

---

## Inheritance

### Classical Inheritance

Create new model from existing one:

```python
class Inheritance1(models.Model):
    _name = 'inheritance.1'
    _description = 'Inheritance One'

    name = fields.Char()

class Inheritance2(models.Model):
    _name = 'inheritance.2'
    _inherit = ['inheritance.1']
    _description = 'Inheritance Two'
```

### Extension

Extend existing model in-place:

```python
class Extension0(models.Model):
    _name = 'extension.0'
    _description = 'Extension zero'

    name = fields.Char(default="A")

class Extension0(models.Model):
    _inherit = 'extension.0'

    description = fields.Char(default="Extended")
```

> **New in 20**: an extension class may not set `_table` — it raises
> `TypeError: ... cannot redefine _table while extending model ...` (`odoo/orm/model_classes.py`).
> Remove `_table` or set a different `_name`.

### Delegation

```python
class Screen(models.Model):
    _name = 'delegation.screen'

    size = fields.Float(string='Screen Size')

class Laptop(models.Model):
    _name = 'delegation.laptop'

    _inherits = {
        'delegation.screen': 'screen_id',
    }

    name = fields.Char(string='Name')
    screen_id = fields.Many2one('delegation.screen', required=True, ondelete="cascade")

# Can access size directly on laptop
laptop.size
```

### Removed / Deprecated Model APIs in 20

| Removed or deprecated                                 | Use instead                                        |
| ----------------------------------------------------- | -------------------------------------------------- |
| `check_access_rights()`, `check_access_rule()`, `_filter_access_rules()` (removed) | `check_access()`, `has_access()`, `_filtered_access()` |
| `_check_access()` (warns "Since 20.0")                | `_access_domain(operation)`                        |
| `_check_field_access()` (warns "Since 20.0")          | `check_field_access(field, operation)` / `has_field_access()` |
| `_init_column()` (warns "Since 20.0")                 | field `init_storage=` (see field guide)            |
| `copy_translations()` (removed)                       | field `copy` callables / `mark_as_copy`            |

---

## References

- Source: Odoo 20 `odoo/orm/models.py`, `odoo/orm/domains.py`, `odoo/orm/query.py`, `odoo/orm/cache.py`, `odoo/orm/model_classes.py`, `odoo/addons/base/models/ir_config_parameter.py`
- Docs: `content/developer/reference/backend/orm.rst` and `orm/changelog.rst` (20.0 branch)
