---
name: odoo-20-api-highlights
description: Version-distinguishing API patterns for Odoo 20. Read this when the target version is 20.0 so the reviewer/tracer applies the right rules.
---

# Odoo 20 API Highlights

Use this file as the version-specific ruleset when the resolved Odoo version is `20.0`. It supplements — not replaces — the general review checklist. Everything from 19 applies unless noted below. Odoo 20 bundles the Odoo Online 19.1–19.5 changes, so the version a change landed in is given in parentheses when known.

Runtime: Python >= 3.12, PostgreSQL >= 16 (`odoo/release.py`).

## Security: `ir.access` (19.4)

- `ir.model.access` and `ir.rule` are **gone**. One model, `ir.access`, holds both: `name`, `model_id`, `group_id`, `operation` (letters from `crud`), `domain`, `active` (`odoo/addons/base/models/ir_access.py`).
- File: `security/ir.access.csv`, header `id,name,model_id,group_id/id,operation,domain`. `model_id` is the **model name** (`sale.order`), not `model_sale_order`.
  ```csv
  id,name,model_id,group_id/id,operation,domain
  access_my_model_user,my.model user,my.model,base.group_user,crud,"[('company_id', 'in', company_ids)]"
  ```
- Semantics: rows with a group are **permissions** (OR-ed); rows without a group are **restrictions** (AND-ed). No permission → no access. `odoo/orm/models.py` `_access_domain`.
- Domain eval context: `user`, `company_id`, `company_ids`, `time` (+ `website` with the website module).
- New domain operator `access` (19.3): `[('move_id', 'access', 'read')]` — many2one or `id` only, operations read/write/create/unlink.
- Disable another module's access row: `<record id="product.product_pricelist_comp_rule" model="ir.access"><field name="active" eval="False"/></record>`.
- Convert a 19 module: `odoo-bin upgrade_code --script 19.4-00-ir-access` (not idempotent: run once).
- Check API: `check_access()` / `has_access()` / `_filtered_access()`. `check_access_rights()`, `check_access_rule()`, `_filter_access_rules()` are **removed**. Overriding `_check_access` is deprecated ("Since 20.0, use Model._access_domain instead"): override `_access_domain`. `_check_field_access` deprecated → `check_field_access` / `has_field_access`, override `_has_field_access`.
- Reference: `references/odoo-20-security-guide.md`.

## Models and ORM

- `_table_query` **removed** (20.0) → a `_table_sql` property returning `SQL` (see `addons/sale/report/sale_report.py`).
- `_rec_names_search` is a tuple (19.5): `('name', 'default_code')`.
- `concat()` / `union()` take one iterable and are called on an empty recordset: `Model.concat(recs)` (20.0 warns otherwise).
- x2many reads return only records the user can read (19.3), unless sudo or `bypass_search_access`.
- Field updates and inverses run in `(field.write_sequence, field index)` order in `create()` and `write()` (19.4).
- Domains: `'<>'`, `'=='` and upper-case operators raise "Invalid operator" (20.0). `odoo.osv` is **removed**: `from odoo.fields import Domain`.
- `read_group()` is final with a new signature `(domain, groupby, aggregates, having, offset, limit, order)` returning tuples. Use `_read_group()` in backend code, `formatted_read_group()` for the old formatted output.
- SQL builder (19.1): `from odoo.models import Query`; `query.table` is a `TableSQL` (`t.field`, `t.m2o.field`); `env.execute_query(SQL(...))`. String arguments to `select`/`order` warn.
- `ir.config_parameter` (19.1): `get_param` / `set_param` are **removed** → `get_str/get_int/get_float/get_bool(key, default)` and `set_str/set_int/set_float/set_bool`.
- ormcache (19.4/20.0): `@api.ormcache(...)` (importing from `odoo.tools` warns "Since 20.0 import ormcache from odoo.api"); `registry.clear_cache(name)` removed → `self.env.transaction.invalidate_ormcache(name)`; `ormcache_context` removed; `models.CachedModel` new.
- `env.clear()` deprecated → `env.transaction.clear()` / `reset()`. `self._cr` / `_uid` / `_context` still deprecated.
- Removed: `toggle_active` (use `action_archive` / `action_unarchive`), `_check_recursion` (use `_has_cycle`), `copy_translations`. An extension class cannot set `_table` (TypeError).
- New model attribute `_explanation`.
- Reference: `references/odoo-20-model-guide.md`.

## Fields

- **Binary fields hold `BinaryValue`** (19.2–20.0), not base64: write `BinaryBytes(data, filename=...)` (`from odoo.tools import BinaryBytes`), read `rec.f.content` / `rec.f.to_base64()`. Writing plain `bytes` raises `TypeError` (except `ir.attachment.raw`). RPC `read()` returns `{'content': b64, 'size': n[, 'filename']}`.
- `ir.attachment.datas` is **removed** → `raw`.
- `compute_sql` (19.1): a non-stored compute becomes searchable/sortable/groupable via `compute_sql='_compute_sql_x'` (`def _compute_sql_x(self, table) -> SQL`); set `compute_sudo` explicitly.
- `copy=` accepts a callable (20.0); `copy=mark_as_copy('title')`. A Char `name` / `x_name` defaults to " (copy)": drop `copy_data` overrides that only add it, or the suffix doubles.
- `init_storage='_init_x'` (20.0) replaces overriding `_init_column`.
- Several languages in one write (19.3): `rec.write({'name': {'en_US': 'Fork', 'fr_FR': 'Fourchette'}})`.
- Reference: `references/odoo-20-field-guide.md`.

## Decorators

- `@api.ormcache` is exported by `odoo.api`; `@api.deprecated` is not (use `odoo.tools.func.deprecated`). `@api.returns` still absent.
- Reference: `references/odoo-20-decorator-guide.md`.

## Views and QWeb

- **Icons (19.5)**: Font Awesome is out of the bundles. ❌ `<i class="fa fa-user"/>`, `icon="fa-envelope"` → ✅ `<i class="oi" data-icon="person"/>`, `icon="mail" icon_class="oi-filled"` (Material Symbols names). Migrate: `odoo-bin upgrade_code --script 19.5-00-material-symbols-icons`.
- Search date filters (20.0): `start_month` / `end_month` / `start_year` / `end_year` are rejected by the RNG; keep `default_period` (now also `today`, `this_week`, `this_month`, `this_quarter`, `this_year`). New `end_date="date_stop"` range filter.
- New: root attribute `dialog_size`, list `<column>` stacked cell, `card` view type (`<kanban card_id="%(xmlid)d">`), button `confirm-title`. Calendar `date_delay` removed (use `date_stop`).
- Server QWeb: `t-esc` and `t-raw` are **removed** (logged as unknown, render nothing) → `t-out`. OWL templates still accept `t-esc` with a warning.
- `t-call` parameters (19.1): ❌ `<t t-call="x"><t t-set="title">Events</t></t>` → ✅ `<t t-call="x" title.translate="Events"/>`, `var="expr"`, `var.f="literal"`. Inner `t-set` no longer reaches the callee.
- Reference: `references/odoo-20-view-guide.md`.

## Data files

- Binary file data: ❌ `type="base64" file="..."` (warns "Since 20.0, use type=bytes instead of type=base64") → ✅ `type="bytes"`.
- `<template technical_usage="...">` is new. XML eval `pytz` is a shim over `zoneinfo`.
- Reference: `references/odoo-20-data-guide.md`.

## Actions and cron

- `ir.actions.act_window.views` and `ir.actions.client.params` are `fields.Json`. New `ir.actions.actions.explanation`.
- Server action code: `timezone` is `ZoneInfo`; `BinaryBytes` available; no `Warning` (raise `UserError`).
- Cron: `_commit_progress()` (and new `_rollback_progress()`); `_notify_progress` deprecated; `_trigger(at=None, *, coalesce=minutes)`; minimum time per job 120 s.
- Reference: `references/odoo-20-actions-guide.md`.

## Reports

- PDF engines are modules (19.4): `base_report_wkhtmltox` (auto-installed) and `base_report_paper_muncher`; report type `qweb-pdf-<engine>` or system parameter `report.pdf_engine_default`. ❌ `_run_wkhtmltopdf` → ✅ `_run_pdf_engine` / `_run_pdf_engine_without_processing`.
- `ir.actions.report.report_file` removed. Report attachments use `raw`.
- Reference: `references/odoo-20-reports-guide.md`.

## Mixins

- Tracking engine moved to `mail.track.mixin` (inherited by `mail.thread`). Renamed hooks: `_track_subtype` → `_track_log_get_default_subtype`, `_track_template` → `_track_template_parameters`, `_track_get_default_log_message` → `_track_log_get_default_body`, `_track_set_author` → `_track_set_log_author`; `_track_filter_for_display` removed.
- `activity_schedule(..., activity_user_id_fname=...)`. `website.published.mixin` adds `publish_on` / `published_date`.
- Reference: `references/odoo-20-mixins-guide.md`.

## HTTP and controllers

- `odoo/http/` is a package (19.4); its root exports only `request`, `request_var`, `Response`, `Controller`, `route`. ❌ `from odoo.http import content_disposition, Stream` → ✅ `from odoo.http.stream import content_disposition, Stream`. `retrying` → `odoo.http.retrying`.
- Route types: `http`, `jsonrpc`, `json2`; `type='json'` is a deprecated alias of `jsonrpc`. `auth='bearer'` requires `bearer_scope`.
- `request.cr` / `uid` / `context` deprecated → `request.env.*`. In model code use `self.env.website`, not `get_current_website()` / `request.website`.
- New h11 HTTP server (19.4). `--http-interface` defaults to `127.0.0.1` (20.0); new `--gevent-workers`.
- External API: XML-RPC / JSON-RPC are deprecated and the `db` service is removed; use `POST /json/2/<model>/<method>`.
- Reference: `references/odoo-20-controller-guide.md`, `references/odoo-20-transaction-guide.md`.

## Frontend (OWL 3)

- `web/static/lib/owl` is OWL **3** (`3.0.0-alpha`), loaded with a compatibility layer (`web/static/src/owl2/`). Migrate components with `odoo-bin upgrade_code --script owl3-migration` (`--from` does not pick it up; check its `ActionManagerPlugin` rewrite: the real export is `ActionPlugin`).
- Props: ❌ `static props = {...}` / `static defaultProps` (throws) → ✅ `props = useProps({ x: t.number().optional(1) })`.
- State: ❌ `useState()` / `reactive()` → ✅ `proxy({...})` or `signal(value)`; `computed`, `untrack`, `markRaw`.
- Refs: ❌ `useRef("x")` + `t-ref="x"` → ✅ `x = signal.ref()` + `t-ref="this.x"`, read `this.x()`.
- Effects: `useEffect(fn)` tracks signals itself; the old deps form is `useLayoutEffect` from `@web/owl2/utils`. ❌ `useExternalListener` → ✅ `useListener`.
- Templates: component members need `this.` (`this.props.x`, `this.state.y`); ❌ `t-slot` → ✅ `t-call-slot`; `t-model="this.sig"` (signal) or `t-model.proxy="this.state.x"`; `t-portal` → `t-custom-portal`; `t-esc` still renders in OWL but warns → `t-out`.
- Services → plugins: `usePlugin(ORM)` from `@web/core/orm_plugin` (`useService` kept as an alias); `rpc`, `user`, `router` are module imports. `orm.create(model, [vals])` takes a list; `notification.add(message, options)`.
- Icons: `<i class="oi" data-icon="star"/>`; SCSS ❌ `.fa-star` → ✅ `[data-icon='star']`.
- Reference: `references/odoo-20-owl-guide.md`.

## Manifest and CLI

- Removed manifest keys: `init_xml`, `update_xml`, `demo_xml`, `images_preview_theme`. New: `other_files`, `iap_paid_service`. List `security/ir.access.csv` in `data`.
- `odoo-bin duplicate` is the old `populate` (copies existing records); the new `odoo-bin populate -b BLUEPRINT` comes from the `populate` addon.
- `odoo-bin scaffold` still generates `ir.model.access.csv`: replace it with `security/ir.access.csv`.
- Reference: `references/odoo-20-manifest-guide.md`, `references/odoo-20-development-guide.md`.

## Tests and tooling

- Default test tags are `standard` + `post_install` (20.0; 19 defaulted to `at_install`): `@tagged('-at_install', 'post_install')` is now redundant.
- Removed: `SingleTransactionCase`, `odoo.tests.common.Form` (use `odoo.tests.Form`), `assertQueriesContain`. New: `MockHTTPClient`; `file_read` returns `BinaryBytes`.
- Tours: `registry.category("web_tour.tours").add(name, {steps: () => [...]})`.
- Manifest terms (name, summary, description) are translated in the module's own `.po`, no longer in `base`.
- Reference: `references/odoo-20-testing-guide.md`, `references/odoo-20-migration-guide.md`, `references/odoo-20-translation-guide.md`.

## Quick review checks (v20-specific)

- ❌ `ir.model.access.csv` / `ir.rule` records → `security/ir.access.csv` (`model_id` = model name).
- ❌ `check_access_rights()` / `check_access_rule()` → `check_access()` / `has_access()`; customise with `_access_domain`.
- ❌ `base64.b64encode` / `b64decode` around Binary fields, `ir.attachment.datas` → `BinaryBytes`, `.content`, `raw`.
- ❌ `type="base64"` in XML data → `type="bytes"`.
- ❌ `get_param` / `set_param` → typed `get_*` / `set_*`.
- ❌ `_table_query` → `_table_sql`.
- ❌ `from odoo.osv import expression` → `Domain`.
- ❌ `registry.clear_cache(...)`, `tools.ormcache` → `env.transaction.invalidate_ormcache(...)`, `api.ormcache`.
- ❌ `fa fa-*` icons → `oi` + `data-icon`.
- ❌ `t-esc` / `t-raw`, `t-set` inside `t-call` → `t-out`, `t-call` attributes.
- ❌ `start_month` / `end_year`… on date filters.
- ❌ `from odoo.http import Stream, content_disposition` → `odoo.http.stream`.
- ❌ `toggle_active()`, `_check_recursion()`, `concat(a, b)`.
- ❌ `_track_subtype` / `_track_template` overrides → renamed hooks.
- ❌ `static props`, `useState`, `useRef`, `t-slot`, bare `props.x` in OWL templates → `useProps`, `proxy` / `signal`, `signal.ref()`, `t-call-slot`, `this.props.x`.
- All 19 rules still apply (`models.Constraint` / `Index`, no `_sql_constraints`, `_read_group`, explicit `_name`, `@api.private`, `<list>`, `aggregator=`, no demo data by default).
