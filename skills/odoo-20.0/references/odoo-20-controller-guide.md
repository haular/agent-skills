# Odoo 20 Controller Guide

## Coding Conventions

Group related routes in one controller and preserve extensibility by redecorating
overrides and calling `super()` where the parent behavior remains relevant.
Name controller files after the served model or feature (for example,
`controllers/my_model.py`), never a catch-all `my_controller.py`.
Keep `request` in controllers and `ir.http` overrides; model code reads
`self.env` (`env.context`, `env.website`) instead.

Guide for creating HTTP controllers and routes in Odoo 20.

## Table of Contents

- [Imports (`odoo/http/` package)](#imports-odoohttp-package)
- [Controllers](#controllers)
- [Routes](#routes)
- [Authentication](#authentication)
- [Request](#request)
- [Response](#response)
- [JSON-RPC](#json-rpc)
- [External API (`/json/2`)](#external-api-json2)
- [Website Routes](#website-routes)
- [HTTP Server](#http-server)

---

## Imports (`odoo/http/` package)

Since 19.4, `odoo/http.py` is a package (`odoo/http/`). The package root only
exposes `request`, `request_var`, `Response`, `Controller` and `route`;
everything else must be imported from its submodule.

```python
# ✅ Odoo 20
from odoo import http
from odoo.http import Controller, Response, request, route
from odoo.http.stream import Stream, content_disposition, STATIC_CACHE_LONG
from odoo.http.requestlib import Request
from odoo.http.session import SessionExpiredException
from odoo.http.dispatcher import serialize_exception
from odoo.http.router import root, db_list, dispatch_rpc
from odoo.http.retrying import retrying

# ❌ Odoo 19 style: ImportError in 20
from odoo.http import content_disposition, Stream, SessionExpiredException
```

| Symbol | Odoo 20 module |
| ------ | -------------- |
| `Controller`, `route`, `ROUTING_KEYS` | `odoo.http.routing_map` (root re-exports `Controller`, `route`) |
| `Request`, `fragment_to_query_string` | `odoo.http.requestlib` |
| `Response` | `odoo.http` / `odoo.http.response` |
| `Stream`, `content_disposition`, `STATIC_CACHE*` | `odoo.http.stream` |
| `Session`, `session_store`, `SessionExpiredException` | `odoo.http.session` |
| `HttpDispatcher`, `JsonRPCDispatcher`, `Json2Dispatcher`, `serialize_exception`, `conceal_debug_traceback` | `odoo.http.dispatcher` |
| `root`, `db_list`, `db_filter`, `dispatch_rpc` | `odoo.http.router` |
| `retrying` | `odoo.http.retrying` (was `odoo.service.model.retrying`) |

---

## Controllers

Controllers provide extensibility similar to models, but with a separate mechanism (since database may not be available).

Controllers are created by inheriting from `odoo.http.Controller`:

```python
from odoo import http

class MyController(http.Controller):
    @http.route('/some_url', auth='public')
    def handler(self):
        return stuff()
```

`Controller.env` returns `request.env` (or `None` outside a request), so
`self.env['res.partner']` works inside a handler.

### Controller Inheritance

To override a controller, inherit from its class and override methods:

```python
class Extension(MyController):
    @http.route()
    def handler(self):
        do_before()
        return super().handler()
```

**Important**:

- Always re-apply `@http.route()` decorator to keep route visible
- Without decorator, a warning is logged and the method is decorated automatically
- Decorator arguments override previous ones
- An override cannot change the route `type` (a warning is logged and the original type is kept)

### Change Authentication

```python
class Restrict(MyController):
    @http.route(auth='user')
    def handler(self):
        return super().handler()
```

This changes `/some_url` from public to user (requires login).

---

## Routes

### Route Decorator

`@http.route()` defines routing for controller methods.

```python
@http.route('/hello', auth='public', website=True)
def hello(self):
    return "Hello World!"
```

### Route Parameters

| Parameter | Description |
| --------- | ----------- |
| `route` | Route path(s) (string or list) |
| `type` | `'http'` (default), `'jsonrpc'` or `'json2'` |
| `auth` | `'user'` (default), `'public'`, `'none'`, `'bearer'`, or a custom `ir.http._auth_method_<name>` |
| `bearer_scope` | API-key scope; **mandatory** with `auth='bearer'`, must not be set otherwise |
| `methods` | Allowed HTTP methods; all when omitted |
| `csrf` | CSRF check (default: enabled for `http`, disabled for `jsonrpc`) |
| `cors` | `Access-Control-Allow-Origin` value |
| `readonly` | Bool or `callable(controller, rule, args)`: open the cursor on the read-only replica (default `True` only for `auth='none'`) |
| `save_session` | Set the session cookie / save the session (default `False` for `auth='bearer'`, `True` otherwise) |
| `captcha` | Captcha action name, checked on unsafe methods |
| `handle_params_access_error` | `callable(exc)` returning a response when a record in the URL is not accessible |
| `max_content_length` | Per-route request body limit (int or `callable(controller)`) |
| `website`, `multilang`, `sitemap` | Read by `http_routing` / `website` (frontend routes) |

### Route Types

| `type` | Dispatcher | Params from | Returns |
| ------ | ---------- | ----------- | ------- |
| `http` | `HttpDispatcher` | query string + form + files | `str`/`bytes`/`None`/`Response` (anything else raises `TypeError`) |
| `jsonrpc` | `JsonRPCDispatcher` | JSON-RPC 2 body `params` | any JSON-serializable value, wrapped in `{"jsonrpc": "2.0", "result": ...}` |
| `json2` | `Json2Dispatcher` | raw JSON body object | any JSON-serializable value, sent as is; errors as HTTP 4xx/5xx JSON |

```python
# ❌ deprecated alias, warns:
# "Since 19.0, @route(type='json') is a deprecated alias to @route(type='jsonrpc')"
@http.route('/api/data', type='json', auth='user')

# ✅
@http.route('/api/data', type='jsonrpc', auth='user')
```

`odoo-bin upgrade_code` (script `18.1-02-route-jsonrpc`) rewrites `type='json'` in `controllers/`.

### Multiple Routes

```python
@http.route(['/hello', '/bonjour'], auth='public')
def hello_bonjour(self):
    return "Hello or Bonjour!"
```

### HTTP Methods

```python
@http.route('/api/data', methods=['GET'], auth='user', type='jsonrpc')
def get_data(self):
    return {'data': 'value'}

@http.route('/api/data', methods=['POST'], auth='user', type='jsonrpc')
def post_data(self, **kwargs):
    return {'result': 'created'}
```

---

## Authentication

### Authentication Types

| Type | Description |
| ---- | ----------- |
| `user` | Requires an authenticated session (default) |
| `public` | Logged user if any, else the public user (website: the website's public user) |
| `bearer` | `Authorization: Bearer <api key>` checked against `bearer_scope`; falls back to the session (as `user`) when the header is absent; stateless |
| `none` | No user (`request.env.uid` is `None`); works without database |

Website routes are not a distinct `auth` value: use `auth='public'` (or `'user'`) together with `website=True`.

### Examples

```python
# Public route
@http.route('/page', auth='public')
def public_page(self):
    return "Everyone can see this"

# User-only route
@http.route('/my-account', auth='user')
def user_page(self):
    return "Only logged users can see this"

# API-key route: bearer_scope is mandatory (AssertionError otherwise)
@http.route('/my_module/greeting', type='http', auth='bearer', bearer_scope='rpc', readonly=True)
def greeting(self):
    return f"Hello {self.env.user.name}"
```

### Current User

```python
@http.route('/profile', auth='user')
def profile(self):
    return f"Hello, {self.env.user.name}"
```

---

## Request

The request object is bound to `odoo.http.request` (a `LocalProxy` over the
`request_var` context variable) for the duration of each HTTP request.

### Request Properties

| Property | Description |
| -------- | ----------- |
| `httprequest` | Wrapped Werkzeug request |
| `env` | Odoo environment for current request |
| `registry` | Model registry |
| `db` | Current database name |
| `session` | Session |
| `params` | Merged route/query/body parameters |
| `geoip` | GeoIP lookup of the remote address |
| `future_response` | Headers/cookies to inject in the final response |
| `lang` | Frontend language data (`res.lang._get_data(...)`), set by `http_routing` |

Deprecated since 19.0, still working with a `DeprecationWarning`:

```python
request.cr       # "Since 19.0, use request.env.cr directly"
request.uid      # "Since 19.0, use request.env.uid directly"
request.context  # "Since 19.0, use request.env.context directly"
```

Change user/context with `request.update_env(user=..., context=..., su=...)` or
`request.update_context(**overrides)`; assigning `request.uid`/`request.context` raises.

### Example

```python
@http.route('/info', auth='user')
def info(self):
    user = request.env.user
    company = request.env.company
    return f"{user.name} @ {company.name}"
```

### Session

```python
@http.route('/set-value', auth='public', methods=['POST'])
def set_value(self, key, value):
    request.session[key] = value
    return "OK"

@http.route('/get-value', auth='public')
def get_value(self, key):
    return request.session.get(key, 'not set')
```

### `request` in model code

Since 19.4, the current website is an environment property: `env.website`
(`website` browsed from `context['website_id']`, possibly empty). The model
`website` lives in `base`.

```python
# ❌ Odoo 19
website = self.env['website'].get_current_website()   # removed in 20
website = request.website                              # no longer set

# ✅ Odoo 20
website = self.env.website
```

---

## Response

### HTTP Response

`type='http'` returns `str`/`bytes`/`Response`; `type='jsonrpc'`/`'json2'` return JSON-serializable data:

```python
# HTML response
@http.route('/html', auth='public', type='http')
def html_response(self):
    return "<h1>Hello</h1>"

# JSON from an http route
@http.route('/status.json', auth='public', type='http')
def status(self):
    return request.make_json_response({'status': 'ok'})

# JSON-RPC response
@http.route('/json', auth='public', type='jsonrpc')
def json_response(self):
    return {'key': 'value'}
```

### Redirect

```python
# ❌ odoo.http has no `redirect` function
from odoo.http import redirect

# ✅
@http.route('/old-url', auth='public')
def old_url(self):
    return request.redirect('/new-url')          # 303, local by default
```

`request.redirect_query(location, query)` appends a query string;
`request.not_found()` returns a `NotFound` exception to raise.

### File Response

```python
from odoo.http.stream import content_disposition

@http.route('/download', auth='user')
def download_file(self):
    file_content = b'file data'
    return request.make_response(file_content, [
        ('Content-Type', 'application/pdf'),
        ('Content-Disposition', content_disposition('file.pdf')),
    ])
```

### Binary Fields and Attachments

Binary field values are `BinaryValue` objects (19.3+), not base64. Stream them
through `ir.binary`, which returns an `odoo.http.stream.Stream`:

```python
@http.route('/my/document/<int:doc_id>', auth='user')
def document(self, doc_id):
    attachment = self.env['ir.attachment'].browse(doc_id).exists()
    if not attachment:
        raise request.not_found()
    return (
        self.env['ir.binary']
        ._get_stream_from(attachment)          # field_name='raw' by default
        .get_response(as_attachment=True)
    )
```

---

## JSON-RPC

### JSON Controller

```python
@http.route('/api/search', auth='user', type='jsonrpc', methods=['POST'])
def json_search(self, model, domain, fields=None):
    records = request.env[model].search(domain)
    return {'result': records.read(fields)}
```

Request body: `{"jsonrpc": "2.0", "method": "call", "params": {...}, "id": null}`;
`params` must be an object (by-name parameters only).

### Call from JavaScript

```javascript
import { rpc } from "@web/core/network/rpc";

const result = await rpc("/api/search", {
    model: "res.partner",
    domain: [["is_company", "=", true]],
    fields: ["name", "email"],
});
```

---

## External API (`/json/2`)

- `POST /json/2/<model>/<method>` (addon `rpc`, `type='json2'`, `auth='bearer'`,
  `bearer_scope='rpc'`): headers `Authorization: bearer <api key>`,
  `Content-Type: application/json`, optional `X-Odoo-Database`; body
  `{"ids": [...], "context": {...}, <method kwargs>}`. Only public methods
  (not `_`-prefixed, not `@api.private`) are callable; `@api.readonly` methods
  run on the read-only cursor.
- `/xmlrpc`, `/xmlrpc/2`, `/jsonrpc` are deprecated (warning logged on each call:
  "deprecated in Odoo 19 and scheduled for removal in Odoo 22"). The `db` service
  is removed in Odoo 20 (`dispatch_rpc` accepts only `common` and `object`).
- Your own `@route(type='jsonrpc')` controllers are not affected by this deprecation.

---

## Website Routes

### Website Page

```python
class WebsiteController(http.Controller):
    @http.route('/my-page', auth='public', website=True)
    def my_page(self):
        return request.render('my_module.my_page_template', {
            'title': 'My Page',
        })
```

### Template

```xml
<template id="my_page_template" name="My Page">
    <t t-call="website.layout">
        <div id="wrap">
            <div class="oe_structure"/>
            <h1 t-out="title"/>
            <div class="oe_structure"/>
        </div>
    </t>
</template>
```

---

## HTTP Server

Since 19.4 Odoo serves HTTP itself with an `h11`-based server
(`odoo/http/server.py`, `HTTPSocket`) in threaded, prefork and gevent modes;
`werkzeug.serving` is no longer used. Odoo must run behind a reverse proxy
(`--proxy-mode` applies `ProxyFix` when `X-Forwarded-Host` is present).

| Option | Odoo 20 |
| ------ | ------- |
| `--http-interface` | default `127.0.0.1` (was `0.0.0.0` in 19.0) |
| `--gevent-workers` | new: number of gevent (websocket) workers in prefork mode, default `1` (requires `SO_REUSEPORT`) |
| `--gevent-port` | unchanged, default `8072` |

---

## Controllers Best Practices

### Always Return a Value

```python
# BAD: no return
@http.route('/bad', auth='public')
def bad(self):
    pass

# GOOD: return something
@http.route('/good', auth='public')
def good(self):
    return "Response"
```

### Use Proper Authentication

```python
# BAD: public for sensitive data
@http.route('/sensitive', auth='public')
def sensitive(self):
    return secret_data()

# GOOD: user authentication
@http.route('/sensitive', auth='user')
def sensitive(self):
    return secret_data()
```

### CSRF Protection

CSRF is enabled by default for `type='http'` routes on unsafe methods (not `GET`/`HEAD`/`OPTIONS`/`TRACE`) and is not checked for `type='jsonrpc'`. Disable with caution:

```python
@http.route('/webhook', auth='public', methods=['POST'], csrf=False)
def webhook(self):
    # External webhook, no CSRF token
    return "OK"
```

Forms post the token as `csrf_token` (`request.csrf_token()`).

---

## References

- Source: `odoo/http/` (`routing_map.py`, `requestlib.py`, `dispatcher.py`, `stream.py`, `router.py`, `server.py`), `odoo/addons/base/models/ir_http.py`, `addons/rpc/controllers/`
- Odoo 20 documentation `/developer/reference/backend/http.rst`, `/developer/reference/external_api.rst`
