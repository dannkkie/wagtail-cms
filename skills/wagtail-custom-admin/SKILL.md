---
name: wagtail-custom-admin
description: 'Customize and extend the Wagtail admin interface. Use this any time the user mentions customizing the Wagtail admin, adding admin views, ModelAdmin, SnippetViewSet, custom panels or widgets, admin hooks, moderation workflow customization, custom choosers, or extending the Wagtail admin with React components via the telepath adapter system.'
---

# Wagtail Custom Admin

## Orientation

Wagtail's admin is built on a hook system (`wagtail_hooks.py`), panel composition (each page type declares its `content_panels`), and Django's class-based views. Extending it means working WITH these patterns, not against them.

The two big axes of customization are:
1. **Per-page-type** — custom panels, widgets, and StreamField blocks for a specific page model
2. **Global** — admin menu items, dashboard panels, moderation steps, custom admin views

## Custom Panels

### FieldPanel (most common)

```python
from wagtail.admin.panels import FieldPanel, MultiFieldPanel

class BlogPage(Page):
    content_panels = Page.content_panels + [
        MultiFieldPanel([
            FieldPanel('intro'),
            FieldPanel('body'),
        ], heading='Content'),
        MultiFieldPanel([
            FieldPanel('header_image'),
            FieldPanel('author'),
        ], heading='Metadata'),
    ]
```

### TabbedInterface — when there are many panels

```python
from wagtail.admin.panels import TabbedInterface, ObjectList

class BlogPage(Page):
    content_panels = Page.content_panels + [
        # Content tab
        FieldPanel('intro'),
        FieldPanel('body'),
        FieldPanel('header_image'),
    ]

    promote_panels = Page.promote_panels + [
        FieldPanel('categories'),
        FieldPanel('tags'),
    ]

    settings_panels = Page.settings_panels + [
        FieldPanel('canonical_url'),
    ]

    edit_handler = TabbedInterface([
        ObjectList(content_panels, heading='Content'),
        ObjectList(promote_panels, heading='Promote'),
        ObjectList(settings_panels, heading='Settings'),
    ])
```

### Conditional panels

```python
from wagtail.admin.panels import FieldPanel

class ConditionalFieldPanel(FieldPanel):
    def on_model_bound(self):
        super().on_model_bound()
        # Example: only show this panel if the page has a specific template
        if self.model.template != 'custom_template.html':
            self.widget = forms.HiddenInput()
```

Or use `FieldPanel` with `classname` and `show_if` attributes:

```python
FieldPanel('custom_cta', classname='show-if-template-xyz'),
```

### Bindings for panels

Panels can bind to model fields, computed properties, or model methods:

```python
FieldPanel('full_name')  # FK: my_field.full_name is accessed via __getattr__
FieldPanel('get_readable_date')  # Calls page.get_readable_date()
```

## Custom Widgets

```python
from django import forms

class ColorPickerWidget(forms.TextInput):
    template_name = 'widgets/color_picker.html'

    def get_context(self, name, value, attrs):
        ctx = super().get_context(name, value, attrs)
        ctx['widget']['type'] = 'color'
        return ctx

class BlogPage(Page):
    accent_color = models.CharField(max_length=7, default='#000000')

    content_panels = Page.content_panels + [
        FieldPanel('accent_color', widget=ColorPickerWidget),
    ]
```

## ModelAdmin / SnippetViewSet

Wagtail 5.0+ uses `SnippetViewSet`; older versions use `ModelAdmin`. Use `SnippetViewSet` for new projects.

```python
from wagtail.admin.panels import FieldPanel
from wagtail.snippets.views.snippets import SnippetViewSet
from wagtail.snippets.models import register_snippet

class PersonViewSet(SnippetViewSet):
    model = Person
    icon = 'user'
    menu_label = 'People'
    menu_order = 200
    add_to_admin_menu = True
    list_display = ('name', 'job_title', 'website')
    list_filter = ('job_title',)
    search_fields = ('name', 'bio')
    panels = [
        FieldPanel('name'),
        FieldPanel('job_title'),
        FieldPanel('photo'),
        FieldPanel('bio'),
        FieldPanel('website'),
    ]

register_snippet(Person, PersonViewSet)
```

## Hooks

All hooks go in a `wagtail_hooks.py` file at the app root.

```python
# wagtail_hooks.py
from wagtail import hooks

@hooks.register('register_admin_urls')
def register_custom_admin_urls():
    from django.urls import path
    from . import views
    return [
        path('custom-dashboard/', views.custom_dashboard, name='custom_dashboard'),
    ]

@hooks.register('construct_main_menu')
def hide_items_for_non_admins(request, menu_items):
    if not request.user.is_superuser:
        menu_items[:] = [item for item in menu_items if item.name != 'settings']

@hooks.register('construct_homepage_panels')
def add_custom_dashboard_panel(request, panels):
    panels.append(MyDashboardPanel())

@hooks.register('after_create_page')
def after_page_create(request, page):
    page.owner = request.user
    page.save()

@hooks.register('before_publish_page')
def before_publish_page(request, page):
    # Validation before publish
    if page.title == '':
        raise ValidationError("Title required before publishing")

@hooks.register('register_rich_text_features')
def register_blockquote_feature(features):
    features.default_features.append('blockquote')

@hooks.register('register_icons')
def register_custom_icons(icons):
    return icons + [
        'icons/custom.svg',
        'icons/another.svg',
    ]

@hooks.register('insert_global_admin_css')
def global_admin_css():
    return '<link rel="stylesheet" href="{}">'.format(static('css/admin.css'))

@hooks.register('insert_global_admin_js')
def global_admin_js():
    return '<script src="{}"></script>'.format(static('js/admin.js'))

@hooks.register('insert_editor_js')
def editor_js():
    return '<script src="{}"></script>'.format(static('js/editor.js'))
```

### Common hook reference

| Hook | When it fires | Use for |
|---|---|---|
| `register_admin_urls` | Admin URL conf setup | Custom admin views |
| `construct_main_menu` | Before admin menu render | Reorder/hide menu items |
| `construct_homepage_panels` | Dashboard render | Custom dashboard widgets |
| `after_create_page` | After page saved in admin | Default owner, auto-tag |
| `before_publish_page` | Before page published | Validation, checks |
| `after_publish_page` | After page published | Notifications, sync |
| `before_delete_page` | Before page deletion | Permission checks, cleanup |
| `after_delete_page` | After page deletion | Cache invalidation |
| `register_rich_text_features` | Rich text config | Add/remove editor features |
| `insert_editor_js` | Edit view | Custom StreamField widgets |
| `insert_global_admin_js` | All admin pages | Admin-wide JS |
| `insert_global_admin_css` | All admin pages | Admin-wide CSS |

## Custom Choosers

### Chooser for a custom model

```python
from wagtail.admin.viewsets.chooser import ChooserViewSet

class ProductChooserViewSet(ChooserViewSet):
    model = Product
    icon = 'snippet'
    choose_one_text = 'Choose a product'
    choose_another_text = 'Choose another product'
    edit_item_text = 'Edit this product'
    fields = ['name', 'price', 'thumbnail']
```

### Using a chooser in a StreamField block

```python
from wagtail import blocks

class ProductChooserBlock(blocks.ChooserBlock):
    target_model = Product

    class Meta:
        icon = 'snippet'
        template = 'blocks/product_chooser.html'

    def get_api_representation(self, value, context=None):
        if value:
            return {
                'id': value.id,
                'name': value.name,
                'price': str(value.price),
                'thumbnail': value.thumbnail.get_rendition('fill-300x200').url,
            }
        return None
```

## Telepath / React in the Admin

Wagtail uses a JavaScript adapter system called Telepath to connect Python admin components to client-side JavaScript. Custom StreamField blocks that need rich interactivity use this:

```python
from wagtail.admin.panels import FieldPanel
from wagtail.telepath import register

class InteractiveWidget(forms.Widget):
    template_name = 'widgets/interactive.html'

    class Media:
        js = ['js/interactive-widget.js']

class InteractiveWidgetAdapter(WidgetAdapter):
    js_constructor = 'mysite.widgets.InteractiveWidget'

    def js_args(self, widget):
        return [
            widget.attrs.get('data-config', '{}'),
        ]

register(InteractiveWidgetAdapter(), InteractiveWidget)
```

For simple interactivity, use `insert_editor_js` hook. For complex StreamField blocks that need to manage their own state in the admin, use Telepath adapters.

## Workflow & Moderation

```python
# settings.py
WAGTAIL_WORKFLOW_ENABLED = True

# In code — create a task type
from wagtail.models import Task

class SeniorReviewTask(Task):
    def get_approve_url(self):
        return reverse('senior_review:approve', args=[self.id])
```

```python
# Custom workflow logic
from wagtail.signals import workflow_approved

@receiver(workflow_approved)
def on_workflow_approved(sender, instance, **kwargs):
    # Send notification, sync to external system
    pass
```

## Admin Performance

- **Don't query too much in panel methods.** `FieldPanel('get_related_items')` that does 50 queries will slow the edit view.
- **Use `defer_streamfields()`** on listing pages in the admin.
- **`prefetch_related` in `get_queryset` overrides** for custom `SnippetViewSet` / `ModelAdmin`.
- **Admin CSS/JS should be minimal** — large bundles slow every admin page load.
