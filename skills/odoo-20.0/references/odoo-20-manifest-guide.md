# Odoo 20 Manifest Guide

## Coding Conventions

Load groups before the files that reference them, actions before menus, and the
`security/ir.access.csv` file last in `data` (as Odoo 20 addons do); use focused model-based filenames.

Guide for configuring `__manifest__.py` in Odoo 20 modules (Python ≥ 3.12, PostgreSQL ≥ 16,
`odoo/release.py`).

## Table of Contents
- [Manifest File](#manifest-file)
- [Required Fields](#required-fields)
- [Module Information](#module-information)
- [Dependencies](#dependencies)
- [Data Files](#data-files)
- [Assets](#assets)
- [Hooks](#hooks)
- [External Dependencies](#external-dependencies)
- [Auto-Install](#auto-install)
- [Accepted Keys](#accepted-keys)
- [Manifest Translations](#manifest-translations)

---

## Manifest File

The manifest file declares a python package as an Odoo module and specifies module metadata.
Parsing/validation: `odoo/modules/module.py` (`_DEFAULT_MANIFEST`, `_load_manifest`).

File: `__manifest__.py`

```python
{
    'name': "A Module",
    'version': '1.0',
    'depends': ['base'],
    'author': "Author Name",
    'license': 'LGPL-3',
    'category': 'Category',
    'data': [
        'views/my_model_views.xml',
        'security/ir.access.csv',
    ],
    'demo': [
        'demo/demo_data.xml',
    ],
}
```

---

## Required Fields

### name (str, required)

The human-readable name of the module (stored as `ir.module.module.shortdesc`).

```python
'name': "My Module",
```

### author, license (str, expected)

Not hard-required, but a missing value logs a warning and is defaulted:

- `author` → `contributors` or `maintainer` value, else `''`
  (`"Missing `author` key in manifest for %r, defaulting to %r"`)
- `license` → `'LGPL-3'` (`"Missing `license` key in manifest for %r, defaulting to LGPL-3"`)

---

## Module Information

### version (str, default: `'1.0'`)

Accepted formats: `x.y`, `x.y.z`, `20.0.x.y` or `20.0.x.y.z`. Short versions are prefixed
with the series (`'1.0'` → `'20.0.1.0'`). A version for another series (e.g. `'19.0.1.0'`)
logs "has an incompatible version, setting installable=False" and the module is not installable.

```python
'version': '1.0.0',
```

### summary (str)

One-line short description (the manifest linter rejects newlines).

### description (str, default: README)

Extended description in reStructuredText. If empty, Odoo reads `README.rst`, `README.md`,
`README.txt` or `README` from the module folder.

### author / website / maintainer (str)

```python
'author': "UncleCat",
'website': "https://github.com/unclecat",   # https:// URL
```

### license (str, default: LGPL-3)

Possible values: `GPL-2`, `GPL-2 or any later version`, `GPL-3`, `GPL-3 or any later version`,
`AGPL-3`, `LGPL-3`, `Other OSI approved licence`, `OEEL-1`, `OPL-1`, `Other proprietary`.

### category (str, default: Uncategorized)

Use existing categories or create hierarchies with `/`:

```python
'category': 'Tools / My Category',
```

### application (bool, default: False)

```python
'application': True,  # Appears as an app in the Apps menu
```

### installable (bool, default: True)

```python
'installable': False,  # Cannot be installed
```

### Other informative keys

`icon` (path, default `/<module>/static/description/icon.png`), `images`, `sequence`
(default 100), `countries` (list of 2-letter codes, for localizations), `iap_paid_service`
(bool, new default key in 20: module contains In-App Purchases).

---

## Dependencies

### depends (list(str))

```python
'depends': ['base', 'web', 'sale'],
```

An empty `depends` is replaced by `['base']` (except for `base` itself). All dependencies are
installed first.

---

## Data Files

### data (list(str))

Loaded at installation and update, in list order.

```python
'data': [
    'security/my_module_groups.xml',      # res.groups / res.groups.privilege
    'data/my_model_data.xml',
    'views/my_model_views.xml',
    'report/my_model_reports.xml',
    'views/my_module_menus.xml',
    'security/ir.access.csv',             # ir.access (replaces ir.model.access.csv + ir.rule, 19.4)
],
```

❌ `security/ir.model.access.csv` and `ir.rule` records — those models no longer exist in 20;
✅ one `security/ir.access.csv` (see security guide). The 19.4 upgrade script appends it
at the end of `data`, which is where `sale`, `crm`, `project`, `utm` list it.

### demo (list(str))

Only loaded in demonstration mode. New databases are created **without** demo data unless
the server runs with `--with-demo` (`--without-demo` is the default).

```python
'demo': [
    'demo/demo_data.xml',
],
```

### other_files (list(str), new in 20)

Data files shipped in the module but not loaded by `data`/`demo` (e.g. `hr` declares
`'data/scenarios/hr_scenario.xml'`). The manifest linter (`test_lint/tests/test_manifests.py`)
treats `data`, `demo` and `other_files` as the data-file keys.

❌ Removed keys (no longer in `_DEFAULT_MANIFEST`; flagged as unknown by the linter):
`init_xml`, `update_xml`, `demo_xml`, `images_preview_theme`.

### populate/ (not a manifest key)

Synthetic-data blueprints (`populate.blueprint` records) live in the module's `populate/`
directory; they are loaded when the `populate` module is upgraded, not through the manifest.

---

## Assets

### assets (dict)

```python
'assets': {
    'web.assets_backend': [
        'my_module/static/src/**/*',
        ('remove', 'my_module/static/src/dashboard/**/*'),
    ],
    'my_module.dashboard': [               # custom bundle, lazy-loaded
        'my_module/static/src/dashboard/**/*',
    ],
    'web.assets_frontend': [
        'my_module/static/src/js/frontend.js',
    ],
},
```

Directives (`odoo/addons/base/models/ir_asset.py`): `append` (default), `prepend`, `after`,
`before`, `remove`, `replace`, `include`:

```python
('include', 'web._assets_helpers'),
('after', 'web/static/src/core/x.js', 'my_module/static/src/y.js'),
('replace', 'web/static/src/a.js', 'my_module/static/src/a.js'),
```

The web client bundles load OWL 3 (`web/static/lib/owl/owl.js`) followed by the OWL 2
compatibility layer (`web/static/src/owl2/*`); icons come from `web.icons_fonts`
(Material Symbols + `odoo_ui_icons`; Font Awesome was dropped in 19.5).

---

## Hooks

### pre_init_hook, post_init_hook, uninstall_hook, post_load (str)

Names of functions exported by the module's `__init__.py`
(`odoo/modules/loading.py`, `odoo/modules/module.py`).

```python
# In __init__.py
def pre_init_hook(env):
    """Before module installation (models of the module are not loaded yet)."""

def post_init_hook(env):
    """Right after module installation."""

def uninstall_hook(env):
    """After module uninstallation."""

def post_load():
    """Called with no argument when the module's Python package is imported."""
```

```python
# In __manifest__.py
'pre_init_hook': 'pre_init_hook',
'post_init_hook': 'post_init_hook',
'uninstall_hook': 'uninstall_hook',
'post_load': 'post_load',
```

**Usage**: only when setup/cleanup is extremely difficult or impossible through data files.

---

## External Dependencies

### external_dependencies (dict(key=list(str)))

```python
'external_dependencies': {
    'python': ['requests', 'openpyxl'],   # PyPI requirement names (version specifiers allowed)
    'bin': ['zip', 'unzip'],
},
```

Missing dependencies prevent installation; a Python name that is not a valid PyPI package logs
"does not appear to be a valid PyPI package".

---

## Auto-Install

### auto_install (bool or list(str), default: False)

```python
'auto_install': True,             # install when all `depends` are installed
'auto_install': ['sale', 'crm'],  # install when these (subset of depends) are installed
'auto_install': [],               # always install
```

A list containing a module that is not in `depends` fails the assertion
"auto_install triggers must be dependencies".

---

## Accepted Keys

`_DEFAULT_MANIFEST` keys in 20.0: `application`, `assets`, `auto_install`, `bootstrap`,
`category`, `cloc_exclude`, `configurator_snippets`, `configurator_snippets_addons`, `countries`,
`data`, `demo`, `depends`, `description`, `external_dependencies`, `iap_paid_service`, `images`,
`installable`, `kpi_providers`, `live_test_url`, `new_page_templates`, `other_files`,
`post_init_hook`, `post_load`, `pre_init_hook`, `sequence`, `summary`, `test`,
`theme_customizations`, `uninstall_hook`, `version`, `web`, `website`.
Also accepted by the linter: `name`, `icon`, `author`, `license`, `contributors`, `maintainer`,
`url`, `price`, `currency`, `support`.

---

## Manifest Translations

Since 20.0 the translations of `name`, `summary` and `description` are exported to and read
from the **module's own** `i18n/<lang>.po` (references
`model:ir.module.module,shortdesc|summary|description:base.module_<module>`), no longer from
`base`'s PO files (`Manifest.get_translations`, `odoo/tools/translate.py` export). Run the
module's i18n export to get these terms in its `.pot`.

---

## Complete Example

```python
{
    'name': "My Awesome Module",
    'version': '1.0.0',
    'category': 'Tools',
    'summary': 'Does something awesome',
    'description': """
My Awesome Module
=================

This module adds awesome functionality to Odoo.
""",
    'author': "UncleCat",
    'website': "https://github.com/unclecat/my_module",
    'license': 'LGPL-3',
    'depends': ['base', 'web'],
    'data': [
        'security/my_module_groups.xml',
        'data/my_model_data.xml',
        'views/my_model_views.xml',
        'report/my_model_templates.xml',
        'report/my_model_reports.xml',
        'views/my_module_menus.xml',
        'security/ir.access.csv',
    ],
    'demo': [
        'demo/demo_data.xml',
    ],
    'assets': {
        'web.assets_backend': [
            'my_module/static/src/**/*',
        ],
    },
    'external_dependencies': {
        'python': ['requests'],
    },
    'application': True,
}
```

---

## References

- Source: `odoo/modules/module.py`, `odoo/modules/loading.py`, `odoo/addons/base/models/ir_asset.py`
- Linter: `odoo/addons/test_lint/tests/test_manifests.py`
- Documentation: `content/developer/reference/backend/module.rst` (20.0)
