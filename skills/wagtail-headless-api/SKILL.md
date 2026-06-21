---
name: wagtail-headless-api
description: 'Build headless / API-first Wagtail setups. Use this any time the user mentions Wagtail headless, decoupled Wagtail, Wagtail as a headless CMS, Wagtail REST API, wagtail-grapple GraphQL, Next.js + Wagtail, Nuxt + Wagtail, consuming Wagtail content from a separate frontend, API caching, or preview URLs in decoupled architectures. Covers REST API v2 field control, GraphQL setup, image renditions over API, CORS, and frontend integration patterns.'
---

# Wagtail Headless / API-First

## Orientation

Wagtail can serve a traditional server-rendered site, a pure headless API consumed by a separate frontend (Next.js, Nuxt, Flutter, native mobile), or a hybrid — Wagtail serves marketing pages while the API feeds a React dashboard. The key architectural choice is: **is Wagtail the frontend, the backend, or both?**

This skill focuses on the API-consumer side. It assumes you already have pages and models; the work is exposing them cleanly, controlling what gets serialized, and making the API fast and stable for frontend consumers.

## Step 0: confirm the architecture

Before writing API config, confirm:
1. **REST or GraphQL?** Wagtail ships REST API v2. GraphQL is available via `wagtail-grapple` (third-party, well-maintained).
2. **Full headless or hybrid?** Is Wagtail admin the ONLY admin, or does the frontend have its own backend?
3. **Preview — hard to get right headless.** Wagtail's preview depends on its template engine. Making preview work against a decoupled Next.js frontend is the single trickiest part of headless Wagtail. Confirm whether preview matters.
4. **Image renditions — on-the-fly or pre-generated?** Wagtail's `{% image %}` tag generates renditions server-side. Headless consumers need rendition URLs in API responses.

## REST API v2

### Enabling

```python
# settings.py
INSTALLED_APPS = [
    'wagtail.api.v2',
    'rest_framework',  # if using DRF alongside
    # ...
]

# urls.py
from wagtail.api.v2.router import WagtailAPIRouter
from wagtail.api.v2.views import PagesAPIViewSet
from wagtail.images.api.v2.views import ImagesAPIViewSet
from wagtail.documents.api.v2.views import DocumentsAPIViewSet

api_router = WagtailAPIRouter('wagtailapi')
api_router.register_endpoint('pages', PagesAPIViewSet)
api_router.register_endpoint('images', ImagesAPIViewSet)
api_router.register_endpoint('documents', DocumentsAPIViewSet)

urlpatterns = [
    path('api/v2/', api_router.urls),
    # Wagtail URLs last
    path('', include(wagtail_urls)),
]
```

### Controlling API Fields Per Page Model

By default, the API exposes `id`, `type`, `detail_url`, `html_url`, `slug`, `title`, `first_published_at`, `latest_revision_created_at`, and a minimal `parent`/`children`/`ancestors`. Everything else (body content, custom fields, images, authors) must be explicitly enabled:

```python
from wagtail.api import APIField
from wagtail.images.api.fields import ImageRenditionField

class BlogPage(Page):
    body = RichTextField()
    intro = models.CharField(max_length=255)
    header_image = models.ForeignKey(
        'wagtailimages.Image', null=True, on_delete=models.SET_NULL
    )
    author = models.CharField(max_length=255)
    tags = ClusterTaggableManager(through=BlogPageTag)

    # API exposure — explicit is the rule
    api_fields = [
        APIField('intro'),
        APIField('body'),
        APIField('author'),
        APIField('tags'),
        APIField('header_image'),
        APIField('header_image_rendition', serializer=ImageRenditionField(
            'fill-1200x600', source='header_image'
        )),
    ]
```

**Convention: every page model gets an explicit `api_fields` list.** Leaving it default means consumers get shell data (title + metadata only) and can't render the page. Be deliberate about what's exposed — export only what the frontend actually needs.

### Image Renditions Over API

Frontend consumers can't use Wagtail's `{% image %}` template tag. Instead, expose rendition URLs:

```python
from wagtail.images.api.fields import ImageRenditionField

class GalleryPage(Page):
    gallery_image = models.ForeignKey('wagtailimages.Image', ...)

    api_fields = [
        APIField('gallery_image'),  # Full image metadata
        APIField('thumbnail', serializer=ImageRenditionField(
            'fill-400x300', source='gallery_image'
        )),
        APIField('hero', serializer=ImageRenditionField(
            'fill-1200x600-c50', source='gallery_image'
        )),
    ]
```

Now the API response includes:
```json
{
    "thumbnail": {
        "url": "/media/images/photo.fill-400x300.jpg",
        "width": 400,
        "height": 300,
        "alt": "A mountain landscape"
    },
    "hero": {
        "url": "/media/images/photo.fill-1200x600-c50.jpg",
        "width": 1200,
        "height": 600,
        "alt": "A mountain landscape"
    }
}
```

Always include `alt` — it's in the image metadata and frontends need it for accessibility.

**Rendition performance**: every unique filter spec creates a new row in `ImageRendition` and a file on disk. Frontend consumers requesting arbitrary dimensions (e.g., `fill-${width}x${height}`) will flood your rendition table. Either:
- Export a fixed set of sizes (thumbnail, medium, hero) and let the frontend pick
- Use an image CDN (Imgix, Cloudinary) instead of Wagtail's rendition system

### StreamField Serialization

StreamField blocks need explicit API representation:

```python
from wagtail import blocks
from wagtail.api import APIField
from wagtail.images.blocks import ImageChooserBlock
import json

class ImageBlock(blocks.StructBlock):
    image = ImageChooserBlock()
    caption = blocks.CharBlock(required=False)

    class Meta:
        icon = 'image'

class APIImageBlock(ImageBlock):
    """API-friendly image block that includes rendition URLs."""
    def get_api_representation(self, value, context=None):
        if not value or not value.get('image'):
            return None
        image = value['image']
        return {
            'image_id': image.id,
            'image_title': image.title,
            'url': image.get_rendition('fill-800x600').url,
            'width': 800,
            'height': 600,
            'alt': image.title,
            'caption': value.get('caption', ''),
        }
```

Every StreamField block that references images, pages, or documents should have a custom `get_api_representation()` that resolves the reference. Don't make the frontend do a second API call for every chooser block.

### Filtering & Ordering

```python
# /api/v2/pages/?type=blog.BlogPage&order=-first_published_at&limit=10&fields=title,intro,header_image_rendition
# /api/v2/pages/?slug=about
# /api/v2/pages/?descendant_of=12  # All pages under page ID 12
```

For custom filtering, subclass `PagesAPIViewSet`:

```python
from wagtail.api.v2.views import PagesAPIViewSet
from wagtail.api.v2.filters import FieldsFilter, OrderingFilter

class BlogPagesAPIViewSet(PagesAPIViewSet):
    filtering_backends = PagesAPIViewSet.filtering_backends + [
        FieldsFilter,
        OrderingFilter,
    ]
    
    meta_fields = PagesAPIViewSet.meta_fields + ['parent']
    
    def get_queryset(self):
        return super().get_queryset().filter(
            content_type__model='blogpage'
        ).live().select_related('owner').prefetch_related('tagged_items__tag')
```

### CORS

Headless = cross-origin requests. In `settings.py`:

```python
INSTALLED_APPS += ['corsheaders']
MIDDLEWARE.insert(0, 'corsheaders.middleware.CorsMiddleware')

# In production, restrict to your frontend domains:
CORS_ALLOWED_ORIGINS = [
    'https://www.example.com',
    'https://admin.example.com',
]
# In dev only:
CORS_ALLOW_ALL_ORIGINS = DEBUG
```

## GraphQL with `wagtail-grapple`

### Setup

```bash
pip install wagtail-grapple
```

```python
# settings.py
INSTALLED_APPS += ['grapple', 'graphene_django']

GRAPHENE = {
    'SCHEMA': 'grapple.schema.schema',
}
```

```python
# urls.py
from graphene_django.views import GraphQLView
from grapple.schema import schema

urlpatterns = [
    path('api/graphql/', GraphQLView.as_view(graphiql=True, schema=schema)),
    # ...
]
```

### Exposing page fields in GraphQL

```python
from grapple.types import StreamFieldType, ImageRenditionType
from grapple.models import GraphQLString, GraphQLImage, GraphQLCollection

class BlogPage(Page):
    body = RichTextField()
    intro = models.CharField(max_length=255)
    header_image = models.ForeignKey('wagtailimages.Image', ...)

    graphql_fields = [
        GraphQLString('intro'),
        GraphQLString('body'),
        GraphQLImage('header_image'),
    ]

    # Custom field with image rendition
    def header_image_url(self, info, **kwargs):
        if self.header_image:
            return self.header_image.get_rendition('fill-1200x600').url
        return None

    graphql_fields += [
        GraphQLString('header_image_url'),
    ]
```

Then query:
```graphql
query {
    pages {
        ... on BlogPage {
            title
            intro
            header_imageUrl
        }
    }
}
```

### When Grapple > REST

- **Varying field shapes**: frontend needs different fields per component and doesn't want over-fetching
- **Nested data in one request**: a page + its related snippets + referenced images all in a single query
- **Frontend team prefers GraphQL**: aligns with their existing tooling (Apollo, Relay, etc.)

### When REST > Grapple

- **Simplicity**: no GraphQL schema to maintain, no N+1 trap (Grapple's default resolvers can be N+1-heavy without careful dataloader setup)
- **Caching at CDN level**: REST responses are easily cached by URL; GraphQL POST requests are harder
- **Small API surface**: a handful of endpoints, not 30+ page types

## Frontend Integration Patterns

### Next.js + Wagtail REST API

```typescript
// lib/wagtail.ts
const WAGTAIL_API = process.env.WAGTAIL_API_URL || 'http://localhost:8000/api/v2';

export async function getPage(slug: string, locale?: string) {
    const params = new URLSearchParams({
        type: 'blog.BlogPage',
        slug,
        fields: 'title,intro,body,header_image_rendition,author,tags,first_published_at',
    });
    if (locale) params.set('locale', locale);

    const res = await fetch(`${WAGTAIL_API}/pages/?${params}`);
    const data = await res.json();
    return data.items?.[0] ?? null;
}

export async function getPageByPath(path: string) {
    const res = await fetch(`${WAGTAIL_API}/pages/find/?html_path=${path}`);
    return res.json();
}
```

```typescript
// app/blog/[slug]/page.tsx
import { getPage } from '@/lib/wagtail';

export default async function BlogPost({ params }: { params: { slug: string } }) {
    const post = await getPage(params.slug);
    return (
        <article>
            <h1>{post.title}</h1>
            <p className="intro">{post.intro}</p>
            {post.header_image_rendition && (
                <img src={post.header_image_rendition.url} alt={post.header_image_rendition.alt} />
            )}
            <div dangerouslySetInnerHTML={{ __html: post.body }} />
        </article>
    );
}

export async function generateStaticParams() {
    const posts = await getPages();
    return posts.map((post: any) => ({ slug: post.slug }));
}
```

### The Preview Problem

Wagtail's built-in preview assumes server-side template rendering. For headless preview:

**Option A: use Wagtail's `serve_preview` endpoint with a frontend preview URL.**
```python
# settings.py
WAGTAIL_PREVIEW_URL = 'https://your-frontend.com/api/preview'
```

Wagtail POSTs preview data to your frontend's preview endpoint. The frontend renders it in a preview mode.

**Option B: use `wagtail-headless-preview` package.**
```bash
pip install wagtail-headless-preview
```
This adds a `HeadlessPreviewMixin` to page models that redirects preview to a configurable frontend URL.

**Option C: live preview via WebSocket (experimental).** Some projects set up a dev-time WebSocket that pushes content changes to the frontend's HMR. This avoids the full preview infrastructure but only works locally.

### Nuxt 3 + Wagtail

Same patterns as Next.js, adjusted for Nuxt's `useFetch` / `useAsyncData`:

```typescript
// composables/useWagtailPage.ts
export const useWagtailPage = (slug: string) => {
    const config = useRuntimeConfig();
    return useAsyncData(`page-${slug}`, () =>
        $fetch(`${config.public.wagtailApiUrl}/pages/`, {
            params: { slug, fields: 'title,body,intro,header_image_rendition' }
        }).then((data: any) => data.items?.[0] ?? null)
    );
};
```

## Cache Strategy

Wagtail API responses should be cached aggressively — they change only when content is published.

```python
# settings.py
WAGTAILAPI_BASE_URL = 'https://cms.example.com/api/v2'

# Use wagtail-cache (Redis-backed) for API responses
CACHES = {
    'default': {
        'BACKEND': 'django_redis.cache.RedisCache',
        'LOCATION': 'redis://127.0.0.1:6379/1',
    }
}

# Cache wagtail-cache fragments in the API
WAGTAIL_CACHE = True
```

At the CDN level (Vercel, Cloudflare, Fastly), cache API responses by URL with invalidation on publish. Wagtail's `page_published` signal can trigger CDN cache purges:

```python
from wagtail.signals import page_published
from django.dispatch import receiver

@receiver(page_published)
def purge_cdn_cache(sender, instance, **kwargs):
    url = f"{settings.WAGTAILAPI_BASE_URL}/pages/{instance.id}/"
    purge_cdn_url(url)  # Your CDN purge logic
```

## Reference files

- **`references/rest-api-v2.md`** — full API reference: all built-in endpoints, filter backends, `FieldsFilter` syntax, pagination, `html_url` vs `detail_url`, the `find` endpoint, custom `APIViewSet` subclass patterns, `meta_fields`, serialization depth control, nested snippets.
- **`references/grapple-graphql.md`** — Grapple deep dive: schema customization, `@register_query_field`, Snippet types, custom resolvers, dataloader N+1 prevention, image rendition types, rich text serialization options (raw HTML vs block-level JSON), subscriptions for live content.
- **`references/frontend-integration.md`** — production integration patterns: SSG/ISR with Next.js, on-demand revalidation on publish, Nuxt 3 `useAsyncData` patterns, SvelteKit + Wagtail, Flutter mobile app consumption, static site generation via wagtailbakery, and the hybrid approach (Wagtail serves some pages, API serves others).
