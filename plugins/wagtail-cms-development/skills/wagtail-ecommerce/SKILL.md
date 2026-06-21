---
name: wagtail-ecommerce
description: 'Integrate ecommerce capabilities with Wagtail. Use this any time the user mentions Wagtail + ecommerce, Wagtail shop, product pages in Wagtail, connecting Wagtail to Saleor/Shopify, cart/checkout with Wagtail content, or building a content-driven store with Wagtail as the CMS layer.'
---

# Wagtail Ecommerce

## Orientation

Wagtail is a CMS, not an ecommerce platform. The pattern is: **Wagtail manages the content** (product descriptions, landing pages, blog, brand storytelling) while a dedicated ecommerce system handles transactions, inventory, carts, and payments.

Common pairings:
- **Saleor** — Django-native GraphQL-first ecommerce; Wagtail + Saleor is the most Django-idiomatic stack
- **Shopify** — embed Shopify Buy Buttons or use Storefront API; small shops
- **Oscar** — Django ecommerce framework; heavy, full-featured, legacy
- **Snipcart** — add a cart to any HTML; easiest integration, 2% transaction fee
- **Medusa** — Node.js headless commerce; good if frontend is already JS

## Decision framework

| Setup | Best for |
|---|---|
| Wagtail product pages + Snipcart | Small catalog (<50 products), fast to launch |
| Wagtail + Saleor | Larger Django-native stack, team knows Django |
| Wagtail + Shopify Storefront API | Mid-size catalog, Shopify handles logistics |
| Wagtail + Oscar | Complex ecommerce rules (b2b, wholesale, subscriptions) |

## Wagtail Product Pages + Snipcart

The simplest approach. Wagtail manages product content; Snipcart adds a cart/checkout overlay.

```python
class ProductPage(Page):
    price = models.DecimalField(max_digits=8, decimal_places=2)
    description = RichTextField()
    image = models.ForeignKey('wagtailimages.Image', ...)
    sku = models.CharField(max_length=100)
    
    content_panels = Page.content_panels + [
        FieldPanel('sku'),
        FieldPanel('price'),
        FieldPanel('description'),
        FieldPanel('image'),
    ]
```

Template:
```html
<!-- Add Snipcart JS and a buy button -->
<button class="snipcart-add-item"
    data-item-id="{{ page.sku }}"
    data-item-price="{{ page.price }}"
    data-item-url="{{ page.full_url }}"
    data-item-name="{{ page.title }}"
    data-item-image="{{ page.image.get_rendition('fill-800x800').url }}">
    Add to cart
</button>
```

## Wagtail + Saleor

Saleor is the most Django-idiomatic ecommerce backend. Wagtail and Saleor run as separate services sharing patterns.

### Architecture
```
[Wagtail CMS] ←--→ [Saleor API] ←--→ [Next.js Storefront]
     |                    |
  Content pages      Product catalog
  Blog, landing      Cart, checkout
  Brand storytelling  Inventory, orders
```

### Integration pattern

Wagtail doesn't directly access Saleor's database. Instead, Wagtail pages fetch product data from Saleor's GraphQL API:

```python
# services/saleor.py
import requests

class SaleorClient:
    def __init__(self, api_url, token):
        self.api_url = api_url
        self.token = token

    def get_product(self, slug):
        query = '''
        query($slug: String!) {
            product(slug: $slug) {
                id, name, description, 
                thumbnail { url, alt }
                pricing { priceRange { start { gross { amount } } } }
                variants { id, name, quantityAvailable }
            }
        }'''
        resp = requests.post(
            self.api_url,
            json={'query': query, 'variables': {'slug': slug}},
            headers={'Authorization': f'Bearer {self.token}'},
        )
        return resp.json()['data']['product']
```

```python
class ProductPage(Page):
    saleor_product_slug = models.CharField(max_length=255)

    def get_context(self, request):
        context = super().get_context(request)
        client = SaleorClient(settings.SALEOR_API_URL, settings.SALEOR_TOKEN)
        context['product'] = client.get_product(self.saleor_product_slug)
        return context
```

Cache Saleor responses in Redis to avoid API calls on every page view.

## Wagtail + Shopify Storefront API

```python
# services/shopify.py
import shopify

class ShopifyClient:
    def __init__(self, domain, token):
        self.session = shopify.Session(domain, '2024-01', token)
        shopify.ShopifyResource.activate_session(self.session)
    
    def get_product(self, handle):
        products = shopify.Product.find(handle=handle)
        return products[0] if products else None
```

Same pattern as Saleor — Wagtail pages reference Shopify product handles, fetch product data server-side, cache in Redis.

## Wagtail + Oscar

Oscar is a Django ecommerce framework. Wagtail and Oscar run in the SAME Django project:

```python
# urls.py
urlpatterns = [
    path('shop/', include('oscar.urls')),  # Oscar handles /shop/*
    path('admin/', include(wagtailadmin_urls)),
    # Wagtail serves content pages from root
    path('', include(wagtail_urls)),
]
```

Wagtail pages can access Oscar product data directly:

```python
from oscar.core.loading import get_model
Product = get_model('catalogue', 'Product')

class ProductPage(Page):
    oscar_product = models.ForeignKey(Product, on_delete=models.SET_NULL, null=True)

    def get_context(self, request):
        context = super().get_context(request)
        if self.oscar_product:
            context['product'] = self.oscar_product
            context['add_to_cart_url'] = reverse('basket:add', args=[self.oscar_product.id])
        return context
```

## Common patterns across all integrations

### Content-driven commerce

Wagtail excels at the content around products:
- **Buying guides** — StreamField pages with embedded product references
- **Lookbooks / collections** — curated galleries linking to product pages
- **Blog → product attribution** — "Shop the look" widgets on blog posts

### Product data — where does it live?

| Data | Wagtail | Ecommerce system |
|---|---|---|
| Product description/marketing copy | ✅ | Maybe |
| Price, inventory, SKU | ❌ | ✅ (source of truth) |
| Product images | ✅ (for content) | ✅ (for checkout) |
| Categories/taxonomy | Sync'd | ✅ (source of truth) |
| Landing pages, blog | ✅ | ❌ |
| Reviews, UGC | Store in ecommerce | Store in ecommerce |

### Cache aggressively

Product data from external APIs must be cached:

```python
from django.core.cache import cache

def get_product(slug):
    key = f'product:{slug}'
    product = cache.get(key)
    if product is None:
        product = saleor_client.get_product(slug)
        cache.set(key, product, 300)  # 5 min TTL
    return product
```

Invalidate on product webhook: ecommerce system POSTs to Wagtail when a product changes.
