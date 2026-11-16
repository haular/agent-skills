# Odoo 20 Testing Guide

Guide for testing Odoo 20: Python unit tests, JS tests (Hoot), tours (integration tests), and
synthetic data with `populate`.

## Table of Contents
- [Test Types](#test-types)
- [Python Tests](#python-tests)
- [Test Classes](#test-classes)
- [Test Decorators](#test-decorators)
- [Test Selection](#test-selection)
- [Test Utilities](#test-utilities)
- [JS Tests](#js-tests)
- [Integration Tests (Tours)](#integration-tests-tours)
- [Performance Testing](#performance-testing)
- [Synthetic Data (populate)](#synthetic-data-populate)
- [Best Practices](#best-practices)

---

## Test Types

Odoo has three kinds of tests:

| Type | Purpose |
|------|---------|
| **Python unit tests** | Test model business logic |
| **JS unit tests** | Test JavaScript code in isolation (Hoot) |
| **Tours** | Integration testing (Python + JS together) |

---

## Python Tests

### Test Structure

Create a `tests` sub-package in your module:

```
your_module/
├── ...
├── tests/
│   ├── __init__.py
│   ├── test_bar.py
│   └── test_foo.py
```

**Important**: Import test modules from `tests/__init__.py` (modules not imported are not run)

```python
# tests/__init__.py
from . import test_foo, test_bar
```

### Test Methods

Test methods must start with `test_`:

```python
class TestModelA(TransactionCase):
    def test_some_action(self):
        record = self.env['model.a'].create({'field': 'value'})
        record.some_action()
        self.assertEqual(record.field, expected_value)
```

---

## Test Classes

### TransactionCase

Most common test class. All test methods run in a single transaction; each method runs in a
savepoint rolled back at the end. Put shared fixtures in `setUpClass` (done once per class).

```python
from odoo.tests import TransactionCase

class TestMyModel(TransactionCase):
    @classmethod
    def setUpClass(cls):
        super().setUpClass()
        cls.partner = cls.env['res.partner'].create({'name': 'Test'})

    def test_create_record(self):
        record = self.env['my.model'].create({'name': 'Test', 'partner_id': self.partner.id})
        self.assertTrue(record.id)
```

### SingleTransactionCase — removed in 20

```python
# ❌ Odoo 20: ImportError, the class no longer exists in odoo/tests/common.py
from odoo.tests import SingleTransactionCase

# ✅ Use TransactionCase; build data shared by several tests in setUpClass
from odoo.tests import TransactionCase
```

### HttpCase

For HTTP/browser tests (`url_open`, `browser_js`, `start_tour`). Runs in registry test mode
(`cls.registry_test_mode()`), headless Chrome for browser tests. Logs a warning
`HttpCase test should be in post_install only` if a browser test runs before the registry is
fully loaded.

```python
from odoo.tests import HttpCase

class TestWebsite(HttpCase):
    def test_homepage(self):
        response = self.url_open('/hello')
        self.assertEqual(response.status_code, 200)
```

Other `HttpCase` helpers: `authenticate(login, password)`, `logout()`, `make_jsonrpc_request(route, params)`,
`update_session(**items)` / `update_session_context(**items)` (new in 20), `csrf_token()`.

---

## Test Decorators

### tagged

Class decorator (no effect on methods) adding or removing tags.

```python
from odoo.tests import TransactionCase, tagged

@tagged('-standard', 'nice')
class NiceTest(TransactionCase):
    pass
```

### Default Tags — changed in 20

`BaseCase` subclasses are now tagged **`standard` + `post_install`** by default (19 and earlier:
`standard` + `at_install`). A class must be exactly one of `at_install` / `post_install`;
`tagged()` logs `A tests should be either at_install or post_install` otherwise.

```python
# ❌ Odoo 19 habit: now redundant (post_install is the default)
@tagged('-at_install', 'post_install')
class WebsiteVisitorTests(HttpCase): ...

# ✅ Odoo 20: run right after module install only when really needed;
# remove post_install explicitly and say why in a comment
@tagged('at_install', '-post_install')  # needs to run before dependent modules are installed
class TestInstallBehavior(TransactionCase): ...
```

`at_install` should be used seldom: it prevents runbot from parallelizing test runs.

### users / warmup / no_retry

```python
from odoo.tests import TransactionCase, users, warmup, new_test_user

class TestPerf(TransactionCase):
    @classmethod
    def setUpClass(cls):
        super().setUpClass()
        new_test_user(cls.env, login='salesman', groups='sales_team.group_sale_salesman')

    @users('admin', 'salesman')   # runs the method once per login, in a subTest
    @warmup                       # runs once to warm caches, then for real
    def test_read(self):
        with self.assertQueryCount(admin=3, salesman=4):
            self.env['res.partner'].search([]).mapped('name')
```

`@no_retry` disables automatic retry on a method or class.

---

## Test Selection

`--test-tags` implies `--test-enable` (and `--stop-after-init`). `--test-enable` alone means
`--test-tags +standard`.

### By Tag

```bash
# Run only nice tests
odoo-bin --test-tags nice

# Run nice and standard tests
odoo-bin --test-tags nice,standard

# Run standard except slow
odoo-bin --test-tags 'standard,-slow'
```

### By Module

```bash
# Run only sale module tests
odoo-bin --test-tags /sale

# Run sale module but not slow tests
odoo-bin --test-tags '/sale,-slow'

# Run stock or slow tests
odoo-bin --test-tags '-standard, slow, /stock'
```

### By Specific Test

```bash
# Run specific test function
odoo-bin --test-tags .test_supplier_invoice

# Equivalent (full path)
odoo-bin --test-tags /account:TestAccount.test_supplier_invoice

# Run one Python test file
odoo-bin --test-file addons/my_module/tests/test_foo.py
```

### Tag Format

```
[-][tag][/module][:class][.method][[params]]
```

| Prefix | Meaning |
|--------|---------|
| `-` | Deselect/remove tag |
| `+` | Select/add tag (implicit, optional) |
| `/module` | Specific module |
| `:class` | Specific class |
| `.method` | Specific method |
| `[params]` | Parameters for tests supporting them, e.g. `/web.test_js[mail]` |

---

## Test Utilities

### browse_ref / ref

```python
record = self.browse_ref('base.user_admin')   # record
user_id = self.ref('base.user_admin')         # database id
```

### new_test_user

```python
from odoo.tests import new_test_user

user = new_test_user(self.env, login='portal_u', groups='base.group_portal')
# groups: comma-separated xmlids; password defaults to login padded to 8 chars;
# email generated if login is not an email
```

### Form

Server-side form emulation (onchanges, defaults, x2many). Import from `odoo.tests`
(`odoo.tests.common.Form` was deprecated since 18.0 and is gone in 20).

```python
from odoo.tests import Form

with Form(self.env['sale.order']) as form:
    form.partner_id = self.partner
    with form.order_line.new() as line:
        line.product_id = self.product
        line.product_uom_qty = 10
order = form.record   # saved at the end of the with block

# or explicitly
form = Form(self.env['sale.order'])
form.partner_id = self.partner
order = form.save()
```

### Patching and mocking

```python
self.patch(obj, 'attr', value)          # unittest.mock.patch.object + cleanup
self.classPatch(obj, 'attr', value)     # class-level variant
mock = self.startPatcher(patch('odoo.addons.x.models.y.call_api'))
self.enterContext(cm)                   # enter a context manager until test end
```

`BaseCase` blocks every un-mocked external HTTP request in `standard` tests
(`BlockedRequest: External requests verboten`); localhost is allowed. New in 20,
`MockHTTPClient` mocks `requests` calls:

```python
from odoo.tests.common import MockHTTPClient

with MockHTTPClient(url='https://api.example.com/v1/items',
                    return_json={'items': []}) as api_mock:
    self.env['my.connector']._sync()
    api_mock.assert_called_once()
```

Arguments: `url`, `method`, `matcher(request)`, `return_status`, `return_json`, `return_body`,
`return_headers`, `side_effect`; assertions `assert_called(n_times)`, `assert_called_once()`,
`assert_not_called()`; recorded requests in `.calls`.

### Other helpers

| Helper | Purpose |
|--------|---------|
| `self.assertRecordValues(records, [{...}, ...])` | Compare field values of a recordset |
| `RecordCapturer(model, domain)` | `with` block capturing records created inside (`.records`) |
| `self.file_read(path)` | Read a module file as `BinaryBytes` (new in 20; Binary fields hold `BinaryValue`) |
| `TransactionCase.mock_datetime_and_now(dt)` | Freeze time and `env.cr.now()` (classmethod context manager) |
| `self.registry_test_mode()` | Registry test mode context manager (`enter_registry_test_mode()` is a deprecated alias; 19's `registry_enter_test_mode` is gone) |
| `self.debug_mode()` | Enable debug-mode effects (`base.group_no_one`) |
| `can_import('module')` | Use with `unittest.skipUnless` for optional dependencies |

---

## JS Tests

Odoo uses **Hoot** for JS unit testing. Test files are named `*.test.js` under `static/tests/`
and must be added to the `web.assets_unit_tests` bundle. They run at `/web/tests` (Python
entry points: `addons/web/tests/test_js.py`).

```
your_module/
└── static/
    └── tests/
        └── my_test.test.js
```

```python
'assets': {
    'web.assets_unit_tests': [
        'your_module/static/tests/**/*.test.js',
    ],
},
```

```javascript
import { describe, expect, test } from "@odoo/hoot";
import { defineModels, fields, models, mountView, onRpc } from "@web/../tests/web_test_helpers";

class Partner extends models.Model {
    _name = "res.partner";
    name = fields.Char();
}

defineModels([Partner]);

describe("My Module", () => {
    test("form renders name", async () => {
        onRpc(({ method }) => expect.step(method));
        await mountView({ type: "form", resModel: "res.partner", arch: `<form><field name="name"/></form>` });
        expect(".o_field_widget[name=name]").toHaveCount(1);
    });
});
```

`test.tags("desktop")` / `test.tags("mobile")` restrict a test to a preset.

---

## Integration Tests (Tours)

Tours simulate real user scenarios in the browser.

### Tour Structure

```
your_module/
├── static/
│   └── tests/
│       └── tours/
│           └── my_tour.js
├── tests/
│   ├── __init__.py
│   └── test_my_tour.py
└── __manifest__.py
```

### Register Tour (JavaScript)

```javascript
// ❌ Legacy API (still printed in the official doc, not used in 20 code)
import tour from 'web_tour.tour';
tour.register('my_tour', { url: '/web' }, [ ... ]);

// ✅ Odoo 20: registry category "web_tour.tours", steps is a function
import { registry } from "@web/core/registry";
import { stepUtils } from "@web_tour/tour_utils";

registry.category("web_tour.tours").add("my_tour", {
    steps: () => [
        stepUtils.showAppsMenuItem(),
        {
            trigger: '.o_app[data-menu-xmlid="my_module.menu_root"]',
            run: "click",
        },
        {
            trigger: 'input[name="name"]',
            run: "edit New Record",
        },
        {
            content: "Record is listed",
            trigger: '.o_data_row:contains("New Record")',
        },
    ],
});
```

### Add to Manifest

```python
'assets': {
    'web.assets_tests': [
        'your_module/static/tests/tours/my_tour.js',
    ],
},
```

### Start Tour (Python)

```python
from odoo.tests import HttpCase

class TestMyTour(HttpCase):   # post_install by default
    def test_my_tour(self):
        self.start_tour("/odoo", "my_tour", login="admin")
```

`start_tour(url_path, tour_name, step_delay=None, **kwargs)` forwards kwargs to
`browser_js(url_path, code, ready='', login=None, timeout=60, cookies=None, error_checker=None,
watch=False, success_signal='test successful', debug=False, cpu_throttling=None)`.

### Tour Step Options

| Option | Description |
|--------|-------------|
| `trigger` | Selector/element to run action on (required) |
| `run` | Action to perform (helper name or async function) |
| `isActive` | Array of conditions (`desktop`/`mobile`, `community`/`enterprise`, `auto`/`manual`, or selectors) |
| `content` | Tooltip content, also logged (recommended) |
| `tooltipPosition` | `top`, `right`, `bottom`, or `left` |
| `timeout` | Wait time in ms (default: 10000) |

### Run Actions

| Action | Description |
|--------|-------------|
| `click` | Clicks the element |
| `dblclick` | Double-clicks the element |
| `drag_and_drop {target}` | Drags to target |
| `edit {content}` | Clears and fills |
| `editor {content}` | WYSIWYG editor fill |
| `fill {content}` | Fills the element |
| `hover` | Hovers over element |
| `press {content}` | Keyboard input |
| `range {content}` | Range slider value |
| `select {value}` | Select by value |
| `selectByIndex {index}` | Select by index |
| `selectByLabel {label}` | Select by label |
| `check` | Checks checkbox |
| `uncheck` | Unchecks checkbox |
| `clear` | Clears input |

The last step(s) must leave the client in a stable state (no pending edition or request).

### Run Tour from Browser

```javascript
odoo.startTour("tour_name");
```

Or use `?debug=tests` URL parameter.

### Debug Tours

```python
# Watch mode (opens Chrome window)
self.start_tour("/odoo", "my_tour", watch=True)

# Debug mode (opens devtools, breakpoint at tour start)
self.start_tour("/odoo", "my_tour", debug=True)
```

### Onboarding Tours

Same JS registration, plus a `web_tour.tour` record (`name` must match the registry key;
`url` defaults to `/odoo`, `sequence` to 1000):

```xml
<record id="my_tour" model="web_tour.tour">
    <field name="name">my_tour</field>
    <field name="sequence">10</field>
    <field name="rainbow_man_message">Great job!</field>
</record>
```

```python
'data': [
    'data/my_tour.xml',
],
'assets': {
    'web.assets_backend': [
        'your_module/static/src/js/tours/my_tour.js',
    ],
},
```

---

## Performance Testing

### Query Count Testing

```python
with self.assertQueryCount(11):
    do_something()

# Per-user limits (keys are logins, combine with @users)
with self.assertQueryCount(admin=3, salesman=5):
    do_something()
```

A lower count than expected only logs at INFO; a higher count fails in a subTest.

### Exact Queries

```python
with self.assertQueries(["""
    SELECT ... FROM "res_partner" WHERE ...
"""]):
    do_something()
```

Matching ignores case/whitespace; `...` is a wildcard. (19's `assertQueriesContain` is removed.)

---

## Synthetic Data (populate)

New in 20: the `populate` addon (`addons/populate`) generates data at scale from declarative
**blueprints** (`populate.blueprint` records) for performance testing. It provides the
`odoo-bin populate` command (`addons/populate/cli/populate.py`).

```bash
odoo-bin -d mydb -i populate          # module must be installed
pip install -r odoo/addons/populate/requirements.txt   # optional: Faker for fake.* generators
odoo-bin populate -d mydb -b sale.fake_sale_demo --scale 10 -j auto --seed 42
odoo-bin populate -d mydb --resume    # resume last interrupted session (or --resume <id>)
odoo-bin populate -d mydb -b my_module.my_blueprint --profile   # ir.profile per job
```

Blueprints live in a module's `populate/` folder (XML files, loaded when the `populate` module
is installed/upgraded: run `-u populate` after installing a module that ships one; an
`__init__.py` there is imported, to register custom generators):

```xml
<!-- my_module/populate/my_populate.xml -->
<odoo>
    <record id="my_blueprint" model="populate.blueprint">
        <field name="name">My Module - Populate</field>
        <field name="definition_xml" type="xml">
            <create model="res.partner" count="500" id="my_partners">
                <field name="name" generator="fake.company"/>
            </create>
            <create model="my.model" count="2000" id="records">
                <field name="partner_id" ref="my_partners"/>
                <field name="date" start="now -2y" end="now"/>
            </create>
        </field>
    </record>
</odoo>
```

Blocks: `<create>` (`count` required), `<write>` / `<function>` (target with `ref` or `domain`);
block attributes `scale`, `parallel`, `batched`, `context`. Full reference:
`doc/developer/reference/backend/populate.rst`. To bulk-duplicate *existing* records instead,
use `odoo-bin duplicate --models ... --factors ...` (`odoo/cli/duplicate.py`).

---

## Best Practices

### Regression and error-path coverage

Every reproducible bug fix should include a test that fails before the fix and passes afterward. New behavior should cover its success path and important error paths.

### Lowest practical permissions

Use `new_test_user()` and `@users` to exercise access-sensitive flows as the least privileged user that should be allowed to perform them. This avoids false positives from tests that run as an administrator.

### Deterministic, isolated fixtures

Create the data needed by the test instead of relying on demo records. Database changes are not rolled back between `subTest` cases, so use unique fixtures or explicitly reset state for each case.

Records stored on a class or test instance retain their original `.env` (including user, context, cursor, and sudo state). After changing the test environment, use `record.with_env(self.env)` before asserting access behavior.

Use fixed dates instead of `datetime.now()` or `datetime.today()` in assertions. Odoo 20 provides the helper through `odoo.tests.common` (it can decorate a test class, a method, or be used as a context manager):

```python
from datetime import datetime
from odoo.tests import TransactionCase
from odoo.tests.common import freeze_time

class TestSomething(TransactionCase):
    @freeze_time("2024-01-01 10:10:10")
    def test_creation_time(self):
        partner = self.env["res.partner"].create({"name": "Foo"})
        self.assertEqual(partner.create_date, datetime(2024, 1, 1, 10, 10, 10))
```

### Mock external services by default

Use `MockHTTPClient`, `self.patch()`, `self.classPatch()`, or `self.startPatcher()` to isolate network and other external services. If a live integration test is necessary, keep it out of the standard test selection with an explicit test tag such as `external`.

### Treat coverage drops as a test signal

When coverage decreases after a change, inspect the uncovered paths and add tests where the behavior is meaningful rather than lowering the expectation.

---

## Running Tests

```bash
# Enable tests (= --test-tags +standard), implies --stop-after-init
odoo-bin -d test_db --test-enable

# With specific tags
odoo-bin -d test_db --test-tags post_install

# Install and test a module
odoo-bin -d test_db -i my_module --test-enable

# Update and test
odoo-bin -d test_db -u my_module --test-tags /my_module
```

Tests only run for installed modules, and need `-d` (a warning `Empty ... tests won't run`
is logged otherwise). New databases are created **without** demo data unless `--with-demo` is
passed (`--without-demo` is the default), so a fresh `-d test_db -i my_module --test-enable`
run has no demo records and no `demo` / `portal` users. Create fixtures in `setUpClass`
(e.g. `new_test_user()`) instead of referencing demo XML IDs; `loaded_demo_data(env)` tells
whether demo data is present.

---

## Screenshot and Screencast

When `HttpCase.browser_js` tests fail, a screenshot is saved to
`{--screenshots}/{db_name}/screenshots/` (default `--screenshots` is `/tmp/odoo_tests`).

```bash
odoo-bin --screenshots /tmp/screenshots
odoo-bin --screencasts /tmp/screencasts   # DIR/{db_name}/screencasts
```

---

## Special Tags Reference

| Tag | Description |
|-----|-------------|
| `standard` | Default tag for BaseCase subclasses; default `--test-tags` |
| `post_install` | Run after all modules installed (**default in 20**) |
| `at_install` | Run right after the module installation (pair with `-post_install`) |
| `-standard` | Remove from default selection |

---

## References

- Source: `odoo/tests/common.py`, `odoo/tests/form.py`, `odoo/tools/config.py` (test options)
- Odoo 20 documentation `/doc/developer/reference/backend/testing.rst`, `populate.rst`
- Hoot helpers: `addons/web/static/tests/web_test_helpers.js`
