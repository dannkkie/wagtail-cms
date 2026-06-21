---
name: wagtail-localize-i18n
description: 'Set up and maintain multi-language Wagtail sites. Use this any time the user mentions multi-language content, i18n, wagtail-localize, wagtail-modeltranslation, translating pages, locale-aware URLs, language switching, PO files, or syncing translated content. Covers both vanilla Wagtail i18n (wagtail-localize + django-treebeard locale tree) and the CRX approach (wagtail-modeltranslation), plus migration between strategies.'
---

# Wagtail Multi-Language & Internationalization (i18n)

## Orientation

Wagtail offers two fundamentally different approaches to multi-language, and they are **not interoperable**. Choosing one locks you in — switching later is a data-migration project, not a config change.

- **Wagtail's native i18n** (powered by `wagtail-localize`): each locale gets its own page tree. A `/fr/` French version of a page is a distinct page instance, linked by a `translation_key` (UUID shared across all translations). Aliases can mirror untranslated pages so they appear under every locale node without duplicating content.
- **`wagtail-modeltranslation`**: translates field values in-place on the SAME model row. One `Page` row has `title_en`, `title_fr`, `title_de` columns. This is the CRX-recommended approach and is conceptually simpler — no separate page trees, no sync — but it ties the content structure to the database schema and makes it harder to add locales later without migrations.

CRX recommends `wagtail-modeltranslation` because CRX ships Bootstrap 5 templates with server-side rendering and doesn't benefit from the headless flexibility that Wagtail's native i18n enables. Vanilla Wagtail projects should use the native i18n path unless there's a strong reason not to.

## Step 0: ask first — it's a one-way door

Before scaffolding any i18n setup, confirm with the user:
1. How many locales? (2–3 vs. 10+ dictates different approaches)
2. Translated by whom? (in-Wagtail editors vs. external translators → different workflows)
3. Is this a new project or an existing one? (retro-fitting i18n onto an existing site is a migration-heavy task)
4. Headless or server-rendered? (native i18n works better for headless)
5. CRX or vanilla Wagtail? (CRX strongly favors `wagtail-modeltranslation`)

## Core conventions (both approaches)

- **URL structure is a product decision, not a technical one.** `example.com/fr/`, `fr.example.com/`, `example.com?lang=fr` — Wagtail supports domain/prefix/query-string strategies via `LocaleMiddleware` + `i18n_patterns`. Prefix (`/fr/`) is the most common and best-supported by search engines.
- **The `Locale` model is central to native i18n.** Every translatable page is assigned a `locale` FK. The `Locale` model is synced from `WAGTAIL_CONTENT_LANGUAGES` via the `sync_page_translation_fields` management command — don't create `Locale` rows manually.
- **Translatable vs. non-translatable fields:** in native i18n, fields can be marked `translatable` (syncs across locales) or non-translatable (shared across translations). In modeltranslation, every translated field is duplicated per locale.
- **Snippets are translatable too.** Register with `@register_snippet` and add `TranslatableMixin` (native i18n) or `TranslationOptions` (modeltranslation). Snippets referenced by pages need careful handling — a French page linking to an English snippet is a common footgun.

## Native i18n: the `wagtail-localize` path

### Setup

```bash
pip install wagtail-localize
```

In `settings.py`:
```python
USE_I18N = True
WAGTAIL_I18N_ENABLED = True
WAGTAIL_CONTENT_LANGUAGES = LANGUAGES = [
    ('en', 'English'),
    ('fr', 'French'),
    ('de', 'German'),
]

MIDDLEWARE = [
    'django.middleware.locale.LocaleMiddleware',  # after SessionMiddleware, before CommonMiddleware
    # ...
]

INSTALLED_APPS = [
    # ...
    'wagtail.locales',
    'wagtail_localize',
    'wagtail_localize.locales',  # optional, for locale management UI
]
```

URL config:
```python
from django.conf.urls.i18n import i18n_patterns

urlpatterns += i18n_patterns(
    path('', include('mysite.urls')),
    # Wagtail's serve view is automatically i18n-aware
)
```

Run `python manage.py makemigrations && python manage.py migrate` then sync locales:
```bash
python manage.py sync_page_translation_fields
```

### Page model setup

Pages become translatable just by existing under a `Locale`-enabled site. The `translation_key` UUID is added automatically by a migration. You don't need to add anything to `Page` subclasses, but you SHOULD mark which fields are translatable:

```python
from wagtail.models import Page, TranslatableMixin

class BlogPage(Page):
    body = RichTextField()
    author = models.CharField(max_length=255)
    
    # These are the fields that vary per locale:
    translatable_fields = ['body', 'title']
    # 'author' is shared across translations
```

### Translation workflows

**In-editor translation:** Wagtail's "Translate" button on each page creates a new page under the target locale's tree, copies content, and links them via `translation_key`. Editors translate in the Wagtail admin.

**PO file / external translation:** Wagtail Localize supports `.po` file export/import:
```bash
python manage.py sync_page_translation_fields
python manage.py export_translations --locale=fr --output=fr.po
# Translator works on fr.po
python manage.py import_translations --locale=fr fr.po
```

**Machine translation:** `wagtail-localize` has optional integrations with DeepL, Google Translate, and Azure Translator via plugins (`wagtail-localize-smartling`, `wagtail-localize-deepl`).

### Aliases: the "same content, different locale" pattern

When you have a page that shouldn't be translated (e.g., a terms-of-service page that's English-only but should appear under `/fr/terms/`), create an **alias**:

```python
from wagtail.models import Page, Locale

english_page = TermsPage.objects.get(locale__language_code='en')
french_locale = Locale.objects.get(language_code='fr')
alias = english_page.copy_for_translation(french_locale, copy_parents=True, alias=True)
```

Aliases share the same `alias_of` FK — they render the origin page's content. No duplication, no sync needed.

### Synchronization

When the English source page changes, translations need updating. `wagtail-localize` tracks this with a `last_translated_at` timestamp and shows "Translation is out of date" warnings in the admin. To sync:

```python
# Programmatically check which pages need retranslation
from wagtail_localize.models import Translation

stale = Translation.objects.filter(
    source__locale=source_locale,
    target_locale=target_locale,
    source__last_published_at__gt=models.F('created_at'),
)
```

## The `wagtail-modeltranslation` path (CRX default)

### Setup

```bash
pip install wagtail-modeltranslation
```

In `settings.py`:
```python
INSTALLED_APPS = [
    'modeltranslation',
    # ...
]

LANGUAGES = [
    ('en', 'English'),
    ('fr', 'French'),
]

MODELTRANSLATION_DEFAULT_LANGUAGE = 'en'
```

For CRX specifically, the `LANGUAGES` setting plus `CRX_ENABLE_MODELTRANSLATION = True` is usually enough — CRX's abstract page models are already registered for translation.

### Page model setup

Instead of separate page trees, you add `TranslationOptions`:

```python
from modeltranslation.translator import translator, TranslationOptions

class BlogPageTranslationOptions(TranslationOptions):
    fields = ('title', 'body', 'intro')

translator.register(BlogPage, BlogPageTranslationOptions)
```

This creates `title_en`, `title_fr`, `body_en`, `body_fr`, etc. in the database automatically (run `makemigrations` + `migrate`).

### Key differences from native i18n

| Concern | Native i18n (`wagtail-localize`) | `wagtail-modeltranslation` |
|---|---|---|
| URL structure | `/fr/about/` — separate page tree per locale | Same URL, content switches by Django's `LANGUAGE_CODE` |
| DB schema | One row per translation (shared `translation_key`) | One row, multiple columns (one per locale per field) |
| Adding a locale | Add to `WAGTAIL_CONTENT_LANGUAGES`, sync | Add to `LANGUAGES`, add columns via migration |
| Headless friendly | Yes — REST API exposes locale tree naturally | No — API returns all locale data in one response |
| CRX integration | Not supported (CRX templates aren't i18n-pattern aware) | First-class, pre-configured |

## Migrating between strategies

This is a data-migration problem, not a config swap. The skill doesn't recommend switching unless you have a specific, articulated reason.

**modeltranslation → native i18n:** Extract locale-specific column values, create new `Page` instances per locale, link them with `translation_key`, update parent/child relationships in the page tree, delete the old multi-column page.

**Native i18n → modeltranslation:** Collapse translation pages into one page per `translation_key`, migrate field values into locale-prefixed columns, restructure the page tree.

Both are custom management commands with real risk — never attempt without a full DB backup and a staging dry-run.

## Testing i18n

```python
from django.test import TestCase, override_settings
from wagtail.models import Locale, Page
from wagtail_localize.test.models import create_test_locales

class MultiLanguageTest(TestCase):
    def setUp(self):
        self.en_locale = Locale.objects.get(language_code='en')
        self.fr_locale = Locale.objects.get(language_code='fr')
    
    def test_page_in_multiple_locales(self):
        page = MyPage.objects.get(locale=self.en_locale)
        self.assertTrue(page.has_translation(self.fr_locale))
```

For modeltranslation, use `override_settings(LANGUAGE_CODE='fr')` and verify the correct field values surface.

## Reference files

- **`references/native-i18n.md`** — deep dive: `TranslatableMixin`, `translation_key`, locale tree structure, `copy_for_translation()`, alias management, `wagtail-localize` configuration, PO file workflows, sync strategies, REST API under i18n, performance (N+1 across locale trees).
- **`references/modeltranslation.md`** — deep dive: `TranslationOptions` API, `wagtail-modeltranslation` config, CRX integration points, field-level translate/fallback settings, admin UI differences, template tag usage (`{% get_current_language %}`, `{% language %}`), limitations (no StreamField translation without `wagtail-modeltranslation` patches).
- **`references/migration-between-strategies.md`** — step-by-step migration command patterns for both directions, rollback procedures, testing migrations in CI.
