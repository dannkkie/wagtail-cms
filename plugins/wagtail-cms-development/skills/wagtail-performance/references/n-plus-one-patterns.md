# N+1 Query Patterns in Wagtail — Exhaustive Catalog

## Template-level N+1

### Pattern 1: Page FK fields in listing loops

```django
<!-- N queries for header_image -->
{% for page in pages %}
    {% image page.header_image fill-800x600 %}
{% endfor %}
```

**Fix:** `select_related('header_image')` on the queryset.

### Pattern 2: Parent page traversal

```django
<!-- Page.get_parent() is treebeard-cached, fine -->
{{ page.get_parent.title }}

<!-- But page.get_parent().specific is NOT cached — N queries -->
{{ page.get_parent.specific.custom_field }}
```

**Fix:** `select_related('parent')` + handle `.specific` in the view.

### Pattern 3: Site settings in templates

```django
<!-- Every include that uses SiteSettings hits the DB -->
{% load wagtailsettings_tags %}
{% settings %}  <!-- 1 query per page, per settings model -->
```

**Fix:** Cache settings in the view context or use template fragment caching.

### Pattern 4: Snippet FK in template

```django
{{ page.author.bio }}  <!-- N queries for author snippets -->
```

**Fix:** `select_related('author')` when author is an FK to a snippet.

## StreamField N+1

### Pattern 5: Image chooser blocks

```django
{% for block in page.body %}
    {% if block.block_type == 'image' %}
        {% image block.value fill-800x600 %}  <!-- 1 query per image block -->
    {% endif %}
{% endfor %}
```

**Fix:** pre-extract image IDs from StreamField, bulk-fetch, pass in context.

### Pattern 6: Page chooser blocks

```django
{% for block in page.body %}
    {% if block.block_type == 'page_link' %}
        <a href="{{ block.value.url }}">{{ block.value.title }}</a>  <!-- 1 query per page chooser -->
    {% endif %}
{% endfor %}
```

**Fix:** pre-extract page IDs, `Page.objects.filter(id__in=ids).specific()`, pass in context.

### Pattern 7: Document chooser blocks

Same pattern as images — pre-extract document IDs, bulk-fetch, pass in context.

### Pattern 8: Snippet chooser blocks

Same pattern — pre-extract, bulk-fetch, pass in context. Snippet models vary, so check `block.value._meta.model` to group by model type.

## Queryset-Level N+1

### Pattern 9: `.specific()` in a loop

```python
for page in pages:
    print(page.specific.custom_field)  # N queries
```

**Fix:** `pages.specific()` — the queryset method does a single efficient query.

### Pattern 10: `.url` property in a loop

Every `.url` call traverses `get_parent()` chain. For listing pages, use `page.get_url(request)` once and cache it, or batch-compute URLs.

### Pattern 11: Tags / M2M

```django
{% for tag in page.tags.all %}  <!-- N queries -->
```

**Fix:** `prefetch_related('tagged_items__tag')`.

### Pattern 12: Related pages via ParentalKey

```python
for related in page.related_posts.all():  # N queries
```

**Fix:** `prefetch_related('related_posts')`.

## API-Level N+1

### Pattern 13: API fields resolving FKs

```python
api_fields = [APIField('author')]  # Default FK serialization queries the related model
```

**Fix:** Override `get_queryset()` in the API viewset to include `select_related('author')`.

### Pattern 14: Image rendition fields

```python
APIField('hero', serializer=ImageRenditionField('fill-1200x600', source='header_image'))
```

Each rendition field queries `ImageRendition` to find or create the rendition. For listings with many images, this can be 2N+1 (one for image, one for rendition per image).

**Fix:** Prefetch image renditions for common specs.

### Pattern 15: Nested serializers

```python
APIField('related_pages')  # Fetches each related page with its own N+1
```

**Fix:** Control serialization depth; avoid nested serializers in listing endpoints.

## Tools for Finding N+1

1. **django-debug-toolbar** — shows all queries on every request
2. **django-querycount** — prints query counts to console in dev
3. **django-perf-rec** — records and compares query counts between code changes
4. **`django.test.utils.CaptureQueriesContext`** — for unit-test query assertions
5. **Postgres `log_min_duration_statement`** — logs slow queries in production
