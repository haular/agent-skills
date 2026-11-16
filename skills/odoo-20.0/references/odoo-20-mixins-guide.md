# Odoo 20 Mixins Guide

Guide for using Odoo 20 mixins: mail.thread, tracking, activities, email aliases, and other useful mixins.

## Table of Contents
- [Messaging Features](#messaging-features)
- [Logging Changes (Tracking)](#logging-changes-tracking)
- [Activities](#activities)
- [Email Aliases](#email-aliases)
- [UTM Mixin](#utm-mixin)
- [Website Mixins](#website-mixins)
- [Rating Mixin](#rating-mixin)
- [Portal Access](#portal-access)

---

## Messaging Features

### Basic Messaging Integration

Add `mail.thread` mixin to your model:

```python
class BusinessTrip(models.Model):
    _name = 'business.trip'
    _inherit = ['mail.thread']
    _description = 'Business Trip'

    name = fields.Char()
    partner_id = fields.Many2one('res.partner', 'Responsible')
    guest_ids = fields.Many2many('res.partner', string='Participants')
```

`mail.thread` inherits `bus.listener.mixin` and, new in 20, `mail.track.mixin` (value
tracking core, see below).

Add chatter to form view:

```xml
<form string="Business Trip">
    <sheet>
        <!-- Your form fields here -->
    </sheet>
    <chatter open_attachments="True"/>
</form>
```

### Chatter Options

| Option | Description |
|--------|-------------|
| `open_attachments` | Shows attachment section expanded |
| `reload_on_attachment` | Reload form on attachment change |
| `reload_on_follower` | Reload form on follower update |
| `reload_on_post` | Reload form on message post |

---

### Posting Messages

#### message_post

Post a new message on one record (keyword-only arguments):

```python
record.message_post(
    body=Markup('<p>Order <b>%s</b> confirmed</p>') % record.name,
    subject='Subject',
    message_type='comment',
    subtype_xmlid='mail.mt_comment',
    attachments=[('report.pdf', pdf_bytes)],
)
```

**Parameters**:
- `body` (str | Markup): Message body (escaped if str, use Markup for HTML)
- `subject` (str): Message subject
- `message_type` (str): `notification`, `comment`, ... (not `user_notification`, reserved for `message_notify`)
- `subtype_xmlid` / `subtype_id`: Message subtype (`subtype` is a forbidden kwarg)
- `parent_id` (int): Reply to message
- `partner_ids` / `partner_cc_ids` (list[int]): Extra recipients
- `attachments` (list): `(name, content)` or `(name, content, info)` tuples, content is
  raw bytes, **not base64**
- `attachment_ids` (list[int]): Existing attachments (not commands)
- `body_is_html` (bool): Treat a str body as HTML (RPC use)
- `**kwargs`: Extra mail.message field values

Must be called on a single business record (`ensure_one()`); use `message_notify()` to
notify users without a document.

#### _message_log

Log a note without notifying followers:

```python
record._message_log(body=_('Imported from %s', source))
```

#### message_post_with_source

Post using a mail template or a QWeb view (replaces `message_post_with_view` /
`message_post_with_template`, which do not exist):

```python
# ❌ Removed API
record.message_post_with_view('my_module.my_template', values={'val': value})
record.message_post_with_template(template.id, composition_mode='comment')

# ✅ Odoo 20
record.message_post_with_source(
    'my_module.my_template',          # xmlid of an ir.ui.view or mail.template, or a record
    render_values={'val': value},
    subtype_xmlid='mail.mt_note',
)
record.message_post_with_source(
    mail_template,
    email_layout_xmlid='mail.mail_notification_light',
    subtype_xmlid='mail.mt_comment',
)
```

To log a QWeb view without notification: `record._message_log_with_view(view_ref, render_values=...)`.

---

### Receiving Messages

#### message_new

Called when new email arrives for an alias:

```python
def message_new(self, msg_dict, custom_values=None):
    # Extract data from email
    name = msg_dict.get('subject', 'New')
    # Create record
    return super().message_new(msg_dict, {
        'name': name,
        **(custom_values or {}),
    })
```

#### message_update

Called when email reply arrives:

```python
def message_update(self, msg_dict, update_vals=None):
    # Update record from email
    return super().message_update(msg_dict, {
        'description': msg_dict.get('body'),
        **(update_vals or {}),
    })
```

---

### Followers Management

#### message_subscribe

Add partners as followers (followers are partners only; there is no `channel_ids`):

```python
# Subscribe partners
record.message_subscribe(partner_ids=[pid1, pid2])

# With specific subtypes
record.message_subscribe(
    partner_ids=[pid1],
    subtype_ids=[subtype_id],
)
```

#### message_unsubscribe

Remove followers:

```python
record.message_unsubscribe(partner_ids=[pid1, pid2])

# Current user
record.message_unsubscribe(partner_ids=self.env.user.partner_id.ids)
```

---

## Logging Changes (Tracking)

Enable field tracking on a model inheriting `mail.thread` (the `tracking` field parameter
is validated by `mail.track.mixin`):

```python
class MyModel(models.Model):
    _name = 'my.model'
    _inherit = ['mail.thread']

    name = fields.Char(tracking=True)
    state = fields.Selection([
        ('draft', 'Draft'),
        ('done', 'Done'),
    ], tracking=True)

    # Integer = display order of the value in the tracking message (True = 100)
    partner_id = fields.Many2one('res.partner', tracking=1)
```

Disable tracking with context `tracking_disable=True` or `mail_notrack=True`.

### Tracking Hooks (renamed in 20)

The tracking engine was moved to `mail.track.mixin` and its hooks renamed. Old overrides are
silently ignored: rename them.

| Odoo 19.0                                       | Odoo 20                                                      |
| ----------------------------------------------- | ------------------------------------------------------------ |
| `_track_subtype(self, init_values)`             | `_track_log_get_default_subtype(self, track_init_values)`    |
| `_track_template(self, changes)`                | `_track_template_parameters(self, tracked_fields)`           |
| `_track_get_default_log_message(self, tracked_fields)` | `_track_log_get_default_body(self, track_init_values)` |
| `_track_set_author(self, author)`               | `_track_set_log_author(self, author)`                        |
| `_track_filter_for_display(self, tracking_values)` | removed                                                   |

```python
# ✅ Odoo 20: subtype depending on the tracked change (track_init_values has only changed fields)
def _track_log_get_default_subtype(self, track_init_values):
    self.ensure_one()
    if 'state' in track_init_values and self.state == 'confirmed':
        return self.env.ref('my_module.mt_state_change')
    return super()._track_log_get_default_subtype(track_init_values)

# ✅ Odoo 20: send a template when a tracked field changes
def _track_template_parameters(self, tracked_fields):
    res = super()._track_template_parameters(tracked_fields)
    record = self[0]
    if 'stage_id' in tracked_fields and record.stage_id.mail_template_id:
        res['stage_id'] = (record.stage_id.mail_template_id, {
            'auto_delete_keep_log': False,
            'subtype_id': self.env['ir.model.data']._xmlid_to_res_id('mail.mt_note'),
        })
    return res
```

Creation hooks are unchanged: `_creation_subtype()` and `_creation_message()`.

---

## Activities

### mail.activity.mixin

Add activity support:

```python
class MyModel(models.Model):
    _name = 'my.model'
    _inherit = ['mail.thread', 'mail.activity.mixin']

    name = fields.Char()
```

### Activity Methods

```python
# Schedule activity
record.activity_schedule(
    'mail.mail_activity_data_todo',
    user_id=user.id,
    summary='Review this',
)

# New in 20: assign to the user stored in a field of the record
# (ignored if user_id is given)
record.activity_schedule(
    'mail.mail_activity_data_todo',
    activity_user_id_fname='user_id',
)

# Mark matching activities done / remove them (by activity type xmlids)
record.activity_feedback(['mail.mail_activity_data_todo'], feedback='Completed')
record.activity_unlink(['mail.mail_activity_data_todo'])

# On mail.activity records
activities = record.activity_ids
activities.action_done()
activities.action_feedback(feedback='Completed')
```

Context `mail_activity_automation_skip=True` makes `activity_schedule` a no-op.

---

## Email Aliases

### mail.alias.mixin

Add email alias support. The mixin `_inherits` `mail.alias` and defines the required
`alias_id` itself; do not redeclare it. Use `mail.alias.mixin.optional` when the alias is
optional.

```python
class MyModel(models.Model):
    _name = 'my.model'
    _inherit = ['mail.alias.mixin', 'mail.thread']

    name = fields.Char()

    def _alias_get_creation_values(self):
        values = super()._alias_get_creation_values()
        values['alias_model_id'] = self.env['ir.model']._get_id(self._name)
        if self.id:
            values['alias_defaults'] = {'name': self.name}
        return values
```

Create alias in data file:

```xml
<record id="my_alias" model="mail.alias">
    <field name="alias_name">my-model</field>
    <field name="alias_model_id" ref="model_my_model"/>
</record>
```

---

## UTM Mixin

### utm.mixin

Add campaign tracking. The mixin provides `campaign_id`, `source_id`, `medium_id`; do not
redeclare them:

```python
class MyModel(models.Model):
    _name = 'my.model'
    _inherit = ['utm.mixin']

    name = fields.Char()
```

---

## Website Mixins

### website.published.mixin

Add website publishing. Provides `is_published` (stored), `website_published` (related to
`is_published`), `can_publish`, and new in 20 `publish_on` (auto-publish datetime) and
`published_date`. `website_url` (compute `_compute_website_url`) now comes from the new
parent `website.located.mixin`:

```python
class MyModel(models.Model):
    _name = 'my.model'
    _inherit = ['website.published.mixin']

    name = fields.Char()
```

### website.seo.metadata

Add SEO metadata. Provides `website_meta_title`, `website_meta_description`,
`website_meta_keywords`, `website_meta_og_img`, `seo_name` (translatable):

```python
class MyModel(models.Model):
    _name = 'my.model'
    _inherit = ['website.seo.metadata']

    name = fields.Char()
```

Model code must not use `request`: the current website is `self.env.website` (19.4).

---

## Rating Mixin

### rating.mixin

Add customer rating (`rating.mixin` already inherits `mail.thread`):

```python
class MyModel(models.Model):
    _name = 'my.model'
    _inherit = ['rating.mixin']

    name = fields.Char()
```

### Rating Methods

```python
# Send rating request: a mail.template record, not an xmlid string
template = self.env.ref('my_module.mail_template_rating_request')
record.rating_send_request(template, lang=record.partner_id.lang, force_send=True)

# Get rating stats
stats = record.rating_get_stats()
grades = record.rating_get_grades()
```

---

## Portal Access

### portal.mixin

Add customer portal access (provides `access_url`, `access_token`, `access_warning`):

```python
class MyModel(models.Model):
    _name = 'my.model'
    _inherit = ['portal.mixin']

    name = fields.Char()
    partner_id = fields.Many2one('res.partner', 'Customer')
```

Override access URL:

```python
def _compute_access_url(self):
    super()._compute_access_url()
    for record in self:
        record.access_url = '/my/model/%s' % record.id
```

Build links with `record.get_portal_url(suffix=None, report_type=None, download=None, ...)`.

---

## References

- Odoo 20 documentation `content/developer/reference/backend/mixins.rst`
- Source: `addons/mail/models/mail_thread.py`, `addons/mail/models/mail_track_mixin.py`,
  `addons/mail/models/mail_activity_mixin.py`, `addons/mail/models/mail_alias_mixin.py`,
  `addons/utm/models/utm_mixin.py`, `addons/website/models/mixins.py`,
  `addons/rating/models/`, `addons/portal/models/portal_mixin.py`
