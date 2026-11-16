# Odoo 20 Reports Guide

## Coding Conventions

Use focused report/template filenames and stable descriptive XML IDs.

Guide for creating QWeb reports in Odoo 20: PDF/HTML reports, templates, PDF engines, and paper formats.

## Table of Contents

- [Report Basics](#report-basics)
- [PDF Engines](#pdf-engines)
- [Report Declaration](#report-declaration)
- [Report Templates](#report-templates)
- [Custom Reports](#custom-reports)
- [Rendering and Attachments in Python](#rendering-and-attachments-in-python)
- [Paper Formats](#paper-formats)
- [Translatable Reports](#translatable-reports)
- [Barcodes](#barcodes)

---

## Report Basics

Reports are written in HTML/QWeb. PDF conversion is delegated to a pluggable PDF engine
module (wkhtmltopdf by default, Paper Muncher optional).

Reports consist of:

1. A **report action** (`ir.actions.report`)
2. A **QWeb template** for the report content

---

## PDF Engines

Since 19.4 the reporting engines are modular: `base` no longer contains wkhtmltopdf code.
`ir.actions.report` in `base` only defines the hooks; engine modules implement them.

| Module                       | Engine name     | `report_type` added       | Notes                                             |
| ---------------------------- | --------------- | ------------------------- | ------------------------------------------------- |
| `base_report_wkhtmltox`      | `wkhtmltopdf`   | `qweb-pdf-wkhtmltopdf`    | `auto_install`; sets `report.pdf_engine_default` |
| `base_report_paper_muncher`  | `paper-muncher` | `qweb-pdf-paper-muncher`  | Requires paper-muncher >= 0.6 on the system      |

Engine resolution (`_get_pdf_engine`):

- `report_type="qweb-pdf-<engine>"` forces that engine for one report.
- `report_type="qweb-pdf"` uses the `report.pdf_engine_default` system parameter
  (`wkhtmltopdf` once `base_report_wkhtmltox` is installed); if it is unset or `html`,
  the PDF request falls back to rendering HTML.

Engine hooks to override in an engine module (all on `ir.actions.report`):

| Method                                                       | Purpose                                     |
| ------------------------------------------------------------ | ------------------------------------------- |
| `get_pdf_engine_state(engine_name)`                          | `'ok'`, `'upgrade'`, `'workers'`, `'broken'`, `'install'` |
| `_run_pdf_engine(engine_name, html, report_ref=False, landscape=False, **kwargs)` | Returns `(pdf_bytes, html_ids)` |
| `_run_pdf_engine_without_processing(engine_name, bodies, report_ref=False, *, header=None, footer=None, landscape=False, specific_paperformat_args=None, **kwargs)` | Raw bodies to PDF bytes |
| `_run_image_engine(engine_name, bodies, width, height, image_format="jpg")` | HTML to images |

Unhandled engines raise `NotImplementedError("Unknown PDF engine: ...")`.

```python
# ❌ Odoo ≤ 19.0: wkhtmltopdf helpers lived in base
self.env['ir.actions.report']._run_wkhtmltopdf(bodies, report_ref=..., header=..., footer=...)

# ✅ Odoo 20: go through the engine hook
Report = self.env['ir.actions.report']
engine = Report._get_pdf_engine(report)
pdf = Report._run_pdf_engine_without_processing(engine, bodies, report_ref=report.report_name)
```

---

## Report Declaration

### No `<report>` Shortcut

Odoo 20 has no `<report>` tag (the XML loader only accepts `record`, `template`,
`menuitem`, `function`, `delete` and `asset`); a `<report .../>` element fails validation.
Always declare an `ir.actions.report` record.

### Using record Tag

```xml
<record id="my_report" model="ir.actions.report">
    <field name="name">My Report</field>
    <field name="model">my.model</field>
    <field name="report_type">qweb-pdf</field>
    <field name="report_name">my_module.my_template</field>
    <field name="print_report_name">'My Report - %s' % (object.name)</field>
    <field name="binding_model_id" ref="model_my_model"/>
    <field name="paperformat_id" ref="base.paperformat_euro"/>
</record>
```

### Report Attributes

| Attribute            | Description                                                         |
| -------------------- | ------------------------------------------------------------------- |
| `id`                 | External identifier of the record                                   |
| `name`               | Report name                                                         |
| `model`              | Model for the report                                                |
| `report_type`        | `qweb-pdf`, `qweb-html`, `qweb-text`; `qweb-pdf-<engine>` with an engine module |
| `report_name`        | External ID of QWeb template                                        |
| `print_report_name`  | Python expression for filename (`object`, `time`)                   |
| `paperformat_id`     | Paper format                                                        |
| `binding_model_id`   | Model to show in Print menu                                         |
| `group_ids`          | Groups allowed to use report                                        |
| `multi`              | If True, don't show on form view                                    |
| `domain`             | Show the action only on records matching this domain                |
| `attachment`         | Python expression for the attachment name (`object`, `time`)        |
| `attachment_use`     | Reload from the stored attachment on reprint                        |

❌ `report_file` no longer exists on `ir.actions.report` in 20: drop it from data files.

---

## Report Templates

### Template Variables

Report templates always have access to:

| Variable            | Description                                   |
| ------------------- | --------------------------------------------- |
| `docs`              | Records for the current report                |
| `doc_ids`           | List of ids for `docs`                        |
| `doc_model`         | Model name for `docs`                         |
| `time`              | Python `time` module                          |
| `user`              | User printing the report                      |
| `res_company`       | Current company (`env.company`)               |
| `web_base_url`      | `web.base.url` system parameter               |
| `context_timestamp` | Function to convert datetime to user timezone |
| `is_html_empty`     | Helper to test empty HTML fields              |

### Minimal Template

```xml
<template id="report_invoice">
    <t t-call="web.html_container">
        <t t-foreach="docs" t-as="o">
            <t t-call="web.external_layout">
                <div class="page">
                    <h2>Invoice</h2>
                    <p>Customer: <span t-field="o.partner_id.name"/></p>
                    <p>Total: <span t-field="o.amount_total"/></p>
                </div>
            </t>
        </t>
    </t>
</template>
```

### External Layout

Calling `web.external_layout` adds the company header and footer. Pass layout values as
**attributes of the `t-call`** (since 19.1 a `t-set` inside the `t-call` body no longer
reaches the layout):

```xml
<!-- ❌ Odoo 19: company/title are ignored by the layout in 20 -->
<t t-call="web.external_layout">
    <t t-set="company" t-value="doc.company_id"/>
    <t t-set="layout_document_title">Order</t>
    <div class="page">...</div>
</t>

<!-- ✅ Odoo 20 (pattern used by sale.report_saleorder_document) -->
<t t-set="layout_document_title">
    <span t-field="doc.name"/>
</t>
<t t-call="web.external_layout"
   company="doc.company_id.sudo()"
   layout_document_title="layout_document_title">
    <div class="page">...</div>
</t>
```

#### Layout Variants

`web.external_layout` dispatches to the company's chosen layout (`report.layout` records):

| Layout                         |
| ------------------------------ |
| `web.external_layout_standard` |
| `web.external_layout_bubble`   |
| `web.external_layout_wave`     |
| `web.external_layout_folder`   |
| `web.external_layout_center`   |
| `web.external_layout_dual`     |
| `web.external_layout_lines`    |
| `web.external_layout_compact`  |

`web.internal_layout` and `web.basic_layout` remain for internal/minimal documents.

---

## Custom Reports

For custom reports, create a report class that overrides `_get_report_values`:

```python
from odoo import models

class ReportMyModuleMyTemplate(models.AbstractModel):
    _name = 'report.my_module.my_template'
    _description = 'My Custom Report'

    def _get_report_values(self, docids, data=None):
        docs = self.env['my.model'].browse(docids)

        return {
            'doc_ids': docids,
            'doc_model': 'my.model',
            'docs': docs,
            'data': data,
            'my_custom_value': self._compute_custom(docs),
        }

    def _compute_custom(self, docs):
        # Custom computation
        return sum(doc.amount for doc in docs)
```

Use custom values in template:

```xml
<template id="my_template">
    <t t-call="web.html_container">
        <t t-foreach="docs" t-as="o">
            <t t-call="web.external_layout">
                <div class="page">
                    <h2>My Report</h2>
                    <p>Total: <span t-out="my_custom_value"/></p>
                </div>
            </t>
        </t>
    </t>
</template>
```

---

## Rendering and Attachments in Python

```python
# PDF bytes + type ('pdf'; 'html' in test mode unless force_report_rendering)
pdf_content, report_type = self.env['ir.actions.report']._render_qweb_pdf(
    'my_module.my_report', res_ids=records.ids)
```

Binary values are raw bytes / `BinaryValue` (19.2–19.3), and `ir.attachment.datas` (base64)
no longer exists in 20: write `raw`.

```python
# ❌ Odoo 19.0 (`datas` field is gone in 20)
self.env['ir.attachment'].create({'name': 'r.pdf', 'datas': base64.b64encode(pdf_content), ...})

# ✅ Odoo 20
self.env['ir.attachment'].create({
    'name': 'r.pdf',
    'raw': pdf_content,
    'res_model': record._name,
    'res_id': record.id,
    'type': 'binary',
})
```

Reading back: `attachment.raw` is a `BinaryValue` (supports the buffer protocol, `bytes(v)`,
`v.content`, `v.open()`, `v.to_base64()`); e.g. the report engine does
`io.BytesIO(attachment.raw)`. `ir.actions.report` stores reports with `attachment` set via
`_prepare_pdf_report_attachment_vals_list()` (also using `raw`). Context
`report_pdf_no_attachment=True` skips both reuse and storage.

---

## Paper Formats

### Built-in Paper Formats

| Format                  | Size   |
| ----------------------- | ------ |
| `base.paperformat_euro` | A4     |
| `base.paperformat_us`   | Letter |

For another size (e.g. `Legal`), define a `report.paperformat` record with that `format`.

### Custom Paper Format

```xml
<record id="paperformat_my_format" model="report.paperformat">
    <field name="name">My Custom Format</field>
    <field name="format">custom</field>
    <field name="page_height">297</field>
    <field name="page_width">210</field>
    <field name="orientation">Portrait</field>
    <field name="margin_top">40</field>
    <field name="margin_bottom">20</field>
    <field name="margin_left">7</field>
    <field name="margin_right">7</field>
    <field name="header_line" eval="False"/>
    <field name="header_spacing">35</field>
    <field name="dpi">90</field>
</record>
```

A named `format` and an explicit `page_width`/`page_height` are mutually exclusive
(`ValidationError`); use `format="custom"` with explicit dimensions.

### Format Attributes

| Attribute           | Description                             |
| ------------------- | --------------------------------------- |
| `format`            | Paper size (A4, Letter, etc.) or custom |
| `page_height`       | Page height in mm                       |
| `page_width`        | Page width in mm                        |
| `orientation`       | Portrait or Landscape                   |
| `margin_top`        | Top margin in mm                        |
| `margin_bottom`     | Bottom margin in mm                     |
| `margin_left`       | Left margin in mm                       |
| `margin_right`      | Right margin in mm                      |
| `header_line`       | Show header line                        |
| `header_spacing`    | Header spacing in mm                    |
| `dpi`               | Resolution (default: 90)                |
| `disable_shrinking` | Disable smart shrinking                 |
| `css_margins`       | Use CSS margins                         |

---

## Translatable Reports

### Two-Template Approach

1. Main template (wrapper)
2. Translatable document template

```xml
<!-- Main template -->
<template id="report_saleorder">
    <t t-call="web.html_container">
        <t t-foreach="docs" t-as="doc">
            <t t-call="sale.report_saleorder_document" t-lang="doc.partner_id.lang"/>
        </t>
    </t>
</template>

<!-- Translatable template -->
<template id="report_saleorder_document">
    <!-- Re-browse with partner language -->
    <t t-set="doc" t-value="doc.with_context(lang=doc.partner_id.lang)"/>
    <t t-call="web.external_layout" company="doc.company_id.sudo()">
        <div class="page">
            <div class="row">
                <div class="col-6">
                    <strong>Invoice address:</strong>
                    <div t-field="doc.partner_invoice_id" t-options='{"widget": "contact", "no_marker": True}'/>
                </div>
            </div>
        </div>
    </t>
</template>
```

### Language Attribute

`t-lang` renders the called template in another language. It is an alias of
`t-options-lang` and only valid on the same node as `t-call` (`SyntaxError` otherwise):

```xml
<t t-call="my.template" t-lang="'fr_FR'"/>
<t t-call="my.template" t-lang="doc.partner_id.lang"/>
```

---

## Barcodes

### Embed Barcode

```xml
<img t-att-src="'/report/barcode/QR/%s' % o.name"/>
<img t-att-src="'/report/barcode/?barcode_type=%s&amp;value=%s&amp;width=%s&amp;height=%s' % ('QR', o.name, 200, 200)"/>
```

The query parameter is `barcode_type` (the route signature), not `type`.

### Barcode Controller Parameters

| Parameter       | Values                                                                 |
| --------------- | ---------------------------------------------------------------------- |
| `barcode_type`  | `QR`, `Code128`, `EAN13`, `EAN8`, `UPCA`, `Standard39`, `auto`, ...    |
| `value`         | Barcode value                                                          |
| `width`         | Width in pixels                                                        |
| `height`        | Height in pixels                                                       |
| `humanreadable` | `1` or `0` (show text below)                                           |
| `quiet`         | `1` (default) or `0`: white margins                                    |
| `mask`          | Mask code drawn on top of a QR code                                    |
| `barLevel`      | QR error correction level (default `L`)                                |

### Example

```xml
<div class="page">
    <h2>Product: <span t-field="o.name"/></h2>
    <img
        t-att-src="'/report/barcode/?barcode_type=%s&amp;value=%s&amp;width=%s&amp;height=%s&amp;humanreadable=1' % ('EAN13', o.barcode, 600, 100)"
        alt="Barcode"/>
</div>
```

---

## QWeb Tips

### Directives

| Directive   | Description                                             |
| ----------- | ------------------------------------------------------- |
| `t-out`     | Escape and output value (`Markup` is output as-is)      |
| `t-field`   | Output field value (formatted)                          |
| `t-if`      | Conditional display                                     |
| `t-elif`    | Else if condition                                       |
| `t-else`    | Else condition                                          |
| `t-foreach` | Loop over collection                                    |
| `t-as`      | Variable name for foreach                               |
| `t-set`     | Set variable                                            |
| `t-call`    | Call another template; pass values as attributes        |
| `t-lang`    | Set language for the called template                    |

❌ `t-esc` / `t-raw` are not compiled anymore in 20 (logged as unknown, render nothing):
use `t-out`.

### t-field Options

```xml
<span t-field="o.date" t-options='{"widget": "date"}'/>
<span t-field="o.amount" t-options='{"widget": "monetary", "display_currency": o.currency_id}'/>
<div t-field="o.partner_id" t-options='{"widget": "contact", "no_marker": True}'/>
```

---

## References

- Odoo 20 documentation `content/developer/reference/backend/reports.rst`
- Source: `odoo/addons/base/models/ir_actions_report.py`,
  `odoo/addons/base/models/report_paperformat.py`, `addons/web/controllers/report.py`,
  `addons/web/views/report_templates.xml`, `addons/base_report_wkhtmltox/`,
  `addons/base_report_paper_muncher/`
