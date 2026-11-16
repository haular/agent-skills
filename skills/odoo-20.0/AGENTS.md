# Odoo 20 Documentation - AI Agents Setup

Setup guide for using Odoo 20 documentation with AI coding assistants (Cursor, Claude Code, Windsurf, Aider, etc.).

## Quick Start

### Install via skills.sh (Recommended)

```bash
# Add Odoo 20 skill to your project
npx skills add unclecatvn/agent-skills
```

Visit [https://skills.sh/](https://skills.sh/) for more installation options.

### Cursor IDE - Remote Rule

Configure once in Cursor settings:
- `Settings` → `Rules` → `Add Remote Rule`
- Source: `Git Repository`
- URL: `git@github.com:unclecatvn/agent-skills.git`
- Branch: `20.0`
- Subfolder: `skills/odoo-20.0/`

---

## Documentation Structure

```
skills/odoo-20.0/
├── SKILL.md                       # Master index (all agents)
├── references/                    # Development guides (18 files)
│   ├── odoo-20-actions-guide.md     # ir.actions.*, cron, bindings
│   ├── odoo-20-controller-guide.md  # HTTP, routing, controllers
│   ├── odoo-20-data-guide.md        # XML/CSV data files, records
│   ├── odoo-20-decorator-guide.md   # @api decorators
│   ├── odoo-20-development-guide.md # Manifest, wizards (overview)
│   ├── odoo-20-field-guide.md       # Field types, parameters
│   ├── odoo-20-manifest-guide.md    # __manifest__.py reference
│   ├── odoo-20-mixins-guide.md      # mail.thread, activities, etc.
│   ├── odoo-20-model-guide.md       # ORM, CRUD, search, domain
│   ├── odoo-20-migration-guide.md   # Migration scripts, hooks
│   ├── odoo-20-owl-guide.md         # OWL components, services
│   ├── odoo-20-performance-guide.md # N+1 prevention, optimization
│   ├── odoo-20-reports-guide.md     # QWeb reports, PDF/HTML
│   ├── odoo-20-security-guide.md    # ACL, record rules, security
│   ├── odoo-20-testing-guide.md     # Test classes, decorators
│   ├── odoo-20-transaction-guide.md # Savepoints, errors
│   ├── odoo-20-translation-guide.md # Translations, i18n
│   └── odoo-20-view-guide.md        # XML views, QWeb
├── CLAUDE.md                      # Claude Code specific
└── AGENTS.md                      # THIS FILE - setup guide
```

---

## Guide Reference

| File | Purpose | When to Use |
|------|---------|-------------|
| `SKILL.md` | Master index for all guides | Find the right guide for your task |
| `references/odoo-20-actions-guide.md` | Actions (window, URL, server, cron) | Creating actions, menus, scheduled jobs |
| `references/odoo-20-controller-guide.md` | HTTP controllers, routing | Writing endpoints |
| `references/odoo-20-data-guide.md` | XML/CSV data files, records | Creating data files |
| `references/odoo-20-decorator-guide.md` | @api decorators usage | Using @api decorators |
| `references/odoo-20-development-guide.md` | Module structure, wizards | Creating new modules |
| `references/odoo-20-field-guide.md` | Field types, parameters | Defining model fields |
| `references/odoo-20-manifest-guide.md` | __manifest__.py reference | Configuring module manifest |
| `references/odoo-20-mixins-guide.md` | mail.thread, activities, mixins | Adding messaging, activities |
| `references/odoo-20-model-guide.md` | ORM methods, CRUD, domains | Writing model methods |
| `references/odoo-20-migration-guide.md` | Migration scripts, hooks | Upgrading modules |
| `references/odoo-20-owl-guide.md` | OWL components, hooks, services | Building OWL UI |
| `references/odoo-20-performance-guide.md` | Performance optimization | Fixing slow code |
| `references/odoo-20-reports-guide.md` | QWeb reports, templates | Creating reports |
| `references/odoo-20-security-guide.md` | ACL, record rules, security | Configuring security |
| `references/odoo-20-testing-guide.md` | Test classes, decorators, mocking | Writing tests |
| `references/odoo-20-transaction-guide.md` | Database transactions, error handling | Savepoints, UniqueViolation |
| `references/odoo-20-translation-guide.md` | Translations, localization, i18n | Adding translations |
| `references/odoo-20-view-guide.md` | XML views, actions, menus | Writing view XML |

---

## AI Agent Configuration

### Cursor IDE

| Setting | Value |
|---------|-------|
| Source | Git Repository |
| URL | `git@github.com:unclecatvn/agent-skills.git` |
| Branch | `20.0` |
| Subfolder | `skills/odoo-20.0/` |

**Globs patterns used by Cursor:**

| File | globs Pattern |
|------|---------------|
| `SKILL.md` | `**/*.{py,xml,csv,js,ts,css,scss}` |
| `references/odoo-20-actions-guide.md` | `**/*.{py,xml}` |
| `references/odoo-20-controller-guide.md` | `**/controllers/**/*.py` |
| `references/odoo-20-data-guide.md` | `**/*.{xml,csv}` |
| `references/odoo-20-decorator-guide.md` | `**/models/**/*.py` |
| `references/odoo-20-development-guide.md` | `**/*.{py,xml,csv,js,ts,css,scss}` |
| `references/odoo-20-field-guide.md` | `**/models/**/*.py` |
| `references/odoo-20-manifest-guide.md` | `**/__manifest__.py` |
| `references/odoo-20-mixins-guide.md` | `**/models/**/*.py` |
| `references/odoo-20-model-guide.md` | `**/models/**/*.py` |
| `references/odoo-20-migration-guide.md` | `**/migrations/**/*.py` |
| `references/odoo-20-owl-guide.md` | `static/src/**/*.{js,xml}` |
| `references/odoo-20-performance-guide.md` | `**/*.{py,xml}` |
| `references/odoo-20-reports-guide.md` | `**/report/**/*.xml` |
| `references/odoo-20-security-guide.md` | `**/security/**/*.{csv,xml}` |
| `references/odoo-20-testing-guide.md` | `**/tests/**/*.py` |
| `references/odoo-20-transaction-guide.md` | `**/models/**/*.py` |
| `references/odoo-20-translation-guide.md` | `**/*.{py,js,xml}` |
| `references/odoo-20-view-guide.md` | `**/views/**/*.xml` |

### Claude Code

```bash
# Install via skills.sh
npx skills add unclecatvn/agent-skills
```

Claude Code reads:
- `CLAUDE.md` - Project overview and quick reference
- `SKILL.md` - Master index for all guides
- Individual guides in `references/` - Detailed information

### Other Agents

| Agent | Setup |
|-------|-------|
| Windsurf | Same as Cursor (uses `.mdc` files) |
| Continue | Place `CLAUDE.md` or `SKILL.md` in root |
| Aider | Place `CLAUDE.md` or add to prompt |
| OpenCode | Copy skill folder to project - no additional config needed |

---

## Cursor / Claude Skills Folder

After installing via `npx skills add unclecatvn/agent-skills`, the skill is placed at:

```
.cursor/skills/
└── odoo-20/
    └── SKILL.md

.claude/skills/
└── odoo-20/
    └── SKILL.md
```

---

## Key Odoo 20 Changes

| Change | Old (19) | New (20) |
|--------|----------|----------|
| Access control | `ir.model.access.csv` + `ir.rule` | `security/ir.access.csv` (`ir.access`) |
| Binary fields | base64 strings, `ir.attachment.datas` | `BinaryValue` / `BinaryBytes`, `raw` |
| Config parameters | `get_param()` / `set_param()` | `get_str()` / `get_int()` / `get_bool()` / `get_float()` |
| Domain helpers | `odoo.osv.expression` | `odoo.fields.Domain` |
| Icons | `fa fa-*` | `<i class="oi" data-icon="..."/>` |
| Frontend | OWL 2 | OWL 3 |

Full list: `references/api-highlights.md`.

## Coding Conventions

Before writing code, run the `odoo-workflow` skill if it is installed (trace the real
source, Context Brief with `file:line`, definition of done); install it alongside this pack.

Read `references/api-highlights.md` before writing code and apply its Quick review
checks: the ❌ items are what not to use in Odoo 20.

Apply the Coding Conventions section in the guide you open. Odoo 20 runtime
behavior takes precedence, followed by the surrounding stable-addon style;
keep convention-only changes focused. The common source is the [Odoo Coding
Guidelines](https://raw.githubusercontent.com/odoo/documentation/17.0/content/contributing/development/coding_guidelines.rst).

- Use the development guide for module layout, asset, JavaScript, CSS, and
  SCSS conventions.
- Use the data, action, report, security, and view guides for XML formatting
  and external IDs.
- Use the model, field, decorator, performance, transaction, and translation
  guides for Python, recordsets, context, database work, and i18n.

---

## Repository

**URL**: `git@github.com:unclecatvn/agent-skills.git`
**Branch**: `20.0`
**License**: MIT
