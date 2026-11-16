# Odoo 20 Performance Guide

## Coding Conventions

Prefer batch-aware ORM operations; do not commit Odoo's ambient transaction.

Guide for optimizing Odoo 20 code: preventing N+1 queries, reducing database queries, caching, and using profiler.

## Table of Contents

- [Profiling](#profiling)
- [Batch Operations](#batch-operations)
- [Grouping (`_read_group`)](#grouping-_read_group)
- [SQL Builder and `compute_sql`](#sql-builder-and-compute_sql)
- [ormcache](#ormcache)
- [Algorithmic Complexity](#algorithmic-complexity)
- [Indexes](#indexes)
- [Performance Pitfalls](#performance-pitfalls)
- [Generating Test Data (`populate`)](#generating-test-data-populate)

---

## Profiling

Odoo provides an integrated profiling tool to record SQL queries and stack traces.

### Enable from User Interface

1. Enable developer mode
2. Toggle **Enable profiling** button
3. Choose expiry time
4. Toggle **Enable profiling** again to start session profiling

Options:

- **Record sql** - Saves all SQL queries with stack trace
- **Record traces** - Saves stack trace periodically (default: 10ms interval)

### Enable from Python Code

```python
from odoo.tools.profiler import ExecutionContext, PeriodicCollector, Profiler

# Basic profiling (default collectors: 'sql', 'traces_async')
with Profiler():
    do_stuff()

# With custom collectors
with Profiler(collectors=['sql', PeriodicCollector(interval=0.1)]):
    do_stuff()

# In tests
with self.profile():
    do_stuff()
```

### Collectors

| Collector          | Key            | Description                                      |
| ------------------ | -------------- | ------------------------------------------------ |
| SQL collector      | `sql`          | Saves SQL queries with stack trace               |
| Periodic collector | `traces_async` | Saves stack trace periodically (separate thread) |
| QWeb collector     | `qweb`         | Saves QWeb directive execution                   |

The Sync collector (`traces_sync`, `SyncCollector`) no longer exists in Odoo 20.

### Execution Context

Add context to identify calls in speedscope:

```python
for index in range(max_index):
    with ExecutionContext(current_index=index):
        do_stuff()
```

### Profiling Pitfalls

- Randomness: other processes or threads running on the system can change results
- Blocking calls may cause unexpected long frames
- Cache state affects results (view/assets in cache)
- Profiler overhead can impact performance (especially SQL collector)
- Large profiles may cause memory issues

---

## Batch Operations

### Avoid Loop Queries

**BAD**: Search in loop (N queries)

```python
def _compute_count(self):
    for record in self:
        domain = [('related_id', '=', record.id)]
        record.count = other_model.search_count(domain)
```

**GOOD**: Use `_read_group` (1 query)

```python
def _compute_count(self):
    domain = [('related_id', 'in', self.ids)]
    counts_data = other_model._read_group(domain, ['related_id'], ['__count'])
    mapped_data = {
        related.id: count
        for related, count in counts_data
        if related
    }
    for record in self:
        record.count = mapped_data.get(record.id, 0)
```

### Batch Creates

**BAD**: Create in loop

```python
for name in ['foo', 'bar']:
    model.create({'name': name})
```

**GOOD**: Batch create

```python
create_values = [{'name': name} for name in ['foo', 'bar']]
records = model.create(create_values)
```

### Prefetch Records

**BAD**: Browse one at a time

```python
for record_id in record_ids:
    record = model.browse(record_id)
    record.foo  # One query per record
```

**GOOD**: Browse all together

```python
records = model.browse(record_ids)
for record in records:
    record.foo  # One query for entire recordset
```

### Control Prefetching

```python
for values in values_list:
    message = self.browse(values['id']).with_prefetch(self.ids)
```

Fetch exactly what you need in one query with `search_fetch(domain, field_names)`
or `records.fetch(field_names)`.

---

## Grouping (`_read_group`)

| Method | Where | Signature / result |
| ------ | ----- | ------------------ |
| `_read_group` | ORM, server code | `(domain, groupby, aggregates, having, offset, limit, order)` → list of tuples with **records** for relational groups |
| `read_group` | ORM, public (RPC) | same signature → list of tuples with **ids** instead of records |
| `formatted_read_group` | `web` module | same signature → list of dicts (UI format) |

```python
# ❌ Odoo 19 deprecated signature, removed in 20
Model.read_group(domain, ['amount:sum'], ['partner_id'], lazy=False)

# ✅ Odoo 20
Model._read_group(domain, ['partner_id'], ['amount:sum'])        # [(partner, 120.0), ...]
Model.read_group(domain, ['partner_id'], ['amount:sum'])         # [(7, 120.0), ...]
```

Aggregates: `'field:agg'` (PostgreSQL aggregate), `'__count'`,
`'field:count_distinct'`, `'field:recordset'`. Groupby granularities:
`'date:day|week|month|quarter|year'` and date parts such as
`'date:month_number'`.

---

## SQL Builder and `compute_sql`

### SQL Builder (19.1+)

Use `SQL` objects (`odoo.tools.SQL`) and the query's `TableSQL`
(`query.table.<field>` renders the field's SQL, adding the joins needed for
inherited, related or `compute_sql` fields):

```python
query = self._search([('id', 'in', self.ids)])
rows = self.env.execute_query(
    query.select(query.table.id, query.table.consolidation_rate)
)
```

- `Query` moved to `odoo.orm.query` (import as `from odoo.models import Query`);
  `odoo/tools/query.py` no longer exists.
- `SQL.code` / `SQL.params` properties are gone; the composed value is
  `sql._sql_tuple` (code, params, to_flush).
- `Model._field_to_sql()` is `@typing.final`: do not override it, use
  `compute_sql` on the field.
- `Model._table_query` is removed (20.0).

### `Field.compute_sql` (19.1+)

A computed non-stored field can provide SQL, making it searchable, groupable
and sortable without storing it. An explicit `compute_sudo` is required
(warning otherwise):

```python
is_favorite = fields.Boolean(
    compute='_compute_is_favorite', compute_sql='_compute_sql_is_favorite',
    compute_sudo=True,
)

def _compute_sql_is_favorite(self, table):
    return SQL(
        "%s IN (SELECT project_id FROM project_favorite_user_rel WHERE user_id = %s)",
        table.id, self.env.uid,
    )
```

---

## ormcache

Since 19.4 ormcache is thread-safe and bound to the transaction: invalidations
are local to the transaction and signaled to other workers at commit.

```python
# ❌ Odoo 19
from odoo import tools
@tools.ormcache('key')                  # warns: "Since 20.0 import ormcache from odoo.api"
self.env.registry.clear_cache()         # removed
self.env.registry.clear_cache('templates')

# ✅ Odoo 20
from odoo import api
@api.ormcache('key', 'type_', cache='stable')
def _get(self, key, type_='str'): ...

self.env.transaction.invalidate_ormcache()             # 'default'
self.env.transaction.invalidate_ormcache('templates')
```

- `odoo-bin upgrade_code` script `19.4-00-ormcache-on-transaction` rewrites
  `registry.clear_cache` to `transaction.invalidate_ormcache`.
- `ormcache_context` and the `skiparg` parameter no longer exist.
- Cache names: `default`, `stable`, `assets`, `templates`, `routing`, `groups`.
- Auto-invalidation on create/write/unlink: set `_clear_cache_name = 'default'`
  (optionally `_clear_cache_on_fields = (...)`, `None` = all fields).
- Never return recordsets from a cached method.
- `models.CachedModel` caches `_cached_data_fields` of all records
  (`_cached_data_domain`) in the `stable` cache; `get_all()` returns them
  (used by `res.lang`, `res.currency`, `res.country`, `res.company`, `website`).

---

## Algorithmic Complexity

### Reduce Nested Loops

**BAD**: O(n²) complexity

```python
for record in self:
    for result in results:
        if result['id'] == record.id:
            record.foo = result['foo']
            break
```

**GOOD**: Use dictionary (O(n))

```python
mapped_result = {result['id']: result['foo'] for result in results}
for record in self:
    record.foo = mapped_result.get(record.id)
```

### Use Set Operations

**BAD**: List-like `in` check (quadratic)

```python
invalid_ids = self.search(domain).ids
for record in self:
    if record.id in invalid_ids:  # O(n) for each record
        ...
```

**GOOD**: Use set (O(n) total)

```python
invalid_ids = set(self.search(domain).ids)
for record in self:
    if record.id in invalid_ids:
        ...
```

**ALTERNATIVE**: Recordset operations

```python
invalid_ids = self.search(domain)
for record in self - invalid_ids:
    ...
```

---

## Indexes

Database indexes speed up search operations.

**Warning**: Don't index every field - indexes consume space and impact INSERT/UPDATE/DELETE performance.

### Field Index Types

| `index=` | Index |
| -------- | ----- |
| `True` / `'btree'` | Standard B-tree (many2one, frequent `=` searches) |
| `'btree_not_null'` | B-tree without NULL values (mostly-NULL columns) |
| `'trigram'` | GIN trigram index (`ilike` searches) |
| `False` / `None` | No index (default) |

```python
partner_id = fields.Many2one('res.partner', index=True)
company_id = fields.Many2one(related='location_id.company_id', store=True, index='btree_not_null')
name = fields.Char(index='trigram')
```

### Declarative Index

For composite or expression indexes, use `models.Index` as a model attribute:

```python
class MyModel(models.Model):
    _name = 'my.model'

    name = fields.Char()
    code = fields.Char()

    _name_code_idx = models.Index('(name, code)')
```

`models.UniqueIndex(definition, message)` creates a unique index.

---

## Performance Pitfalls

### N+1 Query Problem

Occurs when you:

1. Fetch a list of records
2. Loop through them
3. Execute a query for each record

**Detection**: Use `--log-sql` CLI parameter or profiler

**Solution**: Fetch related data in one query using:

- `_read_group()`
- `search_fetch()`
- `fetch()`
- `mapped()`

### Large Recordsets

Processing large recordsets can cause memory issues.

**Solution**: Process in batches

```python
def _process_large_dataset(self):
    limit = 1000
    offset = 0
    while True:
        records = self.search([], limit=limit, offset=offset)
        if not records:
            break
        records.process()
        offset += limit
```

### Computed Field Dependencies

Missing dependencies cause recomputation at wrong time.

**BAD**: Missing dotted dependency

```python
@api.depends('partner_id')
def _compute_email(self):
    for record in self:
        record.email = record.partner_id.email  # N queries!
```

**GOOD**: Include dotted path

```python
@api.depends('partner_id.email')
def _compute_email(self):
    for record in self:
        record.email = record.partner_id.email  # 1 query for all
```

### Context Pollution

Excessive context changes can cause issues.

```python
# BAD: Too many with_context calls
records.with_context(lang='fr').with_context(active_test=False).with_context(...)
```

**Solution**: Consolidate context changes

```python
# GOOD: Single with_context call
records.with_context(lang='fr', active_test=False)
```

### Unnecessary Cache Invalidation

Invalidating the whole record cache too often defeats the purpose of caching
(`invalidate_cache()` does not exist; `env.invalidate_all()` drops everything).

**Solution**: Only invalidate fields that actually changed

```python
# GOOD: Invalidate only changed fields
records.invalidate_recordset(['field1', 'field2'])
# or for all records of the model
self.env['my.model'].invalidate_model(['field1'])
```

---

## Query Count Testing

Use `assertQueryCount` in tests to establish query limits.

```python
with self.assertQueryCount(11):
    do_something()
```

Combine with profiler for analysis:

```python
with self.profile():
    with self.assertQueryCount(__system__=1211):
        do_stuff()
```

---

## Generating Test Data (`populate`)

New in 20.0: the `populate` module generates synthetic data from XML/JSON
blueprints (`populate.blueprint` records, e.g. `sale/populate/populate_demo.xml`).
Install the `populate` module, then:

```console
$ odoo-bin populate -d <database> -b <blueprint xmlid> [--seed N] [--scale F] [-j N|auto] [--resume [session_id]] [--profile]
```

Upgrade `populate` (`-u populate`) after installing a module that ships
blueprints. To multiply existing records instead, use
`odoo-bin duplicate -d <db> --models res.partner --factors 1000`.

---

## Good Practices Summary

| Practice                   | Description                                   |
| -------------------------- | --------------------------------------------- |
| **Batch operations**       | Accumulate operations, execute in batch       |
| **Use \_read_group**       | Replace search/search_count in loops          |
| **Prefetch records**       | Browse all records together                   |
| **Reduce complexity**      | Use dictionaries/sets instead of nested loops |
| **Add indexes**            | On frequently searched fields                 |
| **Use fetch/search_fetch** | For targeted data loading                     |
| **compute_sql**            | Group/sort/search computed fields in SQL      |
| **Profile first**          | Use profiler before optimizing                |
| **Test query counts**      | Use assertQueryCount in tests                 |

---

## References

- Source: `odoo/tools/profiler.py`, `odoo/orm/models.py` (`_read_group`, `read_group`), `addons/web/models/models.py` (`formatted_read_group`), `odoo/orm/cache.py`, `odoo/orm/models_cached.py`, `odoo/orm/query.py`, `odoo/tools/sql.py`, `addons/populate/cli/populate.py`
- Odoo 20 documentation `/developer/reference/backend/performance.rst`, `/developer/reference/backend/populate.rst`
