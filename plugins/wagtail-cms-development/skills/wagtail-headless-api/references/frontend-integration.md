# Frontend Integration Patterns — Production Reference

## Static Site Generation (Next.js)

```typescript
// app/blog/[slug]/page.tsx
export async function generateStaticParams() {
    const pages = await fetchAllPages('blog.BlogPage');
    return pages.map((page) => ({ slug: page.meta.slug }));
}

// Revalidate on demand when content changes
export const revalidate = 3600; // ISR: revalidate every hour
```

### On-Demand Revalidation

Expose a revalidation endpoint in your Next.js app that Wagtail calls on publish:

```python
# In Wagtail signals
from wagtail.signals import page_published
from django.dispatch import receiver
import requests

@receiver(page_published)
def revalidate_frontend(sender, instance, **kwargs):
    url = instance.full_url  # or construct the frontend URL
    requests.post(
        'https://your-frontend.com/api/revalidate',
        json={'url': url, 'secret': settings.REVALIDATION_SECRET},
        timeout=5,
    )
```

```typescript
// app/api/revalidate/route.ts
import { revalidatePath } from 'next/cache';

export async function POST(request: Request) {
    const { url, secret } = await request.json();
    if (secret !== process.env.REVALIDATION_SECRET) {
        return Response.json({ error: 'Invalid secret' }, { status: 401 });
    }
    revalidatePath(url);
    return Response.json({ revalidated: true });
}
```

## Nuxt 3

```typescript
// composables/useWagtailPage.ts
export const useWagtailPage = async (slug: string) => {
    const config = useRuntimeConfig();
    const { data, error } = await useAsyncData(
        `page-${slug}`,
        () => $fetch(`${config.public.wagtailApiUrl}/pages/`, {
            params: { slug, type: 'blog.BlogPage', fields: 'title,body,header_image_rendition' }
        }).then((d: any) => d.items?.[0] ?? null),
        { server: true }
    );
    return { page: data.value, error };
};
```

## SvelteKit

```typescript
// src/routes/blog/[slug]/+page.server.ts
export async function load({ params }) {
    const res = await fetch(`${WAGTAIL_API}/pages/?slug=${params.slug}&type=blog.BlogPage`);
    const data = await res.json();
    return { page: data.items?.[0] ?? null };
}
```

## Flutter / Mobile

```dart
class WagtailService {
  final String baseUrl;

  Future<Page?> getPage(String slug) async {
    final uri = Uri.parse('$baseUrl/api/v2/pages/').replace(queryParameters: {
      'slug': slug,
      'fields': 'title,body,header_image_rendition,intro',
    });
    final response = await http.get(uri);
    final data = json.decode(response.body);
    return data['items']?.isNotEmpty == true ? Page.fromJson(data['items'][0]) : null;
  }
}
```

## Hybrid Approach

The most common production pattern: Wagtail serves marketing pages server-side (SEO, fast TTFB, preview works) while the API feeds complex interactive sections (dashboards, search, user-facing tools).

```python
# urls.py — Wagtail serve handles some paths, API handles others
urlpatterns = [
    path('api/v2/', api_router.urls),
    path('dashboard/', include('dashboard.urls')),  # React SPA, API-only
    path('', include(wagtail_urls)),  # Wagtail serves everything else
]
```

This gets you the best of both: Wagtail's template rendering where it shines, API where you need interactivity.

## Preview in Headless

### HeadlessPreviewMixin

```python
from wagtail_headless_preview.models import HeadlessPreviewMixin

class BlogPage(HeadlessPreviewMixin, Page):
    body = RichTextField()

    # settings.py
    HEADLESS_PREVIEW_URL = 'https://your-frontend.com/api/preview'
```

The mixin redirects Wagtail's preview button to your frontend URL with a token. The frontend fetches preview content from the Wagtail API with the token and renders it.

### Direct Preview API

If you don't want a third-party package:

```python
# views.py
from django.http import JsonResponse
from wagtail.models import Page

def preview_api(request, page_id):
    page = Page.objects.get(id=page_id).specific
    revision = page.get_latest_revision()
    content = revision.as_object()
    return JsonResponse({
        'title': content.title,
        'body': str(content.body),
        'preview_token': str(revision.id),
    })
```

Frontend fetches this endpoint and renders the preview content.
