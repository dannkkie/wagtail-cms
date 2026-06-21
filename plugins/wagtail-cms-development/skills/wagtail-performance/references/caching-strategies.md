# Caching Strategies for Wagtail — Reference

## Cache Backend Setup

### Redis (recommended for production)

```bash
pip install django-redis
```

```python
# settings.py
CACHES = {
    'default': {
        'BACKEND': 'django_redis.cache.RedisCache',
        'LOCATION': 'redis://127.0.0.1:6379/1',
        'OPTIONS': {'CLIENT_CLASS': 'django_redis.client.DefaultClient'},
        'TIMEOUT': 300,
    },
    'renditions': {
        'BACKEND': 'django_redis.cache.RedisCache',
        'LOCATION': 'redis://127.0.0.1:6379/2',
        'TIMEOUT': None,  # Never expire rendition references
    },
    'search': {
        'BACKEND': 'django_redis.cache.RedisCache',
        'LOCATION': 'redis://127.0.0.1:6379/3',
    }
}
```

Separate Redis databases for different cache types — renditions don't evict template caches, search doesn't evict renditions.

### Memcached (alternative)

```python
CACHES = {
    'default': {
        'BACKEND': 'django.core.cache.backends.memcached.PyMemcacheCache',
        'LOCATION': '127.0.0.1:11211',
    }
}
```

Good for simple setups; Redis preferred for Wagtail projects due to `wagtail-cache` features.

## Template Fragment Caching

### `wagtail-cache` tag

```django
{% load wagtail_cache %}

<!-- Time-based cache -->
{% wagtail_cache 'nav-main' 3600 %}
    <nav>{% include 'includes/main-nav.html' %}</nav>
{% endwagtail_cache %}

<!-- Time + variant cache -->
{% wagtail_cache 'sidebar' 1800 page.id request.is_preview %}
    {% include 'includes/sidebar.html' %}
{% endwagtail_cache %}

<!-- Invalidate on publish (automatic if settings configured) -->
{% wagtail_cache 'page-footer' 86400 page.last_published_at %}
    <footer>{% include 'includes/footer.html' %}</footer>
{% endwagtail_cache %}
```

### When to cache a fragment

Cache when:
- The content changes only when a page is published
- The fragment is expensive to render (menus, complex StreamField blocks)
- The fragment appears on many pages (nav, footer, sidebar)

Don't cache when:
- The content varies per user (unless the user ID is in the key)
- The content updates on a timer (news tickers, stock prices)
- It's a form with CSRF tokens

## View-Level Caching

```python
from django.views.decorators.cache import cache_page

@cache_page(60 * 15)
def blog_listing(request):
    ...
```

Useful for non-Wagtail views. Wagtail pages are served through `wagtail_serve`, which is harder to wrap with `cache_page`. Use fragment caching or CDN caching instead.

## Queryset Caching

```python
from django.core.cache import cache

def get_popular_posts():
    cache_key = 'popular_posts_v2'
    posts = cache.get(cache_key)
    if posts is None:
        posts = list(BlogPage.objects.live().order_by('-view_count')[:10])
        cache.set(cache_key, posts, 600)
    return posts
```

Always `list()` the queryset before caching — querysets are lazy, but list evaluation is concrete.

## CDN Caching

### Configuration

```python
WAGTAILFRONTENDCACHE = {
    'cloudflare': {
        'BACKEND': 'wagtail.contrib.frontend_cache.backends.CloudflareBackend',
        'EMAIL': os.environ['CLOUDFLARE_EMAIL'],
        'TOKEN': os.environ['CLOUDFLARE_TOKEN'],
        'ZONEID': os.environ['CLOUDFLARE_ZONEID'],
    }
}
```

Supported backends: Cloudflare, CloudFront, Fastly, Akamai, or a custom backend via `BaseCDNBackend`.

### Purge behavior

Wagtail automatically purges:
- The page's own URL on publish/unpublish
- Pages whose URLs match patterns in `WAGTAILFRONTENDCACHE_URLS`

```python
WAGTAILFRONTENDCACHE_URLS = [
    # Purge the homepage when any page is published
    r'^/$',
    # Purge listing pages
    r'^/blog/$',
    r'^/blog/\d{4}/$',
]
```

### Static file caching

```
# Recommended cache headers for Wagtail static/media:
# /static/ — hashed filenames, cache forever (1 year)
# /media/images/*.jpg?rendition — rendition filenames encode specs, cache forever
# /media/documents/ — cache with caution; documents can change without filename change
```

## Cache Key Design

Good keys are: predictable, unique, invalidatable, and descriptive.

```python
# BAD: opaque, no way to know what it caches
cache.set('x', data)

# GOOD: descriptive, versioned, easy to find and purge
cache.set('blog:popular_posts:v2', data, 600)

# GOOD: specific to an object
cache.set(f'page:{page.id}:sidebar', data, 1800)
```

Include a version number in keys when the rendering logic changes — bump the version, old caches naturally expire.

## Cache Invalidation Strategy

Wagtail publishes = cache purge. But you also need:

```python
# Invalidate when settings change
from wagtail.contrib.settings.models import Setting
from django.db.models.signals import post_save

@receiver(post_save)
def invalidate_on_setting_change(sender, instance, **kwargs):
    if issubclass(sender, Setting):
        cache.delete_pattern('*sidebar*')
        cache.delete_pattern('*nav*')

# Invalidate when snippets change
from wagtail.snippets.models import Snippet
from django.db.models.signals import post_save

@receiver(post_save)
def invalidate_on_snippet_change(sender, instance, **kwargs):
    if issubclass(sender, Snippet):
        cache.delete_pattern(f'*snippet:{sender.__name__}:{instance.pk}*')
```

## What NOT to Cache

1. **CSRF tokens** — they're per-session, caching them creates security errors
2. **User-specific content** — unless the user ID or session key is in the cache key
3. **Forms** — form state, validation errors, and CSRF tokens can't be cached
4. **Admin pages** — Wagtail admin is already fast enough; caching adds stale-data risk
5. **Draft/preview content** — this changes frequently and is per-editor
