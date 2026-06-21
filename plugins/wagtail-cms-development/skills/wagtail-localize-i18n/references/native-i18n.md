# Wagtail Native i18n — Deep Reference

## Architecture

Wagtail's native i18n is built on three layers:

1. **Django's i18n framework** — `LocaleMiddleware`, `i18n_patterns`, `LANGUAGES` setting, `gettext`/`gettext_lazy`
2. **`wagtail.locales.Locale` model** — a database table of enabled locales, synced from `WAGTAIL_CONTENT_LANGUAGES`
3. **`wagtail-localize`** — the translation management layer: `TranslatableMixin`, translation sync, PO file import/export, machine translation integrations

### The locale tree

Each `Locale` gets its own root page under the site's root. Pages are organized in parallel trees:

```
Root (/) 
├── en/                    ← Locale: English
│   ├── about/
│   └── blog/
│       ├── post-1/
│       └── post-2/
└── fr/                    ← Locale: French
    ├── about/             ← translation of /en/about/
    └── blog/
        └── post-1/        ← translation of /en/blog/post-1/
```

`post-2` doesn't exist under `fr/` — it hasn't been translated yet. Or an alias of `/en/blog/post-2/` can be placed under `/fr/blog/` to show English content where French isn't available.

### `TranslatableMixin` and `translation_key`

Every translatable model gets:
- `translation_key` — a UUID shared by all translations of the same piece of content
- `locale` — FK to `Locale`
- `alias_of` — FK to the source page (null for originals and translations; set only for aliases)

```python
class MyPage(Page, TranslatableMixin):  # Page already inherits this
    translatable_fields = ['title', 'body', 'intro']
    # 'author' and 'created_at' are NOT in translatable_fields
    # They're shared across all translations
```

Key methods:
- `page.get_translations()` — returns queryset of all translations (same `translation_key`, different `locale`)
- `page.get_translation(locale)` or `page.get_translation_or_none(locale)` — get a specific locale's version
- `page.has_translation(locale)` — boolean check
- `page.copy_for_translation(locale, copy_parents=True)` — create a translation under the target locale's tree
- `page.localized` — property; returns the page in the current request's locale

### `copy_for_translation()` internals

This is the core operation. It:
1. Creates a new page under the target locale's tree
2. Copies translatable field values (not non-translatable ones)
3. Shares the `translation_key`
4. Sets `alias_of=None` (this is a real translation, not an alias)
5. Optionally copies child pages recursively

Post-copy, the new page is a draft — editors review and publish it.

## Configuration

### `WAGTAIL_CONTENT_LANGUAGES`

```python
WAGTAIL_CONTENT_LANGUAGES = LANGUAGES = [
    ('en', 'English'),
    ('fr', 'French'),
    ('de', 'German'),
    ('es', 'Spanish'),
]
```

Run `python manage.py sync_page_translation_fields` after adding locales. This creates `Locale` rows and adds `translation_key` to `Page` tables.

### `WAGTAIL_I18N_ENABLED = True`

Enables the "Translate" button in the Wagtail admin, locale selector in the page explorer, and locale-aware URL generation.

### URL configuration

```python
from django.conf.urls.i18n import i18n_patterns

urlpatterns = [
    # Non-translatable paths (admin, API, etc.)
    path('django-admin/', admin.site.urls),
    path('admin/', include(wagtailadmin_urls)),
    path('api/v2/', api_router.urls),
]

urlpatterns += i18n_patterns(
    # These get locale prefix: /en/..., /fr/..., etc.
    path('search/', search_views.search, name='search'),
    path('', include(wagtail_urls)),  # Wagtail serve — always last
)
```

### Settings override per site

```python
# In a Site's settings or a SiteSettings model:
class LanguageSettings(BaseSiteSetting):
    auto_redirect = models.BooleanField(
        default=True,
        help_text="Redirect users to their browser's preferred locale"
    )
    default_language = models.ForeignKey(
        'wagtail_locales.Locale', 
        on_delete=models.SET_NULL, 
        null=True
    )
```

## PO File Workflows

### Export

```bash
python manage.py sync_page_translation_fields
python manage.py export_translations --locale=fr --output=fr.po
```

Exports all translatable strings for pages that exist in the source locale but not the target. The PO file includes metadata about which page/field each string came from.

### Import

```bash
python manage.py import_translations --locale=fr fr.po
```

Creates translated pages as drafts. Review and publish in the admin. The importer matches strings back to their source pages via PO metadata.

### Automation in CI

```bash
# In CI, after a content change on the English site:
python manage.py export_translations --locale=fr --output=/tmp/fr.po
# Send fr.po to translation service
# Import returned translations
python manage.py import_translations --locale=fr /tmp/translated-fr.po
```

## Machine Translation Integration

### DeepL

```bash
pip install wagtail-localize-deepl
```

```python
# settings.py
WAGTAILLOCALIZE_DEEPL_API_KEY = os.environ['DEEPL_API_KEY']
```

This adds a "Machine translate" button in the admin that auto-fills translations using DeepL. It's a time-saver for first drafts but translations should always be reviewed by a human.

### Smartling

```bash
pip install wagtail-localize-smartling
```

For enterprise translation workflows — Smartling is a translation management platform. This integration creates Smartling jobs directly from Wagtail, tracks translation status, and imports completed translations.

## REST API Under i18n

When i18n is enabled, Wagtail's REST API v2 becomes locale-aware:

```
GET /api/v2/pages/?locale=fr
GET /api/v2/pages/?translation_of=42  # All translations of page 42
```

Page API responses include:
```json
{
    "id": 42,
    "meta": {
        "locale": "fr",
        "translation_key": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
        "translations": {
            "en": {"id": 41, "title": "About Us"},
            "de": {"id": 43, "title": "Über uns"}
        }
    }
}
```

For headless/Next.js consumers, the pattern is:
1. Detect user's locale from URL (`/fr/about`) or `Accept-Language` header
2. Fetch page data from `GET /api/v2/pages/?locale=fr&slug=about`
3. Use `meta.translations` to build the language-switcher UI

## Performance

### The N+1 across locales problem

```python
# BAD: N+1 per locale
for page in BlogPage.objects.filter(locale='en'):
    translations = page.get_translations()  # N queries

# GOOD: prefetch locale tree
from django.db.models import Prefetch
pages = BlogPage.objects.filter(
    locale='en'
).prefetch_related(
    Prefetch(
        'translations',
        queryset=BlogPage.objects.select_related('locale')
    )
)
```

### `select_related` for locale references

Every page has a `locale` FK. Always `select_related('locale')` when listing pages.

## Common Pitfalls

1. **Not running `sync_page_translation_fields` after adding a locale** — pages won't have `translation_key` values.
2. **Using `page.specific` on a translated page** — `specific` respects the current locale, which may not be what you want in admin code.
3. **Forgetting `i18n_patterns`** — Wagtail's `serve()` view works without it, but URL generation won't include the locale prefix.
4. **Mixing translatable and non-translatable on StreamField** — the whole StreamField is either translatable or not. You can't make individual blocks translatable within a StreamField.
5. **Parent page locale mismatch** — a translated page must live under a parent in the SAME locale tree. If the parent doesn't exist in the target locale, use `copy_parents=True`.
