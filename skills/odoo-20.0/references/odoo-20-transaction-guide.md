# Odoo 20 Transaction Guide

## Coding Conventions

- Never call `commit()` or `rollback()` on `self.env.cr`: Odoo owns normal
  request and cron transactions. Re-raise failures or use a savepoint.
- A manual commit or rollback is exceptional and only permitted on a cursor
  explicitly created by the current code. Make the unit restartable and
  document why its independent transaction boundary is safe.
- Prefer a savepoint for recoverable database errors. Catch only exceptions
  that can be handled; re-raise unexpected failures.
- Use `self.env.cr`; `self._cr` is deprecated since 19.0
  ("Deprecated since 19.0, use self.env.cr directly").

Guide for handling database transactions in Odoo 20: errors, savepoints, and serialization failures.

## Table of Contents
- [Transaction Overview](#transaction-overview)
- [Cursor API](#cursor-api)
- [Database Errors](#database-errors)
- [Savepoints](#savepoints)
- [Error Handling](#error-handling)
- [Serialization Failures](#serialization-failures)
- [Commit and Rollback](#commit-and-rollback)
- [Best Practices](#best-practices)

---

## Transaction Overview

Odoo uses database transactions to ensure data consistency. Cursors run at
PostgreSQL `REPEATABLE READ` isolation.

### Transaction Properties

| Property | Description |
|-----------|-------------|
| **Atomicity** | All or nothing |
| **Consistency** | Data remains valid |
| **Isolation** | Concurrent transactions don't interfere |
| **Durability** | Committed data persists |

### Transaction Flow

```
Begin Transaction
├── Execute Operations (ORM writes are buffered until flush)
├── flush() → precommit hooks → COMMIT → postcommit hooks
│   (or ROLLBACK → prerollback/postrollback hooks)
└── End Transaction
```

---

## Cursor API

`odoo.sql_db.Cursor` (`BaseCursor` is kept only as a backward-compatible alias in 20).

| Member | Description |
|--------|-------------|
| `execute(query, params=None, log_exceptions=True)` | Run an `SQL` object (preferred) or a literal string |
| `executemany(query, vars_list)` | Batch execute (new in 20) |
| `savepoint(flush=True)` | Context manager; flushes the ORM first by default |
| `flush()` | Flush ORM changes and run `precommit` hooks |
| `commit()` / `rollback()` | Only on cursors you opened yourself |
| `precommit`, `postcommit`, `prerollback`, `postrollback` | `Callbacks` hooks (`.add(func)`) |
| `now()` | Transaction timestamp (UTC) |
| `dictfetchone/many/all()` | Rows as dicts |
| `IN_MAX` | Recommended max size for `IN` lists |

Removed in 20: `cr.split_for_in_conditions()` (deprecated in 19, use
`split_every(cr.IN_MAX, ids)`), `cr.clear()` and `cr.reset()` (use
`env.transaction.clear()` / `env.transaction.reset()`). `env.clear()` is
deprecated: "Since 20.0, use transaction.clear or transaction.reset".

Prefer `self.env.execute_query(SQL(...))` / `execute_query_dict(SQL(...))`:
they flush the fields referenced by the `SQL` object before running it.

### Post-commit Hooks

Run side effects (external calls) only once the transaction is committed:

```python
@self.env.cr.postcommit.add
def called_after():
    with Registry(dbname).cursor() as cr:     # new cursor: commits on exit
        env = api.Environment(cr, uid, context)
        ...
```

---

## Database Errors

### Common Errors

| Error | When |
|-------|------|
| `UniqueViolation` | Duplicate unique constraint |
| `NotNullViolation` | NULL in NOT NULL column |
| `ForeignKeyViolation` | Invalid foreign key |
| `CheckViolation` | CHECK constraint failed |
| `SerializationFailure` | Concurrent modification |
| `LockNotAvailable` / `DeadlockDetected` | Lock conflicts (retried like serialization failures) |

### Catch Database Errors

ORM writes are buffered: an integrity error is raised at flush time, not at
`write()`. Catch it around a **savepoint**, which flushes on exit, so the
transaction stays usable:

```python
from psycopg2 import errors

try:
    with self.env.cr.savepoint():
        channel = self.create(vals)
except errors.UniqueViolation:
    # created concurrently: reuse it instead of failing
    channel = self._get_existing(vals)
```

```python
# ❌ the error surfaces later (at flush) and the transaction is aborted
try:
    record.write({'code': 'X'})
except errors.UniqueViolation:
    ...
```

Uncaught `IntegrityError` in a request/RPC is converted by `retrying()` into
a `ValidationError` ("The operation cannot be completed: ...") built with
`model._sql_error_to_message(exc)`.

### Odoo Exceptions (`odoo.exceptions`)

| Exception | HTTP status | Use |
|-----------|-------------|-----|
| `UserError` | 422 | Business error shown to the user |
| `ValidationError` | 422 | Constraint violation (subclass of `UserError`) |
| `AccessError` | 403 | Access rights |
| `AccessDenied` | 403 | Login error; traceback concealed (`conceal_debug_traceback = True`) |
| `MissingError` | 404 | Deleted/nonexistent records |
| `LockError` | 409 | `lock_for_update()` could not lock |
| `RedirectWarning` | - | Warning with redirect action |
| `ConcurrencyError` | - | Low-level: forces a retry by `retrying()` |

`AccessDenied.suppress_traceback()` is removed in 20; the traceback is
concealed through the `conceal_debug_traceback` attribute.

---

## Savepoints

Savepoints isolate errors within a transaction.

### Using Savepoints

```python
def process_records(self):
    for record in self:
        try:
            with self.env.cr.savepoint():
                record.process()
        except (ValidationError, UserError) as error:
            _logger.warning("Failed to process %s: %s", record, error)
```

### Savepoint Lifetime

```python
with self.env.cr.savepoint():
    record.process()
# The context manager flushes and releases the savepoint on success, and rolls
# it back (restoring the ORM transaction state and clearing precommit hooks)
# when an exception escapes the block.
```

The savepoint object can be rolled back explicitly and reused, as done for
sequence numbers (`account/models/sequence_mixin.py`):

```python
with self.env.cr.savepoint(flush=False) as sp:
    while True:
        try:
            self.env.cr.execute(SQL("UPDATE ...", ...), log_exceptions=False)
            break
        except errors.UniqueViolation:
            sp.rollback()
```

Savepoints have a performance cost; avoid one per record in large loops when
not needed.

---

## Error Handling

### Retry on Serialization Failure

Do not retry an ambient transaction inside model code. Requests and RPC
calls go through `odoo.http.retrying.retrying()` (moved from
`odoo.service.model` in 20): on `SerializationFailure`, `LockNotAvailable`,
`DeadlockDetected` or `ConcurrencyError` it rolls back and replays the whole
call up to 5 times with random exponential backoff. Re-raise these errors so
it can do so.

### Locking Rows

```python
records.lock_for_update()                 # raises LockError if any row is locked
locked = records.try_lock_for_update(limit=100)   # skip rows locked elsewhere
```

Both use `FOR UPDATE SKIP LOCKED` (`allow_referencing=True` uses
`FOR NO KEY UPDATE`).

### Handle Validation Errors

```python
from odoo.exceptions import ValidationError
from odoo.tools import email_normalize

@api.constrains('email')
def _check_email(self):
    for record in self:
        if record.email and not email_normalize(record.email):
            raise ValidationError(self.env._("Invalid email: %s", record.email))
```

---

## Serialization Failures

### What is Serialization Failure?

Occurs when two transactions try to modify the same data concurrently.

### Avoid Serialization Failures

```python
# BAD: Loop with search and write
def process(self):
    for record in self.search([('state', '=', 'draft')]):
        record.write({'state': 'done'})

# GOOD: Single write
def process(self):
    self.search([('state', '=', 'draft')]).write({'state': 'done'})
```

### Lock Before Processing

```python
# ❌ hand-written FOR UPDATE: waits on rows locked by other transactions
self.env.cr.execute("SELECT id FROM my_model WHERE id IN %s FOR UPDATE", (tuple(self.ids),))

# ✅ fails fast with LockError instead of waiting
self.lock_for_update()
```

---

## Commit and Rollback

### Auto Commit

Odoo commits the transaction once at the end of a successful request or job
(controller call, RPC call, cron job), not immediately after each ORM call. If
an exception is raised before that boundary, the whole transaction is rolled
back, undoing every write made since it started.

```python
from odoo import http
from odoo.http import request

class MyController(http.Controller):
    @http.route('/process', auth='user', type='jsonrpc')
    def process(self, record_id):
        record = request.env['my.model'].browse(record_id)
        record.write({'field': 'value'})
        # No manual commit here: Odoo commits automatically when this
        # request completes successfully.
```

### Cron Batches

Long crons commit progress through the cron API instead of `cr.commit()`:

```python
self.env['ir.cron']._commit_progress(remaining=len(todo))
for item in todo:
    item._process()
    self.env['ir.cron']._commit_progress(1)
```

### Recoverable Work

```python
try:
    with self.env.cr.savepoint():
        record1.write({'field': 'value'})
        record2.write({'field': 'value'})
except ValidationError:
    # The savepoint rolls back only this recoverable unit.
    raise
```

Only an explicitly created cursor (`with self.env.registry.cursor() as cr:`,
committed on success and closed on exit) may be committed or rolled back. Such
code must document why the independently restartable transaction boundary is
safe.

---

## Best Practices

### Batch Operations

```python
# GOOD: Batch create
def create_records(self, values_list):
    return self.create(values_list)

# BAD: Create in loop
def create_records(self, values_list):
    for values in values_list:
        self.create(values)
```

### Use Context for Special Cases

```python
# Skip tracking for bulk update
records.with_context(tracking_disable=True).write({'field': 'value'})
```

---

## Common Patterns

### Safe Update Pattern

```python
def safe_update(self, values):
    try:
        with self.env.cr.savepoint():
            self.write(values)
    except errors.UniqueViolation:
        raise UserError(self.env._("Duplicate entry!"))
    except errors.NotNullViolation:
        raise UserError(self.env._("Required field missing!"))
```

### Bulk Processing with Savepoints

```python
def bulk_process(self, records):
    for record in records:
        try:
            with self.env.cr.savepoint():
                record.process()
        except (ValidationError, UserError) as error:
            _logger.warning("Failed: %s", error)
```

---

## References

- Source: `odoo/sql_db.py` (Cursor, Savepoint), `odoo/http/retrying.py`, `odoo/exceptions.py`, `odoo/orm/environments.py` (Transaction), `odoo/addons/base/models/ir_cron.py`
- PostgreSQL documentation on transactions
