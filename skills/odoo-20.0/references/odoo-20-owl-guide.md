# Odoo 20 OWL Guide

Guide for building OWL components in Odoo 20. Odoo 20 ships **OWL 3**
(`addons/web/static/lib/owl/owl.js`, version `3.0.0-alpha.50`) plus a temporary
OWL 2 compatibility layer (`addons/web/static/src/owl2/`). Write new code in native OWL 3.

## Table of Contents

- [OWL 3 at a Glance](#owl-3-at-a-glance)
- [Component Structure](#component-structure)
- [Props](#props)
- [Reactivity: Signals, Proxy, Computed](#reactivity-signals-proxy-computed)
- [Hooks](#hooks)
- [Plugins and Services](#plugins-and-services)
- [QWeb Templates](#qweb-templates)
- [Icons](#icons)
- [Registries and Patching](#registries-and-patching)
- [Translations](#translations)

---

## OWL 3 at a Glance

Each line is a migration step of `odoo/upgrade_code/owl3-migration.py` or a rule enforced by
`owl.js` / `web/static/src/owl2/owl3_compatibility_layer.js`.

| OWL 2 (Odoo ≤ 19) ❌                                   | OWL 3 (Odoo 20) ✅                                               |
| ------------------------------------------------------ | ---------------------------------------------------------------- |
| `static props = {...}` / `static defaultProps`         | `props = useProps({...})` class field (compat layer **throws**)  |
| `this.state = useState({...})`                         | `signal(value)` or `proxy({...})` (`useState` no longer exported) |
| `reactive(obj)`                                        | `proxy(obj)`                                                     |
| `this.ref = useRef("name")` + `t-ref="name"`           | `ref = signal.ref()` + `t-ref="this.ref"`, read with `this.ref()` |
| `useEffect(fn, () => [deps])`                          | `useEffect(fn)` auto-tracks signals; OWL-2 form = `useLayoutEffect` from `@web/owl2/utils` |
| `useExternalListener(target, ev, fn)`                  | `useListener(target, ev, fn)`                                    |
| `useService("orm")`                                    | `usePlugin(ORM)` (`useService` still works as an alias)          |
| `<t t-esc="x"/>`                                       | `<t t-out="this.x"/>` (`t-esc` logs a deprecation warning)       |
| `<t t-slot="default"/>`                                | `<t t-call-slot="default"/>`                                     |
| `t-model="state.x"`                                    | `t-model="this.sig"` (signal) or `t-model.proxy="this.state.x"`  |
| `t-portal="..."`                                       | `t-custom-portal` (compat layer directive)                       |
| `<span t-out="props.title"/>` (implicit component ctx) | `<span t-out="this.props.title"/>` — component members need `this.` |
| `<t t-call="tpl"><t t-set="a" t-value="1"/></t>`       | `<t t-call="tpl" a="1"/>` (parametric t-call, 19.1)              |
| `status(this) === "destroyed"`                         | `useScope().status` vs `STATUS` (`status()` is a deprecated compat shim) |

> Source checkout tool: `odoo-bin upgrade_code --script owl3-migration` applies these rewrites
> (best effort; whitelists in the script list templates needing manual work).

---

## Component Structure

### Basic Component

```javascript
import { Component, signal, t, useProps } from "@odoo/owl";

export class MyComponent extends Component {
    static template = "my_module.MyComponent";
    static components = {};

    props = useProps({
        title: t.string().optional(),
        value: t.number().optional(0),
    });

    count = signal(0);

    setup() {
        // hooks (onWillStart, useEffect, ...) go here or in class fields
    }

    increment() {
        this.count.set(this.count() + 1);
    }
}
```

```xml
<templates xml:space="preserve">
    <t t-name="my_module.MyComponent">
        <div class="o_my_component">
            <h1 t-out="this.props.title"/>
            <button class="btn btn-primary" t-on-click="this.increment">
                <t t-out="this.count()"/>
            </button>
        </div>
    </t>
</templates>
```

- `static template` / `static components` are unchanged.
- The `owl="1"` attribute on `t-name` is unnecessary.
- Templates are evaluated with the component as `this`: write `this.props.x`, `this.state.y`,
  `this.method`. Bare names only resolve template locals (`t-set`, `t-foreach`, slot scope,
  t-call attributes).

---

## Props

```javascript
import { Component, t, useProps } from "@odoo/owl";

// ❌ OWL 2 — throws in Odoo 20: 'defines a static "props" or "defaultProps", which Owl 3 ignores'
static props = { size: { type: Number, optional: true } };
static defaultProps = { size: 1 };

// ✅ OWL 3
props = useProps({
    size: t.number().optional(1),          // optional(default)
    record: t.object(),
    onChange: t.function().optional(() => () => {}),
    slots: t.object().optional(),
});
```

- No `useProps` → no `this.props`. `useProps()` without schema accepts every prop
  (used by client action wrappers); `useProps(["a", "b"])` declares names only.
- Validators (`t` / `types` from `@odoo/owl`): `any`, `string`, `number`, `boolean`, `object`,
  `array`, `function`, `instanceOf`, `constructor`, `literal`, `selection`, `or`, `and`,
  `record`, `tuple`, `signal`, `promise`, `customValidator`, `component`.
- Props are validated when the app runs in dev mode (`odoo.debug` or test mode). Each prop is backed by a signal,
  so reading `this.props.x` in a `computed`/`useEffect` tracks it.
- Shared prop sets: `standardFieldProps` (`@web/views/fields/standard_field_props`),
  `standardWidgetProps` (`@web/views/widgets/standard_widget_props`) — spread them into `useProps`.
- `.bind` suffix still works: `<CheckBox onChange.bind="this.onChange"/>`.
- `t-ref` on a component is an error; expose a callback prop instead.

---

## Reactivity: Signals, Proxy, Computed

```javascript
import { Component, computed, proxy, signal } from "@odoo/owl";

export class Counter extends Component {
    static template = "my_module.Counter";

    count = signal(0);                              // read: this.count(), write: this.count.set(1)
    items = signal.Array([]);                       // reactive array content (also signal.Object/Map/Set)
    state = proxy({ query: "", loading: false });   // deep reactive object (replaces useState)
    double = computed(() => this.count() * 2);      // derived, read: this.double()

    increment() {
        this.count.set(this.count() + 1);
    }
}
```

- A component re-renders when a signal/proxy key it read during render changes — nothing
  to declare.
- `signal.trigger(sig)` forces observers to update after in-place mutation of a plain `signal`.
- `untrack(fn)` reads without subscribing; `markRaw(obj)` / `toRaw(obj)` opt out of proxying.
- `asyncComputed(fetcher, { initial })` exists for async derived values.

```javascript
// ❌ OWL 2
this.state = useState({ count: 0 });
// ✅ OWL 3 (what upgrade_code rewrites to)
this.state = proxy({ count: 0 });
```

---

## Hooks

### Lifecycle

| Hook                    | When                                         |
| ----------------------- | -------------------------------------------- |
| `setup()`               | Component creation (hooks registered here)   |
| `onWillStart(fn)`       | Before first render; render awaits its promise |
| `onMounted(fn)`         | After first render in DOM                    |
| `onWillUpdateProps(fn)` | Before new props are applied                 |
| `onWillPatch(fn)`       | Before a re-render is patched                |
| `onPatched(fn)`         | After re-render                              |
| `onWillUnmount(fn)`     | Before removal from DOM                      |
| `onWillDestroy(fn)`     | Before destruction (also in plugins)         |
| `onError(fn)`           | Error in a child                             |

`onWillRender` and the OWL-2 style `useLayoutEffect`, `useEnv`, `useSubEnv` are provided by
the OWL 2 compatibility layer (import them from `@web/owl2/utils`, as the migration script
does); `onRendered`, `useRef`, `useChildSubEnv`, `useExternalListener` and `useState` are not
available at all in Odoo 20.

### Effects, refs, listeners

```javascript
import { Component, onWillStart, signal, useEffect, useListener } from "@odoo/owl";
import { loadJS } from "@web/core/assets";

export class PieChart extends Component {
    static template = "my_module.PieChart";
    canvasRef = signal.ref();

    setup() {
        onWillStart(() => loadJS("/web/static/lib/Chart/Chart.js"));
        // re-runs when any signal/prop it reads changes; returned fn = cleanup
        useEffect(() => {
            const canvas = this.canvasRef();
            if (canvas) {
                const chart = new Chart(canvas, this.chartConfig());
                return () => chart.destroy();
            }
        });
        useListener(window, "resize", () => this.onResize());
    }
}
```

```xml
<div class="o_pie_chart"><canvas t-ref="this.canvasRef"/></div>
```

- `useOnChange(() => [deps], (...deps) => {...})` runs a callback when deps change.
- Odoo hooks in `@web/core/utils/hooks`: `useService`, `useBus`, `useAutofocus({ ref })`,
  `useOwnedDialogs`, `useSpellCheck`, `useBackButton`.

---

## Plugins and Services

In OWL 3 global services are **plugins**: classes extending `Plugin`, registered in the
`services` resource (`@web/core/services`) and started once by the web client.

```javascript
import { Component, Plugin, onWillDestroy, signal, usePlugin } from "@odoo/owl";
import { services } from "@web/core/services";
import { rpc } from "@web/core/network/rpc";
import { ORM } from "@web/core/orm_plugin";
import { NotificationPlugin } from "@web/core/notifications/notification_plugin";

export class ReportingPlugin extends Plugin {
    orm = usePlugin(ORM);
    stats = signal(null);

    setup() {
        this.load();
        const id = setInterval(() => this.load(), 600_000);
        onWillDestroy(() => clearInterval(id));
    }

    async load() {
        this.stats.set(await rpc("/my_module/statistics"));
    }
}
services.add(ReportingPlugin);

export class Dashboard extends Component {
    static template = "my_module.Dashboard";
    reporting = usePlugin(ReportingPlugin);        // same instance for every component
    notification = usePlugin(NotificationPlugin);
}
```

| Plugin               | Import from                                     | Legacy `useService` name |
| -------------------- | ----------------------------------------------- | ------------------------ |
| `ORM`                | `@web/core/orm_plugin`                          | `"orm"`                  |
| `ActionPlugin`       | `@web/webclient/actions/action_plugin`          | `"action"`               |
| `DialogPlugin`       | `@web/core/dialog/dialog_plugin`                | `"dialog"`               |
| `NotificationPlugin` | `@web/core/notifications/notification_plugin`   | `"notification"`         |
| `EffectPlugin`       | `@web/core/effects/effect_plugin`               | `"effect"`               |
| `HotkeyPlugin`       | `@web/core/hotkeys/hotkey_plugin`               | `"hotkey"`               |
| `UIPlugin`           | `@web/core/ui/ui_plugin`                        | `"ui"`                   |
| `PopoverPlugin`      | `@web/core/popover/popover_plugin`              | `"popover"`              |

- `useService(name)` (`@web/core/utils/hooks`) still works: converted plugins keep a
  `registry.category("services")` alias marked `@todo owl3 migration`, and unconverted services
  (`name`, `field`, `view`, `file_upload`, `mail.store`, ...) are still registry services.
- `rpc`, `user`, `router` are **modules, not services**:
  `import { rpc } from "@web/core/network/rpc"`, `import { user } from "@web/core/user"`,
  `import { router } from "@web/core/browser/router"`. ❌ `useService("rpc")`.

### ORM

```javascript
const ids = await this.orm.search("my.model", [["active", "=", true]], { limit: 10 });
const recs = await this.orm.read("my.model", ids, ["name", "value"]);
const rows = await this.orm.searchRead("my.model", [], ["name"]);
const [id] = await this.orm.create("my.model", [{ name: "Test" }]);   // list of dicts
await this.orm.write("my.model", [id], { name: "Updated" });
await this.orm.unlink("my.model", [id]);
await this.orm.call("my.model", "action_confirm", [[id]]);
await this.orm.silent.read("my.model", [id], ["name"]);              // no error dialog
```

### Action, Dialog, Notification

```javascript
await this.action.doAction("base.action_partner_form");
await this.action.doAction({
    type: "ir.actions.act_window",
    res_model: "my.model",
    views: [[false, "list"], [false, "form"]],
});

this.dialog.add(ConfirmationDialog, {        // @web/core/confirmation_dialog/confirmation_dialog
    body: _t("Are you sure?"),
    confirm: () => this.doIt(),
    cancel: () => {},
});

this.notification.add(_t("Saved"), { type: "success" });   // ❌ notification.notify({...})
```

---

## QWeb Templates

```xml
<!-- Output / events -->
<span t-out="this.props.title"/>
<button t-on-click="this.increment">+</button>
<button t-on-click="() => this.select(record.id)">Pick</button>

<!-- Loops and conditions: loop vars are locals (no this.) -->
<t t-foreach="this.state.records" t-as="record" t-key="record.id">
    <span t-out="record.name"/>
</t>
<div t-if="this.state.show">Shown</div>
<div t-else="">Hidden</div>

<!-- Inputs -->
<input t-model="this.searchTerm"/>                 <!-- searchTerm = signal("") -->
<input t-model.proxy.trim="this.state.name"/>      <!-- state = proxy({...}); also .lazy, .number -->

<!-- Slots -->
<t t-call-slot="default"/>
<t t-call-slot="footer" close="() => this.close()"/>
<Layout display="{controlPanel: {}}">
    <t t-set-slot="control-panel-buttons">...</t>
</Layout>

<!-- Dynamic component / props -->
<t t-component="item.Component" t-props="itemProps"/>

<!-- Sub-template with parameters as attributes -->
<t t-call="my_module.Header" fullscreen="this.isFullscreen"/>
```

- `t-model` without `.proxy` requires a signal (a function with `.set`), else
  "Invalid t-model expression".
- `t-custom-click="this.onOpen"` (Odoo directive, `web/static/src/env.js`) handles click +
  middle/ctrl-click.

---

## Icons

Since 19.5 Font Awesome is gone: icons are **Material Symbols** (plus `odoo_ui_icons` for
brand glyphs, `oi_*` names), rendered by the `oi` class and a `data-icon` attribute
(`web/static/src/webclient/icons.scss`). Migration script: `19.5-00-material-symbols-icons.py`.

```xml
<!-- ❌ -->  <i class="fa fa-user fa-fw"/>
<!-- ✅ -->  <i class="oi oi-fw" data-icon="person" role="img" aria-label="User"/>
<!-- filled variant -->  <i class="oi oi-filled" data-icon="star"/>
<!-- dynamic -->  <i class="oi" t-att-data-icon="this.open ? 'visibility' : 'visibility_off'"/>
```

- Modifiers: `oi-fw`, `oi-spin`, `oi-pulse`, `oi-lg`, `oi-sm`, `oi-2x`…`oi-10x`, `oi-rotate-90`,
  `oi-flip-horizontal`, `oi-filled`.
- JS icon strings hold the name only: `icon: "person"` (❌ `icon: "fa fa-user"`).
- SCSS/selectors: ❌ `.fa-star` → ✅ `[data-icon='star']`.

---

## Registries and Patching

```javascript
import { registry } from "@web/core/registry";

// Client action (XML: ir.actions.client with <field name="tag">my_module.dashboard</field>)
registry.category("actions").add("my_module.dashboard", Dashboard);

// Field widget: <field name="x" widget="my_field"/>
registry.category("fields").add("my_field", {
    component: MyField,                 // props = useProps(standardFieldProps)
    supportedTypes: ["char"],
});

// View widget: <widget name="my_widget"/>
registry.category("view_widgets").add("my_widget", { component: MyWidget });
```

OWL 3 also offers a validating `Registry` (`import { Registry, t } from "@odoo/owl"`,
`new Registry({ name, validation: t.object({...}) })`, `addById(item)`, `items()`) for
module-local extension points.

```javascript
import { patch } from "@web/core/utils/patch";
import { ListRenderer } from "@web/views/list/list_renderer";

patch(ListRenderer.prototype, {
    setup() {
        super.setup(...arguments);
        this.notification = useService("notification");
    },
    getPropertyFieldColumns(_, list) {
        const columns = super.getPropertyFieldColumns(...arguments);
        // ...
        return columns;
    },
});
```

---

## Translations

```javascript
import { _t } from "@web/core/l10n/translation";

_t("Hello World");
_t("Install %s", app.name);
_t("Invalid email address: %(address)s", { address });
```

Static text in templates is extracted and translated automatically; use the `.translate`
suffix for string props: `<Dialog title.translate="Confirm"/>`.

---

## Best Practices

- ✅ Class-field `useProps`, `signal`, `usePlugin`; keep `setup()` for lifecycle hooks.
- ✅ Hold shared/long-lived state in a plugin with signals, not in components.
- ✅ Return a cleanup from `useEffect`; use `onWillDestroy` for timers in plugins.
- ❌ New code importing from `@web/owl2/utils` or relying on `this.env` / `useEnv` — that
  compat layer is "a temporary bridge" (`owl3_compatibility_layer.js`).
- ❌ Direct DOM manipulation outside refs/effects.

---

## References

- `addons/web/static/lib/owl/owl.js` (OWL 3 exports)
- `addons/web/static/src/owl2/owl3_compatibility_layer.js`, `owl2/utils.js`
- `odoo/upgrade_code/owl3-migration.py`, `19.5-00-material-symbols-icons.py`
- `addons/web/static/src/core/services.js`, `core/orm_plugin.js`, `core/utils/hooks.js`
- Documentation: `developer/tutorials/client_framework_101` (Odoo 20)
