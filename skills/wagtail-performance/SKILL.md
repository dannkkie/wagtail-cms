---
name: wagtail-performance
description: 'Optimize Wagtail site performance for production. Use this any time the user mentions Wagtail slow pages, N+1 queries, StreamField performance, image rendition caching, database query optimization, Redis caching, CDN strategy, page load speed, profiling queries, or scaling Wagtail. Covers the classic Wagtail N+1 patterns, template fragment caching, wagtail-cache with Redis, image CDN, select_related/prefetch_related discipline, search backend choice, and query-count regression testing.'
---

# Wagtail Performance

## Orientation

Wagtail is fast out of the box for small sites, but it has sharp edges that compound at scale. The biggest issues are:

1. **N+1 queries** — the single most common Wagtail performance problem, especially in template loops over StreamField blocks
2. **Image rendition bloat** — Wagtail generates image renditions eagerly; many unique filter specs = many rows + many files
3. **Uncached StreamField block lookups** — chooser blocks (images, pages, documents) each trigger a query
4. **No query budget discipline** — pages that work fine in dev with 5 items blow up in production with 500

This skill assumes the site already works; the work is making it fast under real traffic and real content volume.

## The N+1 Problem — Wagtail's Signature Footgun

### In templates

```django
<!-- DON'T: N+1 — each page.image lookup hits the DB -->
{% for page in pages %}
    {% image page.header_image fill-800x600 %}
{% endfor %}
```

```django
<!-- DO: prefetch header_image on the queryset -->
{% for page in pages %}
    {% image page.header_image fill-800x600 %}  <!-- cached in memory -->
{% endfor %}
```

### In StreamField blocks — the hardest case

```django
<!-- DON'T: every chooser block in every page's body does a separate query -->
{% for block in page.body %}
    {% if block.block_type == 'image' %}
        {% image block.value fill-800x600 %}  <!-- N queries per page -->
    {% endif %}
{% endfor %}
```

The fix involves prefetching chooser block references BEFORE template rendering:

```python
from wagtail.models import Page
from django.db.models import Prefetch
from wagtail.images.models import Image

class BlogPage(Page):
    def get_context(self, request):
        context = super().get_context(request)
        # Prefetch images referenced in StreamField blocks
        image_ids = self._extract_chooser_ids('body', 'image')
        context['body_images'] = Image.objects.filter(id__in=image_ids)
        return context
    
    def _extract_chooser_ids(self, streamfield_name, block_type):
        """Extract all chooser block IDs from a StreamField."""
        ids = set()
        for block in getattr(self, streamfield_name, []):
            if block.block_type == block_type and block.value:
                if hasattr(block.value, 'id'):
                    ids.add(block.value.id)
        return ids
```

Or use the `wagtail-cache` approach (simpler but less granular):
```python
# Cache the entire rendered StreamField
from wagtail.contrib.frontend_cache.utils import purge_page_from_cache
```

### In querysets — the `select_related` / `prefetch_related` checklist

Every Wagtail page queryset should be scrutinized for N+1. Before returning a queryset to a template, check:

| Relationship | Method |
|---|---|
| Parent page | Already cached by treebeard |
| `locale` FK | `select_related('locale')` |
| `owner` FK | `select_related('owner')` |
| FK fields (`header_image`, `author`) | `select_related('header_image', 'author')` |
| M2M fields (`tags`, `categories`) | `prefetch_related('tagged_items__tag')` |
| Related pages (parental keys) | `prefetch_related('related_posts')` |
| Snippet references | `select_related('snippet_field')` |

```python
# The "production-ready" blog queryset
posts = BlogPage.objects.live().public().select_related(
    'locale',
    'owner',
    'header_image',
    'author',
).prefetch_related(
    'tagged_items__tag',
    'related_posts',
    Prefetch(
        'categories',
        queryset=BlogCategory.objects.select_related('icon')
    ),
).defer_streamfields()  # Skip StreamField JSON if you only need titles
```

## `wagtail-cache` + Redis

### Setup

```bash
pip install wagtail-cache django-redis
```

```python
# settings.py
CACHES = {
    'default': {
        'BACKEND': 'django_redis.cache.RedisCache',
        'LOCATION': 'redis://127.0.0.1:6379/1',
        'OPTIONS': {
            'CLIENT_CLASS': 'django_redis.client.DefaultClient',
        }
    }
}
WAGTAIL_CACHE = True
```

### Template fragment caching

```django
{% load wagtail_cache %}

<!-- Cache the entire header for 1 hour -->
{% wagtail_cache 'header' 3600 %}
    {% include 'includes/header.html' %}
{% endwagtail_cache %}

<!-- Cache a specific component, varied by page ID -->
{% wagtail_cache 'sidebar' 1800 page.id %}
    {% include 'includes/sidebar.html' %}
{% endwagtail_cache %}
```

Cache keys should be: descriptive, include page IDs/variants, and have reasonable TTLs. Don't cache user-specific content (login state, user-specific nav) without including the user ID in the key.

### Page-level caching

```python
from wagtail.contrib.frontend_cache.utils import purge_page_from_cache

# Purge CDN cache when a page is published
from wagtail.signals import page_published

@receiver(page_published)
def purge_cache_on_publish(sender, instance, **kwargs):
    purge_page_from_cache(instance)
```

## Image Performance

### Rendition strategy

Every unique `fill-{W}x{H}` filter spec creates a new `ImageRendition` row + file. Limit the spec surface area:

```python
# settings.py
WAGTAILIMAGES_ALLOWED_FILTERS = [
    'fill-400x300',
    'fill-800x600',
    'fill-1200x600',
    'fill-1920x1080',
    'width-800',
    'original',
]
```

### CDN for renditions

Serving renditions from your Django server doesn't scale. Route them through a CDN:

```
# nginx or CDN config — cache /media/images/*.jpg indefinitely
# Wagtail generates rendition filenames that encode the filter spec
# e.g., photo.fill-800x600.jpg — same spec = same filename = safe to cache forever
```

Or use an image service (Imgix, Cloudinary) that generates renditions on the fly and caches them at the edge:

```python
# Custom image model that delegates to Imgix
class ImgixImage(AbstractImage):
    def get_rendition(self, filter_spec):
        # Construct Imgix URL instead of generating local file
        return ImgixRendition(self, filter_spec)
```

### `defer_streamfields()` — skip StreamField when you don't need it

```python
# Listing page — only need title, date, thumbnail
posts = BlogPage.objects.live().defer_streamfields().select_related('header_image')
```

StreamField data is stored as JSON in a column. `defer_streamfields()` tells Django to skip that column — huge savings on listing pages where you only show titles and thumbnails.

## Database Query Optimization

### django-debug-toolbar & profiling

```bash
pip install django-debug-toolbar
```

Every page in dev should show a query count panel. Set a budget:
- Listing pages: <10 queries
- Detail pages: <20 queries
- API endpoints: <5 queries (they return fast by nature)

### Query-count regression testing

```python
# tests/test_performance.py
from django.test import TestCase
from django.test.utils import override_settings
import pytest

class PageQueryBudgetTest(TestCase):
    fixtures = ['test_data.json']

    def test_blog_listing_under_10_queries(self):
        with self.assertNumQueries(10):
            response = self.client.get('/blog/')
            self.assertEqual(response.status_code, 200)
    
    def test_blog_detail_under_20_queries(self):
        post = BlogPage.objects.first()
        with self.assertNumQueries(20):
            response = self.client.get(post.url)
            self.assertEqual(response.status_code, 200)
```

## Search Backend Choice

| Backend | Scale | Latency | Setup | Best For |
|---|---|---|---|---|
| Database FTS (Postgres `search`) | <100K pages | 50–200ms | Zero config | Simple sites |
| Postgres trigram | <100K pages | 100–500ms | `pg_trgm` extension | Fuzzy search, autocomplete |
| Elasticsearch | 1M+ pages | <50ms | Separate service | Real search features |
| Wagtail DB fallback | Dev only | — | Built-in | Local development |

The cutover from DB search to Elasticsearch should happen when:
- Page count exceeds 10K, OR
- Search latency consistently exceeds 200ms in production, OR
- You need faceted search, aggregations, or advanced relevance tuning

```python
# settings.py
WAGTAILSEARCH_BACKENDS = {
    'default': {
        'BACKEND': 'wagtail.search.backends.elasticsearch8',
        'URLS': ['https://search-cluster.example.com:9200'],
        'INDEX': 'wagtail',
        'TIMEOUT': 5,
    }
}
```

## CDN & Frontend Cache

### Cache-Control headers

```python
# settings.py
WAGTAILFRONTENDCACHE = {
    'cloudflare': {
        'BACKEND': 'wagtail.contrib.frontend_cache.backends.CloudflareBackend',
        'EMAIL': os.environ['CLOUDFLARE_EMAIL'],
        'TOKEN': os.environ['CLOUDFLARE_TOKEN'],
        'ZONEID': os.environ['CLOUDFLARE_ZONEID'],
    }
}
```

### Purge on publish

Wagtail's `page_published` and `page_unpublished` signals automatically purge affected URLs when `WAGTAILFRONTENDCACHE` is configured. No custom signal handling needed for basic purge-on-publish.

## Reference files

- **`references/n-plus-one-patterns.md`** — exhaustive catalog of N+1 patterns in Wagtail templates, StreamField blocks, querysets, and API responses, with fixes for each.
- **`references/caching-strategies.md`** — Redis setup, `wagtail-cache` template tags, view-level caching, queryset caching, CDN invalidation patterns, and cache key design.
