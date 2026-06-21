# Vanilla Wagtail — Deep Reference

Contents: Page models · StreamField & custom blocks · Panels · Snippets · Images & documents · Search · Settings models · Routable pages · QuerySets · Permissions & workflows · Testing · Performance

---

## Page models

A page type is a Django model subclassing `wagtail.models.Page`:

```python
from django.db import models
from wagtail.models import Page
from wagtail.fields import RichTextField
from wagtail.admin.panels import FieldPanel

class BlogIndexPage(Page):
    intro = RichTextField(blank=True)
    content_panels = Page.content_panels + [
        FieldPanel("intro"),
    ]

class BlogPage(Page):
    date = models.DateField("Post date")
    body = RichTextField(blank=True)

    content_panels = Page.content_panels + [
        FieldPanel("date"),
        FieldPanel("body"),
    ]

    parent_page_types = ["blog.BlogIndexPage"]
    subpage_types = []  # leaf page — no children allowed
```

Key attributes:
- `parent_page_types` / `subpage_types` — lists of `"app_label.ModelName"` strings restricting what can be created where. Defaults to "anything" if omitted; set both explicitly once the tree shape is known, otherwise editors can create nonsensical hierarchies.
- `template` — defaults to `app_label/model_name.html` (snake_case); override explicitly when it doesn't match.
- `content_panels`, `promote_panels` (SEO title/slug/meta), `settings_panels` (go-live/expiry dates) — always extend the base list (`Page.content_panels + [...]`), never replace it outright, or you lose the title field and other base functionality.
- `get_context(self, request, *args, **kwargs)` — override to inject extra template context (e.g. paginated child pages), calling `super().get_context(...)` first.
- Pages are **not deleted on app changes** like a normal model — they're tree nodes (django-treebeard under the hood). Don't write raw SQL against the page table; use the ORM/tree methods.

---

## StreamField & custom blocks

This is the highest-leverage part of Wagtail to actually master — it's where "flexible CMS" becomes either elegant or a maintenance swamp depending on how it's used.

### Defining a StreamField

```python
from wagtail.fields import StreamField
from wagtail import blocks
from wagtail.images.blocks import ImageChooserBlock  # simple image chooser
from wagtail.images.blocks import ImageBlock          # newer: chooser + alt text + decorative toggle

class BlogPage(Page):
    body = StreamField([
        ("heading", blocks.CharBlock(form_classname="title")),
        ("paragraph", blocks.RichTextBlock()),
        ("image", ImageBlock()),
    ], blank=True, use_json_field=True)
```

`use_json_field=True` is required explicitly on recent Wagtail versions (it's not implicitly assumed) — store content as native JSON rather than a serialized text blob.

### Built-in field blocks
`CharBlock`, `TextBlock`, `RichTextBlock`, `RawHTMLBlock`, `BooleanBlock`, `IntegerBlock`, `FloatBlock`, `DecimalBlock`, `DateBlock`/`TimeBlock`/`DateTimeBlock`, `EmailBlock`, `URLBlock`, `ChoiceBlock`, `PageChooserBlock`, `DocumentChooserBlock`, `SnippetChooserBlock`. Each accepts the common kwargs (`required`, `help_text`, `default`, `validators`) plus type-specific ones.

### Structural blocks
- **`StructBlock`** — a fixed set of named sub-blocks, like a small form. Access fields by name on the resulting `StructValue` (dict-like).
  ```python
  class QuoteBlock(blocks.StructBlock):
      text = blocks.TextBlock()
      source = blocks.CharBlock(required=False)

      class Meta:
          icon = "openquote"
          template = "blocks/quote_block.html"
  ```
- **`ListBlock`** — repeatable instances of a single block type. `blocks.ListBlock(blocks.CharBlock())` for a simple repeated list, or `blocks.ListBlock(QuoteBlock())` to repeat a struct.
- **`StreamBlock`** — like the top-level StreamField but reusable/nestable; lets editors mix-and-match an arbitrary sequence of *different* block types within a sub-area. Define as its own class when reused across multiple StreamFields:
  ```python
  class CarouselBlock(blocks.StreamBlock):
      image = ImageBlock()
      quote = QuoteBlock()
  ```

Compose these freely — a `StreamBlock` containing `StructBlock`s containing `ListBlock`s of further `StructBlock`s is normal. Keep nesting **shallow** in practice: every extra level makes the editor UI harder to scan and a future refactor more painful. If you're nesting more than 2–3 levels deep, that's usually a sign to rethink the content model rather than push further.

### Custom block classes (full control)

For anything beyond composing built-ins, subclass `blocks.Block` (or more often `blocks.StructBlock`/`blocks.FieldBlock`) and override:
- `get_context(self, value, parent_context=None)` — build the template context dict.
- `render_basic(self, value, context=None)` / a `template` Meta attribute — control output.
- `clean(self, value)` — custom validation; raise `django.core.exceptions.ValidationError` (or `StructBlockValidationError` for struct field-level errors) to block saving as published (note: as-draft saves can bypass `required`-style checks depending on `required_on_save`).

### Previews and descriptions (editor UX)

Recent Wagtail versions support live previews of a block in the block-picker, plus a description shown to editors choosing between similar blocks:

```python
("quote", blocks.StructBlock(
    [("text", blocks.TextBlock()), ("source", blocks.CharBlock())],
    preview_value={"text": "Best CMS I've used.", "source": "A happy editor"},
    preview_template="myapp/previews/blocks/quote.html",
    description="A pull-quote with attribution, rendered as a blockquote.",
))
```
Use this when a site has many similar-looking blocks (e.g. three different "card" variants) — descriptions and previews are the difference between editors picking the right one confidently and guessing.

### The StreamField discipline (read this before reaching for StreamField)

1. **Don't reach for it by default.** A handful of `CharField`/`RichTextField`/`ForeignKey` fields are easier to query, validate, and expose via API than the equivalent StructBlock. Use StreamField when content genuinely varies in shape per page or editors need to reorder/repeat sections.
2. **Never store data you need to query, validate, or expose cleanly via API in StreamField** — titles, SEO metadata, dates, prices, SKUs, important URLs. It's stored as JSON; querying inside it is awkward and migrating it later is painful. Use real model fields for anything load-bearing.
3. **Keep nesting shallow** for the reasons above.
4. **Give blocks icons and descriptions** so the block-picker stays scannable as the catalog grows.

---

## Panels (the editing-form layer)

```python
from wagtail.admin.panels import (
    FieldPanel, MultiFieldPanel, InlinePanel, FieldRowPanel,
    HelpPanel, TabbedInterface, ObjectList,
)
```
- `FieldPanel("field_name")` — the basic building block; works for any model field, including chooser fields (image, page, document, snippet ForeignKeys) — no separate "ChooserPanel" classes needed anymore.
- `MultiFieldPanel([...], heading="...")` — visually groups related fields under a heading.
- `FieldRowPanel([...])` — lays panels out horizontally instead of stacked.
- `InlinePanel("related_name", label="...")` — for editing a related model inline (e.g. a list of team members on a team page). Requires the related model to use `modelcluster.fields.ParentalKey` (not a plain `ForeignKey`) back to the page, plus `wagtail.search.index` only on the page itself.
  ```python
  from modelcluster.fields import ParentalKey
  from modelcluster.models import ClusterableModel

  class TeamMember(models.Model):
      page = ParentalKey("TeamPage", related_name="members", on_delete=models.CASCADE)
      name = models.CharField(max_length=255)
      panels = [FieldPanel("name")]
  ```
- `TabbedInterface` / `ObjectList` — build a fully custom tab layout for the edit form (used for settings models too — see below) when the default Content/Promote/Settings tabs aren't the right shape.

---

## Snippets

Snippets are reusable, non-page content (no tree position, no URL) — a "Person" model used across many pages, or testimonial entries.

```python
from wagtail.snippets.models import register_snippet

@register_snippet
class Testimonial(models.Model):
    author = models.CharField(max_length=255)
    quote = models.TextField()
    panels = [FieldPanel("author"), FieldPanel("quote")]

    def __str__(self):
        return self.author
```

For more control over the admin listing (custom columns, filters, menu placement), use `SnippetViewSet` instead of the bare decorator:
```python
from wagtail.snippets.views.snippets import SnippetViewSet, register_snippet

class TestimonialViewSet(SnippetViewSet):
    model = Testimonial
    list_display = ["author"]
    icon = "quote"

register_snippet(TestimonialViewSet)
```
Reference a snippet from a page via `SnippetChooserBlock` (in StreamField) or a plain `ForeignKey` + `FieldPanel` (for a single fixed relationship).

---

## Images & documents

Never import `wagtail.images.Image` or `wagtail.documents.Document` directly in a model field — use the string/model lookups so the project can swap in a custom Image/Document model later without breaking every reference:

```python
from wagtail.images import get_image_model_string

class ProductPage(Page):
    photo = models.ForeignKey(
        get_image_model_string(),
        null=True, blank=True,
        on_delete=models.SET_NULL,
        related_name="+",
    )
```
`related_name="+"` avoids creating a reverse accessor you'll never use (every page model that has an image FK would otherwise need a unique related_name).

In templates, the `{% image %}` tag generates a rendition on the fly:
```html
{% load wagtailimages_tags %}
{% image page.photo fill-800x600 format-webp as photo %}
<img src="{{ photo.url }}" alt="{{ photo.alt }}">
```
Common filter specs: `fill-WxH` (crop to exact dimensions), `max-WxH` (scale down, preserve aspect), `width-W`/`height-H`, `format-webp`/`format-jpeg`. Renditions are generated once and cached — don't worry about regenerating cost on every request.

---

## Search

```python
from wagtail.search import index

class BlogPage(Page):
    body = RichTextField()
    search_fields = Page.search_fields + [
        index.SearchField("body"),
        index.AutocompleteField("title"),
    ]
```
Query with `Page.objects.live().search("query string")`. The default backend is database search (Postgres-backed full text search if using Postgres, otherwise a weaker SQLite/MySQL fallback) — fine for most small-to-medium sites. Switch to Elasticsearch/OpenSearch only once volume or search-quality needs actually demand it; it's extra infrastructure to run and keep in sync.

---

## Settings models

Two base classes, both registered the same way:

```python
from wagtail.contrib.settings.models import BaseGenericSetting, BaseSiteSetting, register_setting
from wagtail.admin.panels import FieldPanel

@register_setting
class SocialMediaSettings(BaseGenericSetting):  # one set of values, shared across all sites
    facebook = models.URLField(blank=True)
    panels = [FieldPanel("facebook")]

@register_setting
class BrandSettings(BaseSiteSetting):  # one set of values per Site (multi-site projects)
    primary_color = models.CharField(max_length=7, blank=True)
    panels = [FieldPanel("primary_color")]
```
Use `BaseGenericSetting` unless the project genuinely serves multiple `Site`s with different values per site — that's the only case `BaseSiteSetting` earns its extra complexity.

Access in a view: `SocialMediaSettings.load(request_or_site=request)` (generic) or `BrandSettings.for_request(request=request)` (site-specific). In templates, settings are available via the `{% load wagtailcore_tags %}` `{% pageurl %}`-adjacent context processor — check the project's `TEMPLATES` context_processors includes `wagtail.contrib.settings.context_processors.settings` (the CRX and vanilla `wagtail start` scaffolds both include it by default).

---

## Routable pages

For a page that needs custom sub-URLs beyond Wagtail's normal tree routing (e.g. a calendar page with `/events/2026/06/`):
```python
from wagtail.contrib.routable_page.models import RoutablePageMixin, path

class EventIndexPage(RoutablePageMixin, Page):
    @path("")
    def current_events(self, request):
        return self.render(request)

    @path("<int:year>/<int:month>/")
    def events_by_month(self, request, year, month):
        return self.render(request, context_overrides={"year": year, "month": month})
```

---

## QuerySets

`Page` querysets have tree- and status-aware methods beyond standard Django:
- `.live()` — published only. `.live().public()` adds "not behind a password/privacy restriction".
- `.specific()` — converts generic `Page` results into their specific subclass instances (needed to access subclass-only fields) — costs extra queries, so don't call it on large querysets you're not going to render in full.
- `.type(BlogPage)` / `.not_type(...)` — filter by page class.
- `.child_of(page)`, `.descendant_of(page)`, `.sibling_of(page)`, `.ancestor_of(page)` — tree relationships.
- `.in_menu()` — pages flagged `show_in_menus=True`.

---

## Permissions & workflows

Wagtail's permission model is group-based, configured in the admin (Settings → Groups), independent of code changes for most projects. For multi-step editorial review, Wagtail ships built-in moderation **workflows** (Settings → Workflows) — define approval steps without custom code first; only reach for custom workflow tasks if the built-in approve/reject step genuinely doesn't fit.

---

## Testing

```python
from wagtail.test.utils import WagtailPageTestCase

class BlogPageTests(WagtailPageTestCase):
    def test_blog_page_renders(self):
        self.assertPageIsRenderable(self.blog_page)

    def test_routability(self):
        self.assertPageIsRoutable(self.blog_page)
```
`WagtailPageTestCase` saves boilerplate over plain Django `TestCase` + manual `self.client.get()` for page-specific assertions (routability, renderability, correct parent/child type restrictions).

---

## Performance

- **The classic N+1**: looping over a StreamField's chooser blocks (page, image, document, snippet) in a template fires one query per item unless prefetched. For `PageChooserBlock`/`SnippetChooserBlock`-heavy StreamFields rendered in a loop (e.g. a "latest pages" block on an index page), prefetch the underlying objects in the view rather than relying on lazy per-block resolution.
- **`defer_streamfields()`** on a queryset skips parsing StreamField JSON for pages you're only listing (e.g. by title), avoiding unnecessary deserialization cost.
- **Caching**: `wagtail-cache` (a separate pip package, full-page cache keyed on the page's cache-busting on publish/unpublish) is the standard choice for read-heavy marketing/content sites — see `references/deployment.md`.
- **`.specific()` is not free** — only call it on the page(s) you're actually about to render fully, not on a large listing queryset where you only need `title`/`url`.
