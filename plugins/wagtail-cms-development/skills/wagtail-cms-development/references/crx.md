# Wagtail CRX (CodeRed CMS) — Deep Reference

Contents: Installation · The abstract→concrete pattern · Parent/child page rules · Built-in page types · Content & layout block catalog · Snippets · Settings models · SEO · Forms · Writing a custom page type · Custom image/document models · `CRX_*` Django settings · The Bootstrap 5 constraint

Official site: coderedcorp.com/cms · Docs: docs.coderedcorp.com/wagtail-crx · "CRX, formerly CodeRed CMS, is not a fork of Wagtail — it's a pip package of additional features on top of stock Wagtail, the way Wagtail itself is additional features on top of Django."

---

## Installation

```bash
pip install coderedcms
coderedcms start mysite --template pro --sitename "My Company Inc." --domain "www.example.com"
cd mysite
pip install -r requirements-dev.txt   # pro template only — ruff, mypy, pytest
python manage.py migrate
python manage.py createsuperuser
python manage.py runserver
```
- `--sitename`/`--domain` just pre-populate settings — not load-bearing, fine to skip.
- `basic` template (the default if `--template` omitted) is bare-bones. `pro` adds: custom Image/Document models, a custom User model (email as username), custom Navbar/Footer, SCSS→CSS compilation via Python (`django-sass`, no Node.js needed), and dev tooling (ruff, mypy, pytest) pre-configured. **Use `pro` for anything beyond a disposable prototype** — switching Image/Document/User models after the fact is a real migration, not a config flag.
- To recompile SCSS after editing: `python manage.py sass website/static/website/src/custom.scss website/static/website/css/custom.css`, add `--watch` during active development.

---

## The abstract→concrete pattern

CRX ships **abstract** base page classes (`CoderedPage` at the root, then `CoderedWebPage`, `CoderedArticlePage`, `CoderedArticleIndexPage`, `CoderedEventPage`, `CoderedEventIndexPage`, `CoderedFormPage`, `CoderedLocationPage`, `CoderedLocationIndexPage`). The `coderedcms start` scaffold pre-generates **concrete** subclasses of these in your own `website/models.py` — `ArticlePage`, `ArticleIndexPage`, `WebPage`, etc. **These generated concrete classes are yours to edit directly** — add fields, override `parent_page_types`, change the template. This is the normal, expected workflow, not a framework internal to avoid touching.

Why abstract: keeping the actual functionality in abstract base classes means CRX upgrades can land core fixes without forcing migration conflicts in your concrete models — each project's migrations stay project-specific.

```python
from modelcluster.fields import ParentalKey
from coderedcms.models import (
    CoderedArticlePage,
    CoderedArticleIndexPage,
    CoderedFormPage,
    CoderedWebPage,
)

class WebPage(CoderedWebPage):
    class Meta:
        verbose_name = "Web Page"
    template = "coderedcms/pages/web_page.html"

class ArticlePage(CoderedArticlePage):
    class Meta:
        verbose_name = "Article"
        ordering = ["-first_published_at"]
    parent_page_types = ["website.ArticleIndexPage"]
    template = "coderedcms/pages/article_page.html"
    search_template = "coderedcms/pages/article_page.search.html"
```

---

## Parent/child page rules

CRX's built-in page types enforce a specific tree shape out of the box:

| Parent page type | Allowed child page types |
|---|---|
| Web Page | Web Page, Article Landing Page, Event Landing Page, Location Landing Page, Form Page |
| Article Landing Page | Article Page |
| Event Landing Page | Event Page |
| Location Landing Page | Location Page |

Custom page types follow the same `parent_page_types`/`subpage_types` mechanism as vanilla Wagtail — set them explicitly on any new concrete page class rather than relying on defaults.

---

## Built-in page types

- **Web Page** (`CoderedWebPage`) — general-purpose page with the full layout/content StreamField and SEO fields. The default choice for "just a page."
- **Article Pages** (`CoderedArticlePage` + `CoderedArticleIndexPage`) — blog/news. Index page auto-lists children; supports authorship and publish-date display.
- **Event Pages** (`CoderedEventPage` + `CoderedEventIndexPage`) — calendar/event listings with recurrence support.
- **Form Pages** (`CoderedFormPage`) — see Forms section below.
- **Location Pages** (`CoderedLocationPage` + `CoderedLocationIndexPage`) — store-locator style, with Google Maps integration.
- **Stream Forms** — a more flexible form-building variant for advanced field arrangements.

Every `Codered*Page` carries SEO fields out of the box (meta title/description, Open Graph, structured/schema.org data) via integration with the `wagtail-seo` package — no extra setup needed per page.

---

## Content & layout block catalog

Don't hand-roll a block CRX already ships — check this list first.

**Layout blocks** (structure/composition): Hero Block, Responsive Grid Row Block, Column Block, Card Grid Block, HTML Block.

**Content blocks** (things you put inside the layout): Accordion, Button, Card, Carousel, Download, Embed Media (oEmbed — YouTube, etc.), Film Strip, Google Map, HTML, Image, Image Gallery, Image Link, Latest Pages (auto-populated list of recent pages of a chosen type), Modal, Page Preview (card-style link to another page), Price List, Quote, Reusable Content (insert a snippet of shared content), Table, Text.

All are Bootstrap 5-targeted — verify a candidate block's rendered markup matches the site's actual Bootstrap usage before assuming it'll look right unstyled.

---

## Snippets

CRX ships ready-made snippet types for common reusable content: Navigation Bars, Footers, Classifiers (a tagging/categorization system for filterable content — e.g. tag articles by topic and auto-generate filtered listings), Reusable Content, Carousels, Accordions, Content Walls (e.g. age-gates or paywalls), Film Strip. Reach for these before building a custom snippet that duplicates one.

---

## Settings models

- **Layout Settings** (admin label "CRX Settings") — logo, favicon, Google Maps API key, and other site-wide layout config. Backs a *lot* of CRX's rendering — disabling it (`CRX_DISABLE_LAYOUT`) is rarely the right move even when customizing the navbar/footer (use the narrower `CRX_DISABLE_NAVBAR`/`CRX_DISABLE_FOOTER` instead, see below).
- **Analytics Settings** (admin label "Tracking") — Google Analytics and other tracking script configuration, plus per-button/link "Advanced Settings" for event tracking.
- Navbar/Footer settings — editable via the admin unless disabled (see `CRX_*` settings below) in favor of a fully custom-coded nav/footer.

---

## SEO

Every Codered page model inherits rich SEO attributes via `wagtail-seo`: `seo_title`, `seo_description`, `seo_author`, `seo_published_at`, Open Graph image/type, and structured (schema.org) data — all editable per-page in the Promote tab, with sane auto-generated defaults if left blank. Don't hand-roll `<meta>` tags in custom templates that extend a Codered base template — they're already in `base.html` via `{% include "wagtailseo/meta.html" %}` and `{% include "wagtailseo/struct_data.html" %}`.

---

## Forms

`CoderedFormPage` (+ `CoderedFormField` for individual fields, `CoderedEmail` for confirmation emails) is considerably more capable than vanilla Wagtail's bundled form module:
- Multi-step forms.
- Conditional field logic (show/hide fields based on other answers).
- Customized confirmation emails per form.
- MailChimp integration (subscribe submitters to a list).
- File-upload fields write to `CRX_PROTECTED_MEDIA_ROOT`, served through Django (not directly by the web server) and requiring login — keep upload whitelist/blacklist settings (below) tight in production.

Reach for `CoderedFormPage` before vanilla Wagtail's form module on a CRX project — it's the better-supported path and the one CRX's templates assume.

---

## Writing a custom page type

The standard pattern (e.g. adding a "Product Page" that doesn't fit Web/Article/Event/Location):

```python
from django.db import models
from wagtail.admin.panels import FieldPanel
from wagtail.fields import RichTextField
from wagtail.images import get_image_model_string
from coderedcms.models import CoderedWebPage

class ProductIndexPage(CoderedWebPage):
    class Meta:
        verbose_name = "Product Landing Page"
    index_query_pagemodel = "website.ProductPage"  # what .get_index_children() returns
    subpage_types = ["website.ProductPage"]
    template = "website/pages/product_index_page.html"

class ProductPage(CoderedWebPage):
    class Meta:
        verbose_name = "Product Page"
    parent_page_types = ["website.ProductIndexPage"]
    template = "website/pages/product_page.html"

    description = RichTextField(verbose_name="Product Description", blank=True)
    photo = models.ForeignKey(
        get_image_model_string(), null=True, blank=True,
        on_delete=models.SET_NULL, related_name="+",
        verbose_name="Product Photo",
    )

    body_content_panels = CoderedWebPage.body_content_panels + [
        FieldPanel("description"),
        FieldPanel("photo"),
    ]
```
Template extends the matching Codered base and loads CRX's template tags alongside the standard Wagtail ones:
```html
{% extends "coderedcms/pages/web_page.html" %}
{% load wagtailcore_tags wagtailimages_tags coderedcms_tags %}
```
Then the normal Django/Wagtail cycle: `makemigrations` → `migrate` → create the page in the admin under its allowed parent.

---

## Custom image/document models

The `pro` template sets this up from project creation. Retrofitting a `basic`-template project to a custom Image model later is a real data migration (not unique to CRX — same as any Wagtail project) — if there's any chance of needing custom image fields (e.g. photo credit, alt-text policy enforcement) down the line, start with `pro` or do the custom-model migration early, before there's production image data to migrate.

---

## `CRX_*` Django settings

Defined in your project's settings, defaults come from `coderedcms/settings.py`:

- `CRX_BANNER` — text (HTML allowed) shown as a banner on both front-end and admin; useful for flagging staging/non-production. `CRX_BANNER_BACKGROUND` / `CRX_BANNER_TEXT_COLOR` customize its colors.
- `CRX_DISABLE_ANALYTICS` — hides the Tracking settings model entirely.
- `CRX_DISABLE_LAYOUT` — hides CRX Settings (logo, favicon, Maps key, etc.) — used **extensively** throughout CRX's templates; disabling can break things unexpectedly. Prefer the narrower flags below for a custom nav/footer.
- `CRX_DISABLE_NAVBAR` / `CRX_DISABLE_FOOTER` — hide just the built-in Navbar/Footer models, for projects building their own more advanced nav/footer while keeping the rest of Layout Settings.
- `CRX_FRONTEND_*` — a family of settings controlling block/page/template rendering defaults, tuned for Bootstrap 5 but overridable for a different CSS framework (see github.com/coderedcorp/coderedcms `settings.py` for the full list — check the version installed, these have grown over releases).
- `CRX_PROTECTED_MEDIA_ROOT` / `CRX_PROTECTED_MEDIA_URL` — where/how form file uploads are stored and served (login-gated).
- `CRX_PROTECTED_MEDIA_UPLOAD_WHITELIST` / `_BLACKLIST` — allowed/disallowed file extensions for form uploads. Blacklist defaults to blocking executables/scripts (`.sh`, `.exe`, `.bat`, `.ps1`, `.app`, `.jar`, `.py`, `.php`, `.pl`, `.rb`) — tighten further with an explicit whitelist (e.g. `['.pdf', '.doc', '.docx', '.jpg', '.jpeg', '.png']`) on any form accepting public uploads.

---

## The Bootstrap 5 constraint — when to stop fighting it

CRX's blocks, page templates, and `CRX_FRONTEND_*` defaults are deliberately, deeply coupled to Bootstrap 5 — this is a stated design choice, not an oversight. It's a huge accelerant when Bootstrap is the design system. The moment a project needs a genuinely different visual language (a custom Tailwind component system, a non-Bootstrap grid, heavily customized card/carousel markup that no longer resembles Bootstrap's), every additional CRX block adopted is markup to override rather than markup to use as-is. At that point, re-raise Step 0 from `SKILL.md` — it may be cheaper to drop to vanilla Wagtail for the affected page types (or the whole project) than to keep overriding CRX templates piece by piece.
