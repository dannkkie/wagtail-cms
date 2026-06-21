# wagtail-modeltranslation — Deep Reference

## How It Works

Unlike Wagtail's native i18n (separate page trees per locale), `wagtail-modeltranslation` stores all locale data in the SAME database row by adding locale-prefixed columns. A `BlogPage` with fields `title`, `body`, `intro` in three locales gets nine columns:

```
id | title_en | title_fr | title_de | body_en | body_fr | body_de | intro_en | intro_fr | intro_de
```

When Django's active language is `fr`, `page.title` returns `title_fr` automatically — the modeltranslation middleware transparently maps field access to the correct column.

## Setup

```bash
pip install wagtail-modeltranslation
```

### settings.py

```python
INSTALLED_APPS = [
    'modeltranslation',  # Must be before wagtail and your apps
    'wagtail.contrib.settings',
    # ... your apps
]

LANGUAGES = [
    ('en', 'English'),
    ('fr', 'French'),
]

MODELTRANSLATION_DEFAULT_LANGUAGE = 'en'
# Optional: fallback to default language if translation is missing
MODELTRANSLATION_FALLBACK_LANGUAGES = {'default': ('en',), 'fr': ('en',)}
```

### Registering Models

```python
# translation.py (in your app)
from modeltranslation.translator import translator, TranslationOptions

class BlogPageTranslationOptions(TranslationOptions):
    fields = ('title', 'body', 'intro', 'seo_title', 'search_description')
    # Optional: fields that are always in the default language
    # empty_values = ''  # What value means "not translated"

translator.register(BlogPage, BlogPageTranslationOptions)
```

After registration, run `makemigrations` — modeltranslation adds columns for each locale × field combination. Then `migrate`.

## CRX Integration

CRX ships with modeltranslation pre-configured. In a CRX project:

```python
# settings.py
CRX_ENABLE_MODELTRANSLATION = True
LANGUAGES = [
    ('en', 'English'),
    ('fr', 'French'),
]
```

CRX's abstract page models (`CoderedWebPage`, `CoderedArticlePage`, `CoderedEventPage`, `CoderedLocationPage`, `CoderedFormPage`) are already registered with modeltranslation. You just need to register your concrete subclasses:

```python
# your_app/translation.py
from modeltranslation.translator import translator
from coderedcms.models import CoderedArticlePage

class MyArticlePageTranslationOptions(TranslationOptions):
    fields = ('custom_intro', 'custom_sidebar_content')

translator.register(MyArticlePage, MyArticlePageTranslationOptions)
```

CRX's templates use `{% get_current_language as LANGUAGE_CODE %}` and render the correct locale's content automatically through modeltranslation's field access patching.

## Admin UI

modeltranslation adds tabbed fields in the Wagtail admin — one tab per locale. Each field appears once per locale:

```
[EN] [FR] [DE]
Title (en): [________]
Title (fr): [________]
Title (de): [________]
Body (en):  [________]
Body (fr):  [________]
Body (de):  [________]
```

This is simpler than native i18n's separate-page-per-locale approach but gets unwieldy with many locales (10+ locale tabs are a poor UX). For many-locale projects, consider native i18n instead.

## Template Usage

```django
{% load i18n %}

{# Current language #}
{% get_current_language as LANGUAGE_CODE %}

{# Language switcher #}
<form action="{% url 'set_language' %}" method="post">
    {% csrf_token %}
    <select name="language" onchange="this.form.submit()">
        {% get_available_languages as LANGUAGES %}
        {% for lang_code, lang_name in LANGUAGES %}
            <option value="{{ lang_code }}" {% if lang_code == LANGUAGE_CODE %}selected{% endif %}>
                {{ lang_name }}
            </option>
        {% endfor %}
    </select>
</form>

{# Field access is automatic — modeltranslation patches .title to return the current locale's value #}
<h1>{{ page.title }}</h1>
<div>{{ page.body|richtext }}</div>

{# Explicit locale access when needed #}
<p>English title: {{ page.title_en }}</p>
```

## Limitations

### StreamField Is Not Translatable

This is the biggest limitation. modeltranslation works at the field level, and StreamField stores JSON — you can't have a `body_en` and `body_fr` column for the same StreamField value without a third-party patch. The `wagtail-modeltranslation` package added experimental support via a custom `TranslatableStreamField` but it has edge cases with chooser blocks (images, pages, documents — chooser IDs are locale-agnostic but the chooser overlay is locale-aware).

### Queryset Filtering

Filtering on a translated field requires using the locale-prefixed column name:

```python
# Can't do this — modeltranslation doesn't hook into ORM filters
BlogPage.objects.filter(title="Hello")

# Must do this:
BlogPage.objects.filter(title_en="Hello")
```

For search across multiple locales, you need OR-filtering:

```python
from django.db.models import Q
BlogPage.objects.filter(Q(title_en__icontains="search") | Q(title_fr__icontains="search"))
```

### API Exposure

REST API responses include ALL locale data:

```json
{
    "title_en": "Hello",
    "title_fr": "Bonjour",
    "title_de": "Hallo",
    "body_en": "...",
    "body_fr": "...",
    "body_de": "..."
}
```

This is noisy for headless consumers. You either filter the response server-side or let the client pick the right keys. Native i18n's approach (locale-specific API endpoints) is cleaner for headless.

### Adding a Locale Later

Every new locale means new database columns for every translated field on every registered model. This is a schema migration — fine for 2–3 locales, painful for 10+.

```bash
# Adding 'es' Spanish:
# 1. Add to LANGUAGES
# 2. python manage.py makemigrations  # Creates title_es, body_es, etc.
# 3. python manage.py migrate
```

### No Translation Workflow

Unlike `wagtail-localize`, modeltranslation has no concept of "translation in progress" or "out of date translation." Every field for every locale is always editable. There's no sync mechanism, no "needs translation" flag, and no PO file export/import. For sites where translations are done by editors in the admin, this is fine. For external translation workflows, it's a limitation.

## modeltranslation vs. native i18n — Decision Matrix

| Factor | modeltranslation | Native i18n |
|---|---|---|
| Setup complexity | Low (just columns) | Medium (tree structure, sync) |
| Editor UX (2–4 locales) | Good (tabbed fields) | Good (separate pages) |
| Editor UX (10+ locales) | Bad (too many tabs) | Good (pages per locale) |
| Adding locales post-launch | Migration pain | Just `sync` + create pages |
| StreamField translation | Very limited | Full support |
| Headless/API | Noisy responses | Clean locale-specific API |
| External translation (PO) | Not supported | Built-in |
| Machine translation | Manual | DeepL/Smartling plugins |
| CRX compatibility | ✅ Default | ❌ Not supported |
| DB size | One wide row | N rows (one per locale) |
