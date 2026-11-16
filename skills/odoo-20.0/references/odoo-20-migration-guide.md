# Odoo 20 Migration Guide

Guide for migrating custom modules from Odoo 19 to Odoo 20: migration scripts, module hooks,
`odoo-bin upgrade_code`, and the 19.0 → 20.0 breaking changes (Online versions 19.1 → 19.5 are
all included in 20.0).

## Table of Contents
- [Migration Overview](#migration-overview)
- [Migration Scripts](#migration-scripts)
- [Module Hooks](#module-hooks)
- [upgrade_code: Automatic Source Rewrite](#upgrade_code-automatic-source-rewrite)
- [Breaking Changes 19 → 20](#breaking-changes-19--20)
- [Migration Checklist](#migration-checklist)
- [Testing Migrations](#testing-migrations)

---

## Migration Overview

Two different things are called "migration":

| What | Tool | Acts on |
|------|------|---------|
| **Source code** port 19 → 20 | `odoo-bin upgrade_code` + manual fixes | Your `.py`, `.xml`, `.js`, `.csv` files |
| **Data** upgrade of a module version | `migrations/<version>/{pre,post,end}-*.py` | The database, on `-u my_module` |

### Strategy

1. Run `odoo-bin upgrade_code --from 19.0 --dry-run`, then without `--dry-run` (commit first)
2. Run the OWL 3 script separately (`--script owl3-migration`) if the module has JS
3. Go through the [checklist](#migration-checklist); fix what the scripts cannot
4. Bump `__manifest__.py` `version` and add data migration scripts if the schema/data changed
5. Install and update on a 20.0 database with tests (`-u my_module --test-tags /my_module`)

---

## Migration Scripts

### Location and Version Folders

```
my_module/
├── migrations/            # also read: my_module/upgrades/ and --upgrade-path dirs
│   ├── 1.1/               # module version → treated as 20.0.1.1 on a 20.0 server
│   │   ├── pre-migrate.py
│   │   ├── post-migrate.py
│   │   └── end-migrate.py
│   ├── 20.0.1.2/          # full version: only processed by a 20.0 server
│   │   └── post-cleanup.py
│   └── 0.0.0/             # run on every version change of the module
│       └── end-invariants.py
```

- Folder names must match `VERSION_RE` in `odoo/modules/migration.py` (`x.y`, `x.y.z`, or
  `<major>.0.x.y[.z]`); invalid folders are skipped with `Invalid version for upgrade script`.
  A `tests` folder is ignored.
- A folder runs when `installed_version < folder_version <= manifest version`. A short folder
  (`1.1`) is compared on the module part only, so it is not re-run when the module moves from
  `19.0.1.1` to `20.0.1.1`.
- `0.0.0` scripts run first in the `pre` stage and last in `post`/`end`.

### Script Stages

Any `.py` file whose name **starts with** the stage prefix is executed (sorted by file name):

| Prefix | When it runs |
|--------|--------------|
| `pre-` | Before the module's models/data are loaded (old schema, raw SQL only) |
| `post-` | After the module is loaded (new schema, ORM usable) |
| `end-` | After all modules have been updated |

### Script Template

```python
# migrations/1.1/post-migrate.py
from odoo import api, SUPERUSER_ID


def migrate(cr, version):
    """``version`` is the installed version before the update ('' on install)."""
    if not version:
        return
    env = api.Environment(cr, SUPERUSER_ID, {})
    env['my.model'].search([('state', '=', False)]).state = 'draft'
```

`migrate` must be a plain function with exactly two positional parameters named
`cr`/`_cr` and `version`/`_version`; anything else raises `TypeError`
(`exec_script()` in `odoo/modules/migration.py`).

### Related CLI Options

| Option | Effect |
|--------|--------|
| `--upgrade-path PATH,...` | Extra directories with `<module>/<version>/` upgrade scripts |
| `--pre-upgrade-scripts PATH,...` | Scripts run before loading any module when `-u` is given |

---

## Module Hooks

Hooks receive `env`. The manifest value is the **name of a function exposed by the module's
`__init__.py`** (looked up with `getattr(py_module, name)` in `odoo/modules/loading.py`), not
a dotted path.

```python
# my_module/__init__.py
from . import models


def pre_init_hook(env):
    """On install, before the module's models and data are loaded."""


def post_init_hook(env):
    """After the module is installed (install only, not on update)."""


def uninstall_hook(env):
    """Before the module is uninstalled (its data still exists)."""
```

```python
# __manifest__.py
{
    ...
    'pre_init_hook': 'pre_init_hook',
    'post_init_hook': 'post_init_hook',
    'uninstall_hook': 'uninstall_hook',
}
```

```python
# ❌ Wrong: dotted path, raises AttributeError
'post_init_hook': 'my_module.post_init_hook',
```

---

## upgrade_code: Automatic Source Rewrite

`odoo-bin upgrade_code` (`odoo/cli/upgrade_code.py`) runs the scripts of `odoo/upgrade_code/`
over every `.py .js .css .scss .xml .csv .po .pot` file of the addons path (`__pycache__` and
`.git` skipped). Scripts are best-effort: always review the diff.

```bash
# all scripts from 19.1 up to the server version (default --to = odoo.release.version)
odoo-bin upgrade_code --addons-path=/path/to/custom --from 19.1 --dry-run
odoo-bin upgrade_code --addons-path=/path/to/custom --from 19.1

# one script (path or unique name fragment)
odoo-bin upgrade_code --addons-path=/path/to/custom --script owl3-migration

# restrict files
odoo-bin upgrade_code --addons-path=/path/to/custom --from 19.1 --glob 'my_module/**/*.xml'

# standalone, without a configured server
python3 odoo/cli/upgrade_code.py --addons-path /path/to/custom --from 19.1
```

| Option | Meaning |
|--------|---------|
| `--from VERSION` / `--script NAME` | Required, mutually exclusive |
| `--to VERSION` | Inclusive upper bound (default: server version) |
| `--glob GLOB` | Files to rewrite, relative to each addons path (default `**/*`) |
| `--dry-run` | Print `updated:` / `deleted:` files, write nothing |
| `--addons-path PATH,...` | Addons to rewrite (required when run standalone) |

Prints `updated: <path>` / `deleted: <path>` per file; exit code is `1` when something changed.
`--from 19.0` and `--from 19.1` select the same scripts (no `19.0-*` script exists).

### Scripts 19.1 → 20.0

| Script | Rewrites |
|--------|----------|
| `19.1-00-t-call.py` | QWeb XML (not `static/`): `<t t-set>` children of a `t-call` become attributes of the `t-call` (`name="expr"`, `name.f="text"` for literals/`t-valuef`, `name.translate="..."` for body text); t-sets with content/conditions/groups are moved before the `t-call`. Inherited `t-call`s get `position="attributes"`/`"before"` xpaths. |
| `19.3-00-base64-in-xml.py` | `<field type="base64" file="..."/>` → `type="bytes"` in XML data |
| `19.3-00-account-groups.py` | Odoo l10n chart templates only (`account.group` CSV merged into `account.account` CSV) |
| `19.3-00-account-report-foldable.py` | `account.report.line` `foldable` boolean → removed or `foldability` selection |
| `19.4-00-ir-access.py` | Converts `ir.model.access.csv` + `ir.rule` XML records of each module into one `security/ir.access.csv` (or `ir.access.csv` if no `security/` dir), updates the manifest `data`, deletes emptied files. Logs `WARNING`/`ERROR` lines to check manually. Not idempotent: run once, on all modules together. |
| `19.4-00-ormcache-on-transaction.py` | `registry.clear_cache` → `transaction.invalidate_ormcache` in `.py` |
| `19.5-00-material-symbols-icons.py` | Font Awesome / `oi-*` icon markup → Material Symbols: `<i class="fa fa-user"/>` → `<i class="oi" data-icon="person"/>`, also in attributes, dict entries, JS strings, CSS selectors |
| `19.5-00-tuple-rec_names_search.py` | `_rec_names_search = ['name']` → `('name',)` in `models/*.py` |
| `20.0-00-search-date-filters.py` | Drops `start_month`/`end_month`/`start_year`/`end_year` from search `<filter>` |
| `owl3-migration.py` | OWL 2 → 3 (JS + templates). **Not selected by `--from`** (no version prefix): run with `--script owl3-migration`. |

`owl3-migration` sub-steps: `useEffect` → `useLayoutEffect` from `@web/owl2/utils`;
`onWillRender`, `onRendered`, `useComponent`, `useEnv`, `useSubEnv`, `useChildSubEnv`,
`useRef`, `useExternalListener` imports moved from `@odoo/owl` to `@web/owl2/utils`;
`useState`/`reactive` → `proxy`; template `t-esc` → `t-out`, `t-ref` → `t-custom-ref`,
`t-model` → `t-custom-model`, `t-portal` → `t-custom-portal`, `t-slot` → `t-call-slot`;
`this.` prefix added to component variables in templates; parametric `t-call`; `useService(name)`
→ the matching plugin class (`SERVICES_MAPPING`, e.g. `action` → `ActionManagerPlugin`).

---

## Breaking Changes 19 → 20

### Security: `ir.access` replaces ACLs and record rules (19.4)

Models `ir.model.access` and `ir.rule` no longer exist; `ir.access` (`odoo/addons/base/models/ir_access.py`)
holds `model_id`, `group_id`, `operation` (subset of `crud`) and an optional `domain`.

```csv
# ❌ security/ir.model.access.csv (19)
id,name,model_id:id,group_id:id,perm_read,perm_write,perm_create,perm_unlink
access_my_model_user,my.model.user,model_my_model,base.group_user,1,1,1,0

# ✅ security/ir.access.csv (20), model_id is the model name
id,name,model_id,group_id/id,operation,domain
access_my_model_user,own records,my.model,base.group_user,cru,"[('user_id', '=', user.id)]"
access_my_model_manager,all records,my.model,my_module.group_manager,crud,
```

Semantics (`ir_access.py`): a record **with** a group is a *permission* (permissions are OR-ed;
the domain limits which records it grants); a record **without** group is a global
*restriction* (AND-ed for everyone). A 19 "full ACL + restricting rule" pair becomes one
permission row with the rule's domain. Run `19.4-00-ir-access.py`; a 19 ACL without group is
converted to `base.group_everyone`.

### Binary fields hold `BinaryValue` (19.2–20.0)

```python
# ❌ 19: base64 everywhere
attachment = env['ir.attachment'].create({'name': 'a.pdf', 'datas': base64.b64encode(pdf)})
data = base64.b64decode(record.image_1920)

# ✅ 20: raw bytes wrapped in BinaryValue (odoo/tools/binary.py)
from odoo.tools import BinaryBytes
attachment = env['ir.attachment'].create({'name': 'a.pdf', 'raw': pdf})   # 'datas' field removed
record.document = BinaryBytes(pdf, filename='a.pdf')
data = record.image_1920.content          # bytes; also .filename, .mimetype, .size, .to_base64()
```

Writing plain `bytes` to a Binary field other than `raw` raises
`TypeError: use BinaryValue instead of bytes`; a `str` is decoded as base64 (RPC format).
In XML data use `type="bytes"` (`type="base64"` warns `Since 20.0, use type=bytes instead of type=base64`).

### `ir.config_parameter` API (19.1)

```python
# ❌ 19 (removed)
ICP.get_param('my_module.limit', '10'); ICP.set_param('my_module.limit', 10)

# ✅ 20: typed getters/setters (odoo/addons/base/models/ir_config_parameter.py)
ICP = self.env['ir.config_parameter'].sudo()
limit = ICP.get_int('my_module.limit', 10)
ICP.set_str('my_module.api_url', url)   # also get/set_bool, get/set_float, get/set_str
```

### `_table_query` removed (20.0)

```python
# ❌ 19
_table_query = "SELECT ..."

# ✅ 20: override the _table_sql property (odoo/orm/models.py)
@property
def _table_sql(self):
    return SQL("(SELECT ... )")
```

### ormcache invalidation (19.4)

```python
# ❌ 19
self.env.registry.clear_cache('groups')

# ✅ 20 (one cache name per call; odoo/orm/environments.py)
self.env.transaction.invalidate_ormcache('groups')
```

`env.clear_access_cache()` is deprecated (`Since 20.0, renamed to invalidate_access_cache`).

### `request` in model code (19.4)

Core models moved away from `odoo.http.request`; use `self.env.website` (new property, the
`website` record from context `website_id`) and the context instead. `odoo/http.py` is now the
package `odoo/http/`.

### Others

| Change | ❌ 19 | ✅ 20 |
|--------|------|------|
| `odoo.osv` (removed) | `from odoo.osv import expression` | `from odoo.fields import Domain` |
| `_rec_names_search` (19.5) | `['name', 'ref']` | `('name', 'ref')` |
| QWeb `t-call` params (19.1) | `<t t-call="x"><t t-set="a" t-value="1"/></t>` | `<t t-call="x" a="1"/>` (only `t-call` with non-`t-` attributes; implicit t-set collection dropped) |
| Icons (19.5) | `<i class="fa fa-check"/>` | `<i class="oi" data-icon="check"/>` |
| Search date filter (20.0) | `<filter date="date" start_month="-2" .../>` | `<filter date="date"/>` (RNG rejects the range attrs) |
| OWL (20.0) | OWL 2, `useState`, `t-esc` | OWL `3.0.0-alpha`, `proxy`, `t-out` (see OWL guide) |
| Name copy (20.0) | override `copy()`/`copy_data()` to add " (copy)" | Char `name` fields default to `copy=mark_as_copy('name')`; remove the override |
| Test default tag (20.0) | `at_install` | `post_install`; `SingleTransactionCase` removed |
| Manifest i18n (20.0) | module summary/description terms in `base` `.po` | In the module's own `i18n/*.po` |

---

## Migration Checklist

- [ ] `upgrade_code --from 19.1` run and diff reviewed; `--script owl3-migration` for JS
- [ ] `__manifest__.py` `version` bumped; `security/ir.access.csv` listed in `data`
- [ ] No `ir.model.access.csv` / `ir.rule` records left; warnings of the ir-access script checked
- [ ] No `base64` encode/decode around Binary fields; `datas` → `raw`; XML `type="bytes"`
- [ ] `get_param`/`set_param` → `get_<type>`/`set_<type>`
- [ ] `_table_query` → `_table_sql`; `registry.clear_cache` → `transaction.invalidate_ormcache`
- [ ] No `odoo.osv`; `_rec_names_search` tuples; no `request` where `env.website`/context fits
- [ ] QWeb `t-call` parameters as attributes; icons as `oi` + `data-icon`
- [ ] Search filters without `start_month`/`end_year`...
- [ ] Custom " (copy)" logic removed where the default now applies
- [ ] Hooks referenced by function name; data migration scripts `migrate(cr, version)`
- [ ] Tests green on a fresh 20.0 database without demo data

---

## Testing Migrations

```bash
# restore a 19.0 dump, then update with the 20.0 code and run the module's tests
odoo-bin -d migrated_db -u my_module --test-tags /my_module --stop-after-init
```

```python
from odoo.tests import TransactionCase


class TestPostMigrate(TransactionCase):   # post_install by default in 20
    def test_states_filled(self):
        self.assertFalse(self.env['my.model'].search_count([('state', '=', False)]))
```

1. Restore a 19.0 production copy (neutralized)
2. Update all custom modules with the 20.0 code
3. Check the logs for `module X: Running upgrade` lines and warnings
4. Run the module tests and a manual pass on the main flows

---

## References

- `odoo/modules/migration.py`, `odoo/modules/loading.py`
- `odoo/cli/upgrade_code.py`, `odoo/upgrade_code/`
- Odoo 20 ORM changelog `/doc/developer/reference/backend/orm/changelog.rst`
