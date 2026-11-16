# Odoo 20 Decorator Guide

## Coding Conventions

- Keep field declarations and their default, compute, inverse, search, and
  constraint helpers together; name helpers after the field and operation.
- Use `@api.model_create_multi` for `create()` overrides, declare complete
  dotted dependencies for computed fields, and keep constraints batch-safe.
- Use `@api.ondelete(at_uninstall=False)` for deletion validation; do not use
  onchange methods to perform persistence or business-side effects.

Guide for using `@api` decorators in Odoo 20: computed fields, validation, onchange, caching, and more.

## Table of Contents

- [Method Decorators](#method-decorators)
- [@api.depends](#apidepends)
- [@api.constrains](#apiconstrains)
- [@api.ondelete](#apiondelete)
- [@api.onchange](#apionchange)
- [@api.model](#apimodel)
- [@api.model_create_multi](#apimodel_create_multi)
- [@api.autovacuum](#apiautovacuum)
- [@api.private](#apiprivate)
- [@api.readonly](#apireadonly)
- [@api.depends_context](#apidepends_context)
- [@api.ormcache](#apiormcache)
- [Removed decorators](#removed-decorators)

---

## Method Decorators

Decorators are exported by `odoo.api` (defined in `odoo/orm/decorators.py`; `ormcache` in `odoo/orm/cache.py`).
Odoo 20 exports: `autovacuum`, `constrains`, `depends`, `depends_context`, `model`,
`model_create_multi`, `onchange`, `ondelete`, `private`, `readonly`, `ormcache`.

```python
from odoo import api, models

class MyModel(models.Model):
    _name = 'my.model'

    @api.depends('field')
    def _compute_method(self):
        pass
```

---

## @api.depends

For computed fields. Specifies dependencies that trigger recomputation.

### Basic Usage

```python
total = fields.Float(compute='_compute_total', store=True)

@api.depends('value', 'tax')
def _compute_total(self):
    for record in self:
        record.total = record.value + record.value * record.tax
```

### Dotted Paths

Can use dotted paths for relational fields (depending on `id` raises
`NotImplementedError: Compute method cannot depend on field 'id'.`):

```python
@api.depends('partner_id.email')
def _compute_email(self):
    for record in self:
        record.email = record.partner_id.email
```

### Multiple Fields

Compute multiple fields:

```python
discount_value = fields.Float(compute='_apply_discount')
total = fields.Float(compute='_apply_discount')

@api.depends('value', 'discount')
def _apply_discount(self):
    for record in self:
        discount = record.value * record.discount
        record.discount_value = discount
        record.total = record.value - discount
```

### Search on Computed Field

```python
upper_name = fields.Char(compute='_compute_upper', search='_search_upper')

@api.depends('name')
def _compute_upper(self):
    for record in self:
        record.upper_name = record.name.upper() if record.name else False

def _search_upper(self, operator, value):
    if operator == 'like':
        operator = 'ilike'
    return [('name', operator, value)]
```

### Inverse Method

Allow setting computed field:

```python
document = fields.Char(compute='_get_document', inverse='_set_document')

@api.depends('document_path')
def _get_document(self):
    for record in self:
        with open(record.document_path) as f:
            record.document = f.read()

def _set_document(self):
    for record in self:
        if not record.document:
            continue
        with open(record.document_path, "w", encoding="utf-8") as f:
            f.write(record.document)
```

---

## @api.constrains

For validation. Called on create and write.

### Basic Usage

```python
@api.constrains('email')
def _check_email(self):
    for record in self:
        if not tools.email_validation(record.email):
            raise ValidationError("Invalid email")
```

### Multiple Fields

```python
@api.constrains('date_start', 'date_end')
def _check_dates(self):
    for record in self:
        if record.date_end < record.date_start:
            raise ValidationError("End date must be after start date")
```

### No Dotted Paths

Unlike `@api.depends`, **cannot use dotted paths**: they are ignored. The check only runs when
one of the named fields is in the `create`/`write` values.


```python
# BAD: dotted path not supported
@api.constrains('partner_id.email')
def _check_email(self):
    pass

# GOOD: use simple field name
@api.constrains('partner_id')
def _check_email(self):
    for record in self:
        if not tools.email_validation(record.partner_id.email):
            raise ValidationError("Invalid email")
```

---

## @api.ondelete

For delete validation.

```python
@api.ondelete(at_uninstall=False)
def _unlink_if_not_draft(self):
    if any(rec.state != 'draft' for rec in self):
        raise UserError("Cannot delete non-draft records")
```

### Parameters

| Parameter      | Description                                            |
| -------------- | ------------------------------------------------------ |
| `at_uninstall` | Required keyword. `False` (almost always): the check is skipped when the module is uninstalled |

### Why Use @api.ondelete?

- **Better than overriding `unlink()`**: Doesn't break module uninstall
- **Clear intent**: Explicitly for delete validation
- **Automatic**: Called before deletion

### Unlink Override (Anti-pattern)

```python
# BAD: breaks module uninstall
def unlink(self):
    if any(rec.state != 'draft' for rec in self):
        raise UserError("Cannot delete non-draft records")
    return super().unlink()
```

---

## @api.onchange

For form UI updates when field values change.

### Basic Usage

```python
@api.onchange('partner_id')
def _onchange_partner_id(self):
    if self.partner_id:
        self.email = self.partner_id.email
        self.phone = self.partner_id.phone
```

### Multiple Fields

```python
@api.onchange('country_id', 'state_id')
def _onchange_location(self):
    if self.country_id:
        # Update zip format
        pass
```

### No CRUD Operations

**Important**: `onchange` methods should **not** perform CRUD operations.

```python
# BAD: create in onchange
@api.onchange('field1')
def _onchange_field1(self):
    self.env['another.model'].create({'name': 'test'})

# GOOD: only modify current record
@api.onchange('field1')
def _onchange_field1(self):
    self.field2 = 'computed value'
```

### Return Warning

```python
@api.onchange('amount')
def _onchange_amount(self):
    if self.amount < 0:
        return {
            'warning': {
                'title': "Warning",
                'message': "Amount cannot be negative",
            }
        }
```

---

## @api.model

For model-level methods that don't depend on `self`.

### Usage

```python
@api.model
def get_default_values(self):
    return {
        'field1': 'value1',
        'field2': 'value2',
    }
```

### Can be called on any recordset

```python
# Can call on any recordset (self may be empty)
record = self.env['my.model'].browse([1, 2, 3])
defaults = record.get_default_values()
```

---

## @api.model_create_multi

For handling batch create operations.

```python
@api.model_create_multi
def create(self, vals_list):
    # Add default values
    for vals in vals_list:
        vals.setdefault('field', 'default')
    return super().create(vals_list)
```

### Why Use It?

`create()` may be called with a dict or a list of dicts; the decorator always passes a list.
`@api.model` on a method named `create` is turned into `@api.model_create_multi` automatically.

---

## @api.autovacuum

For methods run by the daily `ir.autovacuum` cron. The method name must start with `_`
(assertion). It may return `(done, remaining)` like `ir.cron._commit_progress`.

```python
@api.autovacuum
def _gc_entries(self):
    # Clean old records
    domain = [('create_date', '<', date.today() - timedelta(days=90))]
    self.search(domain).unlink()
```

---

## Decorator Decision Tree

```
Need to define field behavior?
├── Field computed from other fields → @api.depends
│   └── CAN use dotted paths
├── Validate data → @api.constrains
│   └── CANNOT use dotted paths
├── Prevent record deletion → @api.ondelete
└── Update form UI → @api.onchange
    └── NO CRUD operations allowed

Need to define method behavior?
├── Method-level, doesn't depend on self → @api.model
├── Mark method as non-RPC callable → @api.private
├── RPC method safe on a read-only cursor → @api.readonly
├── Cache a pure result per registry → @api.ormcache
└── Normal record method → no decorator needed
```

---

## @api.private

Marks a method as **not callable via RPC** (external API). Since 18.2.

### Usage

```python
from odoo import api, models

class MyModel(models.Model):
    _name = 'my.model'

    @api.private
    def compute_totals(self):
        """Public name, but cannot be called via XML-RPC/JSON-RPC."""
        return self._do_heavy_work()

    def public_action(self):
        """This method CAN be called via RPC."""
        return self.compute_totals()
```

### When to Use

- Methods that should only be called internally (not via API/button)
- Public (non-`_`) method names that must not be exposed, e.g. kept for backward compatibility
- ORM override methods that you don't want exposed

### @api.private vs Underscore Convention

```python
# Underscore prefix: refused over RPC (odoo.models.get_public_method, exported in 20)
def _do_stuff(self):
    pass

# @api.private: same RPC refusal for a method whose name has no underscore
@api.private
def compute_sensitive_data(self):  # Name doesn't need underscore
    pass
```

---

## @api.readonly

Allows `self.env.cr` to be a read-only cursor when the method is called through RPC. Use only for methods that never write.

```python
@api.readonly
def get_dashboard_data(self):
    return self._read_group([], ['state'], ['__count'])
```

---

## @api.depends_context

Declares context keys a non-stored computed field depends on (cache is keyed by them).

```python
@api.depends_context('company')
def _compute_balance(self):
    for record in self:
        record.balance = record._get_balance(self.env.company)
```

---

## @api.ormcache

Moved to `odoo.api` in 20 (thread-safe, transactional cache, 19.4). Arguments are expressions
on the method signature; `cache=` selects the cache name. Never return recordsets.

```python
# ❌ Odoo 19 (still works, warns "Since 20.0 import ormcache from odoo.api")
from odoo.tools import ormcache
@ormcache('key')

# ✅ Odoo 20 (odoo/addons/base/models/ir_config_parameter.py)
@api.ormcache('key', 'type_', cache='stable')
def _get(self, key, type_='str'):
    ...

# invalidate (registry.clear_cache was removed)
self.env.transaction.invalidate_ormcache('stable')
```

`tools.ormcache_context` no longer exists; add the context value to the key expressions.
A model can set `_clear_cache_name` (+ `_clear_cache_on_fields`) to invalidate a cache
automatically on create/write/unlink.

---

## Removed Decorators

| Removed | Since | Replacement |
|---------|-------|-------------|
| `@api.returns(...)` | 19.0 | Drop it; return the recordset directly (`copy()`, `message_post()` need nothing) |
| `@api.deprecated(...)` | 20.0 (exported by `odoo.api` in 19) | `from odoo.tools.func import deprecated` (`warnings.deprecated` on Python 3.13+, a fallback on 3.12), as used in `odoo/orm/models.py` |

---

## Common Patterns

### Computed Field with Inverse

```python
total = fields.Float(compute='_compute_total', inverse='_inverse_total', store=True)

@api.depends('subtotal', 'tax')
def _compute_total(self):
    for record in self:
        record.total = record.subtotal + record.tax

def _inverse_total(self):
    for record in self:
        record.subtotal = record.total - record.tax
```

### Validation with Constraints

```python
@api.constrains('age')
def _check_age(self):
    for record in self:
        if record.age < 18:
            raise ValidationError("Must be 18 or older")
```

### Delete Validation

```python
@api.ondelete(at_uninstall=False)
def _unlink_if_not_cancelled(self):
    if any(rec.state != 'cancel' for rec in self):
        raise UserError("Only cancelled records can be deleted")
```

### Onchange for Defaults

```python
@api.onchange('partner_id')
def _onchange_partner_id(self):
    if self.partner_id:
        self.lang = self.partner_id.lang
        self.user_id = self.partner_id.user_id
```

---

## References

- Source: Odoo 20 `odoo/orm/decorators.py`, `odoo/orm/cache.py`, `odoo/api/__init__.py`
- Docs: `content/developer/reference/backend/orm.rst` (20.0)
