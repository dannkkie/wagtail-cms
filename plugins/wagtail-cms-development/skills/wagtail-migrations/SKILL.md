---
name: wagtail-migrations
description: 'Manage Wagtail/Django migrations safely in production. Use this any time the user mentions Wagtail migrations, schema changes, data migrations for StreamField, squashing migrations, zero-downtime deploys, migration conflicts, or migrating page tree structures. Covers StreamField data migration patterns, zero-downtime strategies, squashing, and testing migrations in CI.'
---

# Wagtail Migrations

## Orientation

Wagtail migrations are Django migrations with two extra dimensions: the page tree (treebeard MP_Node columns) and StreamField JSON. The page tree makes destructive operations riskier (wrong migration = broken site tree), and StreamField migrations need custom data handling because the structure is opaque JSON to the database.

## Core discipline

1. **Read every generated migration before committing.** `makemigrations` output is a starting point, not a deliverable.
2. **Back up the database before running migrations in production.**
3. **Forward-only migrations are the norm.** Rollback migrations exist on paper but are rarely tested — don't count on them.
4. **Data migrations are separate from schema migrations.** One migration changes the schema; a separate migration (with `RunPython`) changes the data.

## StreamField Data Migrations

StreamField content is stored as a JSON list of `{"type": "...", "value": ...}` objects. When you rename or restructure a block, you must rewrite all existing page data.

### Renaming a block

```python
# migrations/0002_rename_old_block.py
from django.db import migrations

def rename_block(apps, schema_editor):
    BlogPage = apps.get_model('blog', 'BlogPage')
    for page in BlogPage.objects.all():
        changed = False
        new_body = []
        for block in page.body:
            if block.block_type == 'old_name':
                block.block_type = 'new_name'
                changed = True
            new_body.append(block)
        if changed:
            page.body = new_body
            page.save(update_fields=['body'])

def reverse_rename(apps, schema_editor):
    BlogPage = apps.get_model('blog', 'BlogPage')
    for page in BlogPage.objects.all():
        changed = False
        new_body = []
        for block in page.body:
            if block.block_type == 'new_name':
                block.block_type = 'old_name'
                changed = True
            new_body.append(block)
        if changed:
            page.body = new_body
            page.save(update_fields=['body'])

class Migration(migrations.Migration):
    dependencies = [('blog', '0001_initial')]
    operations = [migrations.RunPython(rename_block, reverse_rename)]
```

### Changing a block's structure

```python
def restructure_hero_block(apps, schema_editor):
    BlogPage = apps.get_model('blog', 'BlogPage')
    for page in BlogPage.objects.all():
        changed = False
        new_body = []
        for block in page.body:
            if block.block_type == 'hero':
                value = dict(block.value)  # StructBlock values are dicts
                # Add a new field with a default
                if 'overlay_opacity' not in value:
                    value['overlay_opacity'] = 0.5
                # Rename a field
                if 'background' in value:
                    value['background_image'] = value.pop('background')
                block.value = value
                changed = True
            new_body.append(block)
        if changed:
            page.body = new_body
            page.save(update_fields=['body'])
```

### Deleting a block type

```python
def remove_deprecated_block(apps, schema_editor):
    BlogPage = apps.get_model('blog', 'BlogPage')
    for page in BlogPage.objects.all():
        original_len = len(page.body)
        page.body = [b for b in page.body if b.block_type != 'deprecated_block']
        if len(page.body) != original_len:
            page.save(update_fields=['body'])
```

### StreamField migration performance

For large sites (10K+ pages), iterating page by page is too slow. Batch the work:

```python
def batch_migrate_blocks(apps, schema_editor):
    BlogPage = apps.get_model('blog', 'BlogPage')
    batch_size = 500
    total = BlogPage.objects.count()
    
    for offset in range(0, total, batch_size):
        pages = BlogPage.objects.all()[offset:offset + batch_size]
        for page in pages.iterator():
            # ... migrate page.body ...
            pass
        BlogPage.objects.bulk_update(pages, ['body'])
```

## Page Tree Migrations

### Moving pages between parents

```python
def restructure_blog_tree(apps, schema_editor):
    ContentType = apps.get_model('contenttypes', 'ContentType')
    Page = apps.get_model('wagtailcore', 'Page')
    
    blog_index = Page.objects.get(slug='blog')
    new_parent = Page.objects.get(slug='articles')
    
    for child in blog_index.get_children():
        child.move(new_parent, pos='last-child')
```

### Converting a page type

Wagtail has no built-in "change page type." Migrations that change page types are tricky:

```python
def convert_to_new_page_type(apps, schema_editor):
    OldPage = apps.get_model('blog', 'OldBlogPage')
    NewPage = apps.get_model('blog', 'NewBlogPage')
    
    for old in OldPage.objects.all():
        # Create new page at the same position
        new = NewPage(
            title=old.title,
            slug=old.slug,
            path=old.path,  # Same tree position
            depth=old.depth,
            numchild=old.numchild,
            # Map fields
            intro=old.old_intro,
            body=old.old_body,
        )
        new.save()
        # Reassign children
        for child in old.get_children():
            child.move(new, pos='last-child')
```

**Warning:** Changing page types is a high-risk operation. Consider whether a migration is truly needed vs. keeping the old model with a deprecation path.

## Zero-Downtime Deployments

### Additive-only schema changes

Safe (run during normal operation):
- Add a new nullable column
- Add a new model
- Add a new field with a default value

Requires maintenance window:
- Remove a column
- Rename a column
- Change a column type
- Add a non-nullable column without a default

### Strategy: three-deploy pattern

For risky changes:

1. **Deploy 1**: Add the new column (nullable), start writing to both old and new
2. **Deploy 2**: Backfill existing data, switch reads to new column
3. **Deploy 3**: Remove old column

### Example: removing a StreamField block

1. **Deploy 1**: Add new block type, deprecate old one (but don't remove from `body`)
2. **Deploy 2**: Run data migration to convert old blocks → new blocks
3. **Deploy 3**: Remove old block type definition

## Squashing Migrations

```bash
python manage.py squashmigrations blog 0001 0015 --squashed-name=initial_v2
```

Then:
1. Read the squashed migration carefully
2. Remove `replaces = [...]` once you're confident
3. Delete the old migrations
4. Test on a copy of production data

Don't squash migrations casually — especially not on a live project with multiple contributors. Do it as a deliberate step before a major version bump.

## Testing Migrations in CI

```python
# tests/test_migrations.py
from django.test import TransactionTestCase
from django.core.management import call_command

class MigrationTests(TransactionTestCase):
    available_apps = ['blog']
    
    def test_migration_forwards(self):
        """All migrations apply cleanly."""
        call_command('migrate', 'blog', verbosity=0)
    
    def test_migration_backwards(self):
        """Rollback is clean."""
        call_command('migrate', 'blog', '0001', verbosity=0)
    
    def test_data_integrity_after_migration(self):
        """Data survives migration intact."""
        call_command('migrate', 'blog', verbosity=0)
        self.assertTrue(BlogPage.objects.filter(title='Test Post').exists())
```

## Common Pitfalls

1. **`apps.get_model()` in RunPython** — uses a historical model, not the current code. You can't call model methods, `save()`, or `full_clean()`.
2. **Forgetting `update_fields`** — `page.save()` in a migration triggers a full update including treebeard columns. Use `update_fields=['body']` when only changing StreamField data.
3. **Migration + publish signal** — `page.save()` in a migration triggers `page_published` signal, which can fire CDN purges, webhooks, and emails. Use `raw=True` or disable signals.
4. **StreamField data isn't validated in migrations** — invalid block data will persist silently, causing errors when the page is edited later. Validate after migration.
5. **Long-running migrations time out** — for 100K+ pages, use `SeparateDatabaseAndState` + a management command instead of RunPython.
