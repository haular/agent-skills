---
name: odoo-20
description: >-
  Odoo 20 development knowledge base with 18 specialized guides covering
  Actions (ir.actions.*, cron with _commit_progress, server actions),
  Controllers (odoo/http package, jsonrpc/json2 routes, bearer auth, /json/2
  external API), Data files (XML/CSV records, type="bytes" binaries,
  noupdate), API Decorators (@api.depends, @api.constrains, @api.ondelete,
  @api.private, @api.ormcache), Security (ir.access replacing
  ir.model.access/ir.rule, security/ir.access.csv, the access domain
  operator, _access_domain, privilege-based groups), Field types (BinaryValue
  binaries, compute_sql, copy callables, Json, Properties), ORM Model methods
  (Domain, typed ir.config_parameter getters, _table_sql, SQL builder,
  _read_group, ormcache), Manifest configuration, Mixins (mail.thread,
  mail.track.mixin tracking hooks, activities, website mixins), Migration
  (19 to 20 checklist, odoo-bin upgrade_code scripts incl. owl3-migration),
  OWL 3 frontend (useProps, signal/proxy, plugins, t-call-slot), Material
  Symbols icons, Performance (N+1, ormcache, compute_sql, populate CLI), QWeb
  Reports (pluggable PDF engines, t-call attributes, t-out), Testing
  (post_install default, MockHTTPClient, Hoot, tours), Transactions
  (savepoints, retrying, cursor API), Translations (env._, multi-language
  writes, manifest terms in module .po), XML Views (list/form/search, date
  filters, card views, dialog_size). Use when writing, reviewing, or
  debugging any Odoo 20 Python, XML or JS code, creating or modifying
  modules, migrating modules from Odoo 19, fixing performance issues, or
  looking up Odoo 20 API patterns and best practices. Includes CSS/SCSS
  asset authoring and review for Odoo addons.
globs: "**/*.{py,xml,csv,js,ts,css,scss}"
---

# Odoo 20 Skill - Master Index

Master index for all Odoo 20 development guides. Read the appropriate guide from `references/` based on your task.

## Quick Reference

| Topic          | File                                      | When to Use                                             |
| -------------- | ----------------------------------------- | ------------------------------------------------------- |
| Actions        | `references/odoo-20-actions-guide.md`     | Creating actions, menus, scheduled jobs, server actions |
| API Decorators | `references/odoo-20-decorator-guide.md`   | Using @api decorators, compute fields, validation       |
| Controllers    | `references/odoo-20-controller-guide.md`  | Writing HTTP endpoints, routes, web controllers         |
| Data Files     | `references/odoo-20-data-guide.md`        | XML/CSV data files, records, shortcuts                  |
| Development    | `references/odoo-20-development-guide.md` | Creating modules, manifest, reports, security, wizards  |
| Field Types    | `references/odoo-20-field-guide.md`       | Defining model fields, choosing field types             |
| Manifest       | `references/odoo-20-manifest-guide.md`    | **manifest**.py configuration, dependencies, hooks      |
| Migration      | `references/odoo-20-migration-guide.md`   | 19 → 20 upgrade, `upgrade_code`, migration scripts      |
| Mixins         | `references/odoo-20-mixins-guide.md`      | mail.thread, activities, email aliases, tracking        |
| Model Methods  | `references/odoo-20-model-guide.md`       | Writing ORM queries, CRUD operations, domain filters    |
| OWL Components | `references/odoo-20-owl-guide.md`         | Building OWL UI components, hooks, services             |
| Performance    | `references/odoo-20-performance-guide.md` | Optimizing queries, fixing slow code, preventing N+1    |
| Reports        | `references/odoo-20-reports-guide.md`     | QWeb reports, PDF/HTML, templates, paper formats        |
| Security       | `references/odoo-20-security-guide.md`    | `ir.access` rows, access domains, field permissions     |
| Testing        | `references/odoo-20-testing-guide.md`     | Writing tests, mocking, assertions, browser testing     |
| Transactions   | `references/odoo-20-transaction-guide.md` | Handling database errors, savepoints, UniqueViolation   |
| Translation    | `references/odoo-20-translation-guide.md` | Adding translations, localization, i18n                 |
| Views & XML    | `references/odoo-20-view-guide.md`        | Writing XML views, actions, menus, QWeb templates       |

## Coding Conventions

Before writing code, run the `odoo-workflow` skill if it is installed (trace the real
source, Context Brief with `file:line`, definition of done); install it alongside this pack.

Read `references/api-highlights.md` before writing code and apply its Quick review
checks: the ❌ items are what not to use in Odoo 20. Odoo 20 needs Python >= 3.12 and PostgreSQL >= 16.

Apply the convention section in the guide you open. Odoo 20 runtime behavior,
then existing stable-addon style, takes precedence; keep diffs focused.

- Module naming, assets, JavaScript, CSS, and SCSS: development guide.
- XML formatting and IDs: data, actions, reports, security, and views guides.
- Python, recordsets, transactions, and translations: model, field, decorator,
  performance, transaction, and translation guides.

## File Structure

```
skills/odoo-20.0/
├── SKILL.md                          # This file - master index
└── references/                       # Development guides
    ├── odoo-20-actions-guide.md
    ├── odoo-20-controller-guide.md
    ├── odoo-20-data-guide.md
    ├── odoo-20-decorator-guide.md
    ├── odoo-20-development-guide.md
    ├── odoo-20-field-guide.md
    ├── odoo-20-manifest-guide.md
    ├── odoo-20-migration-guide.md
    ├── odoo-20-mixins-guide.md
    ├── odoo-20-model-guide.md
    ├── odoo-20-owl-guide.md
    ├── odoo-20-performance-guide.md
    ├── odoo-20-reports-guide.md
    ├── odoo-20-security-guide.md
    ├── odoo-20-testing-guide.md
    ├── odoo-20-transaction-guide.md
    ├── odoo-20-translation-guide.md
    └── odoo-20-view-guide.md
```

## Base Code Reference (Odoo 20)

All guides are based on analysis of Odoo 20 source code:

- `odoo/orm/models.py` - ORM implementation
- `odoo/orm/fields.py` - Field types
- `odoo/orm/decorators.py` - Decorators
- `odoo/orm/domains.py` - Domain
- `odoo/orm/cache.py` - ormcache
- `odoo/http/` - HTTP layer (package)
- `odoo/addons/base/models/ir_access.py` - Access control (`ir.access`)
- `odoo/upgrade_code/` - 19.x to 20.0 source rewrite scripts
- `addons/web/static/lib/owl/owl.js` - OWL 3
- `odoo/exceptions.py` - Exception types
- `odoo/tools/translate.py` - Translation system
- `odoo/addons/base/models/res_lang.py` - Language model
- `addons/web/static/src/core/l10n/translation.js` - JS translations

## External Documentation

- [Odoo 20 Official Documentation](https://github.com/odoo/documentation/tree/20.0)
- [Odoo 20 ORM Changelog](https://github.com/odoo/documentation/blob/20.0/content/developer/reference/backend/orm/changelog.rst)
- [Odoo Coding Guidelines](https://raw.githubusercontent.com/odoo/documentation/17.0/content/contributing/development/coding_guidelines.rst)
