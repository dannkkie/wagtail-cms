# Headless / REST API & Multi-Language Sites

Contents: Wagtail's REST API (headless/API-first setups) · Multi-language sites (vanilla Wagtail i18n + wagtail-localize) · CRX's translation approach

Both topics are "advanced" in the sense that they change the project's shape early — decide on them during initial scoping, not bolted on after most pages exist.

---

## Headless / REST API

Wagtail ships a REST API (`wagtail.api.v2`) for serving page content to a separate frontend (Next.js, a Flutter app, a native mobile client, etc.) instead of — or alongside — server-rendered templates.

**Reconsider Step 0 first.** Headless setups get almost nothing from Wagtail CRX — CRX's value is concentrated in its server-rendered Bootstrap 5 templates, which a headless frontend doesn't use. Default to vanilla Wagtail for any project that's headless-first.

### Wiring up the API

```python
# api.py
from wagtail.api.v2.views import PagesAPIViewSet, ImagesAPIViewSet, DocumentsAPIViewSet
from wagtail.api.v2.router import WagtailAPIRouter

api_router = WagtailAPIRouter("wagtailapi")
api_router.register_endpoint("pages", PagesAPIViewSet)
api_router.register_endpoint("images", ImagesAPIViewSet)
api_router.register_endpoint("documents", DocumentsAPIViewSet)
```
```python
# urls.py
from .api import api_router

urlpatterns = [
    ...
    path("api/v2/", api_router.urls),
]
```
This alone exposes `/api/v2/pages/`, `/api/v2/images/`, `/api/v2/documents/` with filtering, pagination, and nested relationships.

### Controlling what a page model exposes

```python
from wagtail.api import APIField

class BlogPage(Page):
    date = models.DateField()
    body = StreamField([...])

    api_fields = [
        APIField("date"),
        APIField("body"),
    ]
```
Only fields listed in `api_fields` appear in the API response for that page type — list everything the frontend needs, nothing else (avoid leaking internal-only fields by omission, not by trusting defaults).

### StreamField over the API

StreamField blocks serialize to JSON automatically (each block's `value` becomes the JSON value), which is usually enough for simple field blocks. For blocks where the default shape doesn't match what the frontend wants (e.g. a chooser block that should return a full nested object instead of just an ID), override `get_api_representation(self, value, context=None)` on the block.

### Images over the API

```python
from wagtail.images.api.fields import ImageRenditionField

api_fields = [
    APIField("photo", serializer=ImageRenditionField("fill-800x600", source="photo")),
]
```
Returns a ready-to-use rendition URL rather than forcing the frontend to construct one.

### CORS

A separately-hosted frontend needs `django-cors-headers` configured (`CORS_ALLOWED_ORIGINS` pointing at the frontend's domain) — the API is otherwise same-origin-restricted like the rest of Django.

### Preview

Live preview-while-editing is straightforward in server-rendered Wagtail (it just renders the page template) but needs deliberate setup in headless mode, since there's no server-rendered template to preview — the frontend has to render a draft state itself. Plan for this explicitly rather than discovering the gap after editors start asking "why can't I preview my changes."

---

## Multi-language sites

### Vanilla Wagtail (native i18n + wagtail-localize)

Wagtail's core has built-in multi-locale support:
```python
WAGTAIL_I18N_ENABLED = True
WAGTAIL_CONTENT_LANGUAGES = LANGUAGES = [
    ("en", "English"),
    ("fr", "Français"),
]
```
This enables a `Locale` model and a `locale` field on every `Page`. Editors create a translated copy of a page tree via `Page.copy_for_translation(locale)`, producing a linked sibling page in the new locale (shares a `translation_key`, independent content). For custom non-page models that need the same treatment, use `wagtail.models.TranslatableMixin`.

Wagtail core gives you the *data model* for this; it doesn't give you a translation workflow UI by itself. **`wagtail-localize`** (separate pip package) is the standard addition — it adds a proper translator UI (segment-by-segment, source/target side by side), sync between source and translated pages as the source changes, and pluggable machine-translation backends (DeepL, etc.) for a first-pass draft. Reach for it as soon as a project has more than one or two languages or any non-technical translator involved — hand-rolling the sync logic `copy_for_translation` leaves open is not worth it past a trivial case.

### CRX's translation approach (different from vanilla Wagtail's)

CRX's own how-to guide recommends a **different** package — `wagtail-modeltranslation` — rather than `wagtail-localize`. This isn't an oversight; it follows from how CRX's abstract page models are structured (heavy StreamField usage, fields not all set up for the native `Locale`/`TranslatableMixin` system out of the box). `wagtail-modeltranslation` works differently: it duplicates translatable fields per language at the database level (e.g. `title_en`, `title_fr`) on the *same* page row, rather than Wagtail-core's separate-linked-page-per-locale model.

Practical implications for a CRX project going multi-language:
- Some CRX model fields aren't exposed to the translation package by default — they need to be explicitly re-declared in your concrete page model's `content_panels` (CRX's own example: pulling `body` out of `body_content_panels` and into `content_panels` on a `WebPage` subclass so the translation package can see it).
- Don't mix the two approaches (`wagtail-localize`'s locale-per-page-tree model and `wagtail-modeltranslation`'s field-suffix model) in the same project — pick the one CRX's own docs point to (`wagtail-modeltranslation`) when working in CRX, even if a different vanilla-Wagtail project on the side uses `wagtail-localize`.
- `LANGUAGE_CODE` and `LANGUAGES` in `settings/base.py` control browser-facing default-language signaling (accessibility-relevant) but do not by themselves translate or enable multiple languages — they're necessary but not sufficient.

If a CRX project's multi-language needs turn out to be substantial (many languages, professional translation workflow, machine-translation drafts), that complexity mismatch is itself a signal worth raising against Step 0 in `SKILL.md` — vanilla Wagtail plus `wagtail-localize` is the better-supported combination for serious multi-language work.
