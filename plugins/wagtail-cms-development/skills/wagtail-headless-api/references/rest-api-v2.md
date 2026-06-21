# Wagtail REST API v2 — Full Reference

## Built-in Endpoints

| Endpoint | Description |
|---|---|
| `/api/v2/pages/` | All live, published pages |
| `/api/v2/pages/{id}/` | Single page by ID |
| `/api/v2/pages/find/?html_path=/about/` | Find page by its Wagtail-served URL path |
| `/api/v2/pages/?child_of={id}` | Children of a page |
| `/api/v2/pages/?descendant_of={id}` | All descendants of a page |
| `/api/v2/pages/?ancestor_of={id}` | All ancestors of a page |
| `/api/v2/images/` | All images |
| `/api/v2/documents/` | All documents |
| `/api/v2/snippets/{app_label}/{model_name}/` | Custom snippet endpoints (self-registered) |

## Filtering

### Built-in page filters

```
GET /api/v2/pages/?type=blog.BlogPage&order=-first_published_at&limit=10&offset=20
GET /api/v2/pages/?slug=about-us
GET /api/v2/pages/?show_in_menus=true
GET /api/v2/pages/?locale=fr
```

### FieldsFilter

Returns only the fields you specify:

```
GET /api/v2/pages/?type=blog.BlogPage&fields=title,intro,header_image_rendition,tags,first_published_at
```

Reduces response size dramatically — use this aggressively. The frontend should never receive fields it doesn't need.

### Custom filter backends

```python
from wagtail.api.v2.filters import BaseFilterBackend

class TagFilter(BaseFilterBackend):
    def filter_queryset(self, request, queryset, view):
        tag = request.query_params.get('tag')
        if tag:
            queryset = queryset.filter(tagged_items__tag__slug=tag)
        return queryset

class BlogPagesAPIViewSet(PagesAPIViewSet):
    filter_backends = PagesAPIViewSet.filter_backends + [TagFilter]
```

## Pagination

Default page size: 20 items. Control with `?limit=50&offset=0`.

```python
# settings.py
WAGTAILAPI_LIMIT_MAX = 100
```

## Custom API ViewSets

### Snippets

```python
from wagtail.api.v2.views import BaseAPIViewSet
from wagtail.api.v2.router import WagtailAPIRouter
from .models import Person

class PersonAPIViewSet(BaseAPIViewSet):
    model = Person
    body_fields = ['id', 'name', 'job_title', 'photo', 'bio']
    listing_default_fields = ['id', 'name', 'job_title', 'photo']

api_router.register_endpoint('people', PersonAPIViewSet)
```

### Non-page content types

```python
from wagtail.api.v2.views import BaseAPIViewSet
from wagtail.contrib.redirects.models import Redirect

class RedirectAPIViewSet(BaseAPIViewSet):
    model = Redirect
    body_fields = ['id', 'old_path', 'redirect_link', 'is_permanent']
    listing_default_fields = ['id', 'old_path', 'redirect_link']

api_router.register_endpoint('redirects', RedirectAPIViewSet)
```

## Response Format

```json
{
    "meta": {
        "total_count": 42
    },
    "items": [
        {
            "id": 12,
            "meta": {
                "type": "blog.BlogPage",
                "detail_url": "http://cms.example.com/api/v2/pages/12/",
                "html_url": "http://cms.example.com/blog/my-post/",
                "slug": "my-post",
                "first_published_at": "2024-03-15T10:30:00Z",
                "parent": {
                    "id": 3,
                    "meta": { "type": "blog.BlogIndexPage" }
                }
            },
            "title": "My Blog Post",
            "intro": "A short intro...",
            "header_image_rendition": {
                "url": "/media/images/hero.fill-1200x600.jpg",
                "width": 1200,
                "height": 600,
                "alt": "Hero image description"
            }
        }
    ],
    "total_count": 42
}
```

## `meta_fields` — Adding Default Metadata

```python
class BlogPagesAPIViewSet(PagesAPIViewSet):
    meta_fields = PagesAPIViewSet.meta_fields + [
        'parent',
        'first_published_at',
        'last_published_at',
        'seo_title',
        'search_description',
    ]
```

## Serialization Depth

By default, foreign keys return `{id, meta}`. Control depth:

```python
class BlogPage(Page):
    author = models.ForeignKey(Author, ...)

    api_fields = [
        APIField('author'),  # Default: {id, meta}
        # For nested serialization, write a custom serializer:
        APIField('author_detail', serializer=AuthorDetailSerializer(source='author')),
    ]
```

## Performance

- Use `?fields=` to reduce response size
- `select_related()` on FK references in queryset overrides
- `prefetch_related()` for M2M relationships
- Wagtail's API viewset uses `LivePageManager` by default — already excludes unpublished/draft pages
- For image-heavy responses, consider a CDN that caches rendition URLs rather than hitting Wagtail's media server
