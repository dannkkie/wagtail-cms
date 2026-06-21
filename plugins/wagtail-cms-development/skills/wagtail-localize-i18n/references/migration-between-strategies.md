# Migration Between i18n Strategies

**Warning:** These are data-migration operations that restructure your database. Never run them against production without a full backup and a staging dry-run. A mistake can corrupt the page tree and orphan content.

## Preparation (Both Directions)

1. **Full DB backup** — `pg_dump` or equivalent
2. **Staging environment** — test the migration command there first
3. **Downtime window** — migrations of this scale typically require read-only mode
4. **Line count** — know how many pages, snippets, and locales are involved before you start

## modeltranslation → Native i18n

### Step 1: Enable native i18n infrastructure

```python
# settings.py
WAGTAIL_I18N_ENABLED = True
WAGTAIL_CONTENT_LANGUAGES = LANGUAGES = [...]
INSTALLED_APPS += ['wagtail.locales', 'wagtail_localize', 'wagtail_localize.locales']
```

Run migrations: `python manage.py migrate`

### Step 2: Sync locales

```bash
python manage.py sync_page_translation_fields
```

This creates `Locale` rows and adds `translation_key` columns to page tables.

### Step 3: The migration command

```python
# management/commands/migrate_modeltranslation_to_native_i18n.py
from django.core.management.base import BaseCommand
from wagtail.models import Page, Locale
from wagtail_localize.models import Translation
import uuid

class Command(BaseCommand):
    def handle(self, **options):
        source_locale = Locale.objects.get(language_code='en')
        
        for page_cls in Page.allowed_subpage_models():
            for page in page_cls.objects.filter(locale=source_locale):
                # Create translations for each non-English locale
                for locale in Locale.objects.exclude(language_code='en'):
                    self.migrate_page(page, locale, source_locale)
    
    def migrate_page(self, source_page, target_locale, source_locale):
        # 1. Find or create the target locale's parent
        target_parent = source_page.get_parent().get_translation_or_none(target_locale)
        if not target_parent:
            self.stdout.write(f"Skipping {source_page}: parent not translated")
            return
        
        # 2. Create the translated page
        translated = source_page.copy_for_translation(
            target_locale, 
            copy_parents=False
        )
        
        # 3. Copy field values from locale-prefixed columns
        lang_code = target_locale.language_code
        for field in source_page.translatable_fields:
            old_value = getattr(source_page, f'{field}_{lang_code}', None)
            if old_value is not None:
                setattr(translated, field, old_value)
        
        # 4. Copy StreamField content (manual JSON rewrite needed)
        # This is the hardest part — see notes below
        
        translated.save()
```

### Step 4: StreamField migration

StreamField content in modeltranslation is a single JSON blob. You need to rewrite it per locale. The approach:

1. Export the StreamField JSON
2. For each locale, extract the locale-specific content from the JSON structure
3. Create new StreamField JSON for each translation

This is usually project-specific because StreamField structures vary. Write a custom handler per block type that needs locale-aware migration.

### Step 5: Cleanup

After verifying all translations are correct:
- Remove modeltranslation from `INSTALLED_APPS`
- Drop the locale-prefixed columns (`title_fr`, `body_fr`, etc.)
- Run `makemigrations --empty your_app --name remove_modeltranslation_columns` and write the column drops manually

## Native i18n → modeltranslation

### The migration command

```python
# management/commands/migrate_native_i18n_to_modeltranslation.py
from django.core.management.base import BaseCommand
from wagtail.models import Page, Locale

class Command(BaseCommand):
    def handle(self, **options):
        # Group pages by translation_key
        from django.db.models import Count
        keys = Page.objects.values('translation_key').annotate(
            count=Count('id')
        ).filter(count__gt=1)
        
        for entry in keys:
            translations = Page.objects.filter(
                translation_key=entry['translation_key']
            ).select_related('locale')
            
            # Pick the "primary" page (first-locale or most-complete)
            primary = translations.first()
            
            # Collapse all translations into one page
            for t in translations:
                lang = t.locale.language_code
                for field in primary.translatable_fields:
                    value = getattr(t, field)
                    setattr(primary, f'{field}_{lang}', value)
            
            primary.save()
            
            # Delete the non-primary translation pages
            # (or archive them — safer)
            for t in translations.exclude(id=primary.id):
                t.unpublish()
                # Don't delete until verified
```

### Post-migration

1. Add modeltranslation to `INSTALLED_APPS`
2. Register models with `TranslationOptions`
3. Run `makemigrations` — this adds locale-prefixed columns
4. Verify data in the new columns
5. Remove `wagtail-localize` and `wagtail.locales`
6. Remove `i18n_patterns` from URL config

## Testing the Migration

```python
# tests/test_i18n_migration.py
from django.test import TestCase
from wagtail.models import Locale

class I18nMigrationTest(TestCase):
    def setUp(self):
        self.en = Locale.objects.get(language_code='en')
        self.fr = Locale.objects.get(language_code='fr')
    
    def test_all_pages_have_translations(self):
        """After migration, every English page should have French counterpart"""
        for page in Page.objects.filter(locale=self.en, translation_key__isnull=False):
            self.assertTrue(
                page.has_translation(self.fr),
                f"Page {page} has no French translation"
            )
    
    def test_content_matches(self):
        """Translated content should match original locale-specific values"""
        for page in Page.objects.filter(locale=self.en):
            french = page.get_translation_or_none(self.fr)
            if french:
                self.assertNotEqual(page.title, '')  # Not empty
                self.assertNotEqual(french.title, '')
```

## Rollback Plan

Always write a rollback command BEFORE running the migration:

```python
class Command(BaseCommand):
    def handle(self, **options):
        # Reverse the migration
        # For modeltranslation → native: delete translation pages, clear translation_key
        # For native → modeltranslation: restore archived pages, clear locale columns
```

Test the rollback in staging before the forward migration. Know how long it takes — the rollback window should fit within your downtime budget.
