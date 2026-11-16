# Odoo 20 Field Guide

## Coding Conventions

- Keep fields and their default, compute, inverse, and search helpers together.
- Declare related fields near their source field and use explicit labels only
  when the generated label is not suitable; keep help text user-facing.
- Use `aggregator=` for list/graph aggregation and avoid storing a computed
  field only to search, sort or group on it: prefer `compute_sql=` (19.1).

Guide for defining fields in Odoo 20: field types, parameters, computed fields, and relational fields.

## Table of Contents
- [Field Types](#field-types)
- [Basic Fields](#basic-fields)
- [Advanced Fields](#advanced-fields)
- [Binary Fields](#binary-fields)
- [Date Fields](#date-fields)
- [Relational Fields](#relational-fields)
- [Computed Fields](#computed-fields)
- [Related Fields](#related-fields)
- [Field Parameters](#field-parameters)

---

## Field Types

### Type Summary

| Category | Types |
|----------|-------|
| **Basic** | Boolean, Char, Float, Integer |
| **Advanced** | Binary, Html, Image, Json, Monetary, Selection, Text |
| **Date** | Date, Datetime |
| **Relational** | Many2one, One2many, Many2many |
| **Pseudo** | Reference, Many2oneReference |
| **Properties** | Properties, PropertiesDefinition |

All are exported by `odoo.fields` (`odoo/fields/__init__.py`), together with `Command` and `Domain`.

---

## Basic Fields

### Boolean

```python
active = fields.Boolean(string="Active", default=True)
is_company = fields.Boolean("Is Company")
```

### Char

String with limited length.

```python
name = fields.Char(string="Name", required=True)
code = fields.Char("Code", size=10)
```

A Char field named `name` (or `x_name`) without explicit `copy` is copied with a
" (copy)" suffix (20.0), see [copy](#copy).

### Float

```python
price = fields.Float(string="Price")
weight = fields.Float(digits="Stock Weight")
```

#### Digits

```python
# Using named precision
price = fields.Float(digits="Product Price")

# Custom precision (12 digits, 2 decimal)
amount = fields.Float(digits=(12, 2))
```

### Integer

```python
count = fields.Integer(string="Count")
priority = fields.Integer(default=10)
```

---

## Advanced Fields

### Html

```python
description = fields.Html(string="Description")
notes = fields.Html("Notes", sanitize=False)
```

### Image

Binary subclass for images; resized to `max_width`/`max_height`. Requires `_log_access = True`
on the model (warning otherwise).

```python
image = fields.Image(string="Image")
logo = fields.Image("Logo", max_width=1024, max_height=1024)
```

### Monetary

```python
amount = fields.Monetary(string="Amount", currency_field="currency_id")
```

**Requires** a currency field (default: `currency_id`).

### Selection

```python
state = fields.Selection([
    ('draft', 'Draft'),
    ('confirmed', 'Confirmed'),
    ('done', 'Done'),
], string="State", default='draft')
```

Extend in an inheriting model with `selection_add=[...]` and `ondelete={...}`.
For a selection driven by another model, use a Many2one instead.

### Text

```python
description = fields.Text(string="Description")
```

---

## Binary Fields

> **Breaking (19.2 → 20.0)**: Binary/Image values are **`BinaryValue`** objects, not base64
> strings, and raw bytes are stored. Stop calling `base64.b64encode/b64decode` around field
> values (`odoo/tools/binary.py`, `odoo/orm/fields_binary.py`).

```python
file = fields.Binary(string="File")                    # stored as ir.attachment
db_file = fields.Binary("File", attachment=False)      # bytea column
```

Reading: `record.file` is always a `BinaryValue` (empty `BinaryBytes(b'')` when unset, falsy).

| Member | Meaning |
|--------|---------|
| `.content` | raw `bytes` (also `bytes(value)`) |
| `.size` | length in bytes |
| `.filename` | file name, `''` if none (20.0) |
| `.mimetype` | guessed mimetype |
| `.checksum` | sha256 hex |
| `.to_base64()` | base64 `str` |
| `.open()` | `io.BytesIO` stream |

Writing accepts: a `BinaryValue` (general case), a base64 `str` (RPC compatibility), a dict
`{'content': ..., 'filename': ...}` (only `filename` to rename a single record's file), or
raw `bytes` **only** on `ir.attachment.raw`. Other `bytes` raise
`TypeError: ... use BinaryValue instead of bytes`.

```python
from odoo.tools import BinaryBytes

# ❌ Odoo 19
record.file = base64.b64encode(pdf_bytes)
data = base64.b64decode(record.file)

# ✅ Odoo 20
record.file = BinaryBytes(pdf_bytes, filename='report.pdf')
data = record.file.content
payload = {'logo': company.logo.to_base64()}          # addons/website/models/website.py
self.env['ir.attachment'].create({'name': 'x.pdf', 'raw': BinaryBytes(content)})  # mail_thread.py
```

RPC format (20.0): `read()` returns `{'content': <base64>, 'size': <int>, 'filename': <str>}`
(`filename` omitted when empty or equal to the field name) or `False`. Domains on binary
fields support only existence checks (`('file', '!=', False)`); `'file.size'` and
`'file.filename'` are usable as field expressions.

XML data: use `type="bytes"` for `file=` loading (upgrade script `19.3-00-base64-in-xml.py`
rewrites `type="base64" file=...`).

---

## Date Fields

### Date

```python
date = fields.Date(string="Date")
deadline = fields.Date(default=fields.Date.context_today)
```

#### Date Methods

```python
from odoo.fields import Date

today = Date.context_today(self)
next_week = Date.add(Date.today(), weeks=1)
last_month = Date.subtract(Date.today(), months=1)
start_of_month = Date.start_of(Date.today(), 'month')
end_of_month = Date.end_of(Date.today(), 'month')
date_str = Date.to_string(Date.today())
date_obj = Date.to_date('2023-01-01')
```

### Datetime

```python
datetime = fields.Datetime(string="DateTime")
create_date = fields.Datetime(default=fields.Datetime.now)
```

#### Datetime Methods

```python
from odoo.fields import Datetime

now = Datetime.now()
timestamp = Datetime.context_timestamp(self, datetime)   # user timezone
next_hour = Datetime.add(Datetime.now(), hours=1)
start_of_day = Datetime.start_of(Datetime.now(), 'day')
end_of_day = Datetime.end_of(Datetime.now(), 'day')
date_obj = Datetime.to_datetime('2023-01-01 12:00:00')
date_str = Datetime.to_string(Datetime.now())
```

#### Timezone

Datetime fields are stored as naive UTC. Conversion is client-side or via `context_timestamp`.

---

## Relational Fields

### Many2one

```python
partner_id = fields.Many2one('res.partner', string="Partner")
user_id = fields.Many2one('res.users', 'User', default=lambda self: self.env.user)
```

#### Parameters

| Parameter | Description |
|-----------|-------------|
| `comodel_name` | Related model name |
| `string` | Field label |
| `required` | Whether required |
| `ondelete` | What to do when related record is deleted (`cascade`, `set null`, `restrict`) |
| `domain` | Domain filter |
| `context` | Context for operations |
| `default` | Default value |
| `index` | Add database index |
| `check_company` | Restrict to records of a compatible company; enforced on create/write when the model sets `_check_company_auto = True` |
| `bypass_search_access` | Skip comodel access checks when searching/reading through the field |

```python
partner_id = fields.Many2one(
    'res.partner',
    string="Customer",
    required=True,
    ondelete='cascade',
    domain=[('customer_rank', '>', 0)],
    default=lambda self: self.env.user.partner_id,
)
```

### One2many

```python
line_ids = fields.One2many('sale.order.line', 'order_id', string="Order Lines")
```

| Parameter | Description |
|-----------|-------------|
| `comodel_name` | Related model name |
| `inverse_name` | Inverse Many2one field |
| `string` | Field label |

### Many2many

```python
tag_ids = fields.Many2many('crm.tag', string="Tags")
product_ids = fields.Many2many('product.product', 'my_rel', 'my_id', 'product_id', string="Products")
```

| Parameter | Description |
|-----------|-------------|
| `comodel_name` | Related model name |
| `relation` | Relation table name (auto if not specified) |
| `column1` | Column for this model |
| `column2` | Column for related model |
| `string` | Field label |

#### x2many Values Are Access-Filtered (19.3)

Reading a One2many/Many2many returns only the comodel records readable by the current user
(unless sudo or `bypass_search_access=True`), so cached values no longer leak inaccessible
records. Use `sudo()` when business logic needs every line. x2many fields are written last
(`write_sequence = 20`).

### Commands

```python
from odoo.fields import Command

{'line_ids': [Command.create({'product_id': 1, 'qty': 10})]}   # create
{'line_ids': [Command.update(line_id, {'qty': 20})]}           # update
{'line_ids': [Command.delete(line_id)]}                        # delete record
{'tag_ids': [Command.unlink(tag_id)]}                          # remove link only
{'tag_ids': [Command.link(tag_id)]}                            # add link
{'line_ids': [Command.clear()]}                                # remove all links
{'tag_ids': [Command.set([id1, id2, id3])]}                    # replace all links
```

---

## Computed Fields

### Basic Compute

```python
total = fields.Float(compute='_compute_total')

@api.depends('price', 'qty')
def _compute_total(self):
    for record in self:
        record.total = record.price * record.qty
```

### Store and Search

```python
total = fields.Float(
    compute='_compute_total',
    store=True,
    search='_search_total',
)
```

### `compute_sql` (19.1)

A non-stored computed field can give its SQL expression; it then becomes searchable,
sortable and groupable without `store=True`. The method receives a `TableSQL` (`table.<field>`
is the field's SQL, many2one paths join automatically) and returns `SQL`. `compute_sudo` must
be explicit (warning `compute_sql requires an explicit compute_sudo parameter`), and
`compute_sql` without `compute` warns. Related non-stored fields get `compute_sql` automatically
when the path is SQL-representable.

```python
from odoo.tools import SQL

# addons/product/models/product_template.py
currency_id = fields.Many2one(
    'res.currency', 'Currency',
    compute='_compute_currency_id',
    compute_sql='_compute_sql_currency_id',
    compute_sudo=True,
)

def _compute_sql_currency_id(self, table):
    main_company = self.env['res.company']._get_main_company()
    return SQL("COALESCE(%s, %s)", table.company_id.currency_id, main_company.currency_id.id)

# addons/stock/models/stock_picking_type.py
def _compute_sql_is_favorite(self, table):
    return SQL(
        "%s IN (SELECT picking_type_id FROM picking_type_favorite_user_rel WHERE user_id = %s)",
        table.id, self.env.uid,
    )
```

### Inverse

```python
full_name = fields.Char(
    compute='_compute_full_name',
    inverse='_inverse_full_name',
)

@api.depends('first_name', 'last_name')
def _compute_full_name(self):
    for record in self:
        record.full_name = f"{record.first_name} {record.last_name}"

def _inverse_full_name(self):
    for record in self:
        parts = record.full_name.split(' ', 1)
        record.first_name = parts[0]
        record.last_name = parts[1] if len(parts) > 1 else ''
```

Inverses (and field updates) run in `(write_sequence, field index)` order in both `create()`
and `write()` (19.4). Set `write_sequence=` to force ordering:

```python
# addons/stock/models/stock_move.py
quantity = fields.Float(compute='_compute_quantity', inverse='_set_quantity', store=True,
                        write_sequence=25)  # after lot_ids changes
```

---

## Related Fields

```python
partner_name = fields.Char(related='partner_id.name', string="Partner Name")
partner_email = fields.Char(related='partner_id.email', readonly=True)
```

### Store Related

```python
partner_name = fields.Char(related='partner_id.name', store=True)
```

### Dependencies

```python
# Only recompute when partner_id changes
partner_name = fields.Char(
    related='partner_id.name',
    store=True,
    depends=['partner_id'],
)
```

---

## Field Parameters

### Common Parameters

| Parameter | Description |
|-----------|-------------|
| `string` | Field label |
| `required` | Whether required (create/write) |
| `readonly` | Whether read-only (UI only) |
| `index` | `True`/`'btree'`, `'btree_not_null'`, `'trigram'` |
| `default` | Default value or callable |
| `help` | Tooltip text |
| `groups` | Comma-separated group xml ids |
| `copy` | `bool` or `(record) -> value` callable (20.0), see below |
| `aggregator` | Group-by aggregate (`sum`, `avg`, `min`, `max`, `count`, `count_distinct`, `bool_and`, `bool_or`) |
| `compute_sql` | Method returning the field's SQL (19.1) |
| `write_sequence` | Order of update/inverse in create/write (19.4) |
| `init_storage` | Method initializing the new column for existing rows (20.0) |
| `tracking` | Track changes in chatter (`mail.thread` models) |

`track_visibility` and `group_operator` do not exist: use `tracking=True` and `aggregator=`.

### copy

`copy` may be a callable `(record) -> value` used by `copy_data()` (not called when the field is
in `default`). Defaults: `True` for normal fields; `False` for one2many, computed, related and
`state` fields; `mark_as_copy('name')` for a Char `name`/`x_name` field — it appends
" (copy)" in every stored translation.

```python
from odoo.tools.translate import mark_as_copy

# addons/survey/models/survey_survey.py
title = fields.Char('Survey Title', required=True, translate=True, copy=mark_as_copy('title'))
# addons/stock/models/stock_warehouse.py
code = fields.Char('Short Name', required=True, copy=lambda self: self.env._('COPY'))
# opt out of the default " (copy)" on name
name = fields.Char(copy=True)
```

❌ Do not override `copy()`/`copy_data()` just to add " (copy)" to `name`: 20 already does it.

### init_storage

Initializes NULL values of a new column on existing rows (replaces overriding
`_init_column()`, which warns `"Since 20.0, field initialization is defined in Field.init_storage"`).
The method may not use the ORM; returning a truthy value makes the ORM recompute the field.

```python
# addons/website_slides/models/slide_channel.py
access_token = fields.Char(copy=False, default=_default_access_token,
                           init_storage='_init_access_token')

def _init_access_token(self):
    self.env.execute_query(SQL("UPDATE %s SET access_token = ...", SQL.identifier(self._table)))
```

### Examples

```python
name = fields.Char(
    string="Name",
    required=True,
    index=True,
    default='Untitled',
    help="Enter the name",
    tracking=True,
)
```

---

## Reserved Field Names

| Name | Purpose |
|------|---------|
| `id` | Record identifier |
| `display_name` | Display name |
| `create_date`, `create_uid`, `write_date`, `write_uid` | Access log fields |
| `name` | Default `rec_name`; copied with " (copy)" |
| `active` | Global visibility toggle |
| `state` | Lifecycle stages; not copied by default |
| `parent_id` | Tree structure parent |
| `parent_path` | Tree structure path |
| `company_id` | Multi-company field |

---

## References

- Source: Odoo 20 `odoo/orm/fields.py`, `fields_binary.py`, `fields_textual.py`, `fields_relational.py`, `fields_temporal.py`, `odoo/tools/binary.py`, `odoo/tools/translate.py` (`mark_as_copy`)
- Docs: `content/developer/reference/backend/orm.rst` (20.0, "Binary fields")
