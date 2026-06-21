---
name: wagtail-search
description: 'Configure and optimize Wagtail search. Use this any time the user mentions Wagtail search, search backends, full-text search, Elasticsearch, Postgres full-text search, search indexing, search_fields configuration, autocomplete, faceted search, search relevance tuning, or search performance. Covers database FTS (Postgres trigram), Elasticsearch setup and indexing, faceted/autocomplete, and the decision of when to switch from DB to dedicated search.'
---

# Wagtail Search

## Orientation

Wagtail's search is a layered abstraction: your code calls `Page.objects.search("query")`, and the configured backend handles the actual indexing and retrieval. Wagtail ships with a database backend that works out of the box and an Elasticsearch backend for production.

The key decisions are:
1. **Which backend?** DB for dev and small sites; Elasticsearch for production at scale.
2. **What gets indexed?** Pages, images, documents, and optionally custom models via `index.get_indexed_models()`.
3. **How fields are weighted and configured.** The `search_fields` list on each model determines what's searchable and how heavily each field counts.

## Backends

### Database (default)

```python
# settings.py — this is the default, no config needed
WAGTAILSEARCH_BACKENDS = {
    'default': {
        'BACKEND': 'wagtail.search.backends.database',
        'SEARCH_CONFIG': 'english',  # Postgres text search config
    }
}
```

Uses Postgres full-text search (`to_tsvector`, `to_tsquery`). Good for <10K pages. Falls back to `icontains` on SQLite (much slower, no ranking).

### Postgres Trigram (better fuzzy matching)

```python
WAGTAILSEARCH_BACKENDS = {
    'default': {
        'BACKEND': 'wagtail.search.backends.database',
        'SEARCH_CONFIG': 'english',
    }
}
```

Enable the `pg_trgm` extension:
```sql
CREATE EXTENSION IF NOT EXISTS pg_trgm;
```

Trigram enables similarity matching (`similarity()` function), which handles typos and partial-word matches far better than plain `tsquery`. Add `ATOMIC_REBUILD = False` for large datasets — rebuilds use batched inserts instead of a single transaction.

### Elasticsearch (production)

```bash
pip install elasticsearch>=8.0,<9.0
```

```python
WAGTAILSEARCH_BACKENDS = {
    'default': {
        'BACKEND': 'wagtail.search.backends.elasticsearch8',
        'URLS': [os.environ.get('ELASTICSEARCH_URL', 'http://localhost:9200')],
        'INDEX': 'wagtail',
        'TIMEOUT': 5,
        'OPTIONS': {
            'settings': {
                'number_of_shards': 1,
                'number_of_replicas': 0,
            }
        },
    }
}
```

After configuring, build the index:
```bash
python manage.py update_index
```

Schedule this regularly — content changes need index updates:
```bash
# cron: every 15 minutes for active sites
*/15 * * * * /app/manage.py update_index --age=1h
```

### Elasticsearch AWS / OpenSearch

```python
WAGTAILSEARCH_BACKENDS = {
    'default': {
        'BACKEND': 'wagtail.search.backends.elasticsearch8',
        'URLS': ['https://search-xxx.us-east-1.es.amazonaws.com'],
        'INDEX': 'wagtail',
        'OPTIONS': {
            'http_auth': ('username', 'password'),
            'use_ssl': True,
            'verify_certs': True,
        }
    }
}
```

## `search_fields` Configuration

### Page models

```python
from wagtail.search import index

class BlogPage(Page):
    body = RichTextField()
    intro = models.CharField(max_length=255)
    categories = ParentalManyToManyField('blog.BlogCategory')

    search_fields = Page.search_fields + [
        # Full-text search on these fields
        index.SearchField('body', boost=2),
        index.SearchField('intro', boost=3),
        index.SearchField('get_category_names', boost=1),
        
        # For exact matches/filtering (not full-text)
        index.FilterField('first_published_at'),
        index.FilterField('categories'),
    ]

    def get_category_names(self):
        return ' '.join(c.name for c in self.categories.all())
```

### Field types

| Field | Purpose |
|---|---|
| `index.SearchField(field_name, boost=1)` | Full-text indexed; higher boost = higher relevance |
| `index.FilterField(field_name)` | Exact match filtering, not full-text |
| `index.AutocompleteField(field_name)` | For `autocomplete()` queries — prefix/substring matching |
| `index.RelatedFields(fk_field, fields)` | Index fields on a related model (FK, ParentalKey) |

### Boosting

Boosts are multipliers. A `boost=3` field contributes 3× more to the relevance score than a `boost=1` field. Title should usually be the highest-boost field:

```python
search_fields = [
    index.SearchField('title', boost=5),
    index.SearchField('intro', boost=3),
    index.SearchField('body', boost=1),
]
```

### Related fields

```python
search_fields = [
    index.SearchField('title'),
    index.RelatedFields('author', [
        index.SearchField('name', boost=2),
        index.SearchField('bio'),
    ]),
    index.RelatedFields('categories', [
        index.SearchField('name', boost=1.5),
    ]),
]
```

## Searching — API

```python
# Basic search
Page.objects.live().search("django wagtail tutorial")

# With filters
BlogPage.objects.live().search(
    "tutorial",
    order_by_relevance=True,
    operator='and',  # 'or' (default) or 'and'
).filter(
    first_published_at__gte='2024-01-01'
)

# Autocomplete
BlogPage.objects.live().autocomplete("tuto")  # Matches "tutorial", "tutor", etc.

# Faceted search
from wagtail.search import query

results = Page.objects.live().search("django")
faceted = results.facet('content_type')
# faceted = {'blog.BlogPage': 15, 'blog.AuthorPage': 3, ...}
```

Elasticsearch-only features (not available on DB backend):
- `facet()` — faceted counts
- `autocomplete()` — prefix matching
- `order_by_relevance=True` — true relevance ranking
- `operator='and'` — boolean AND matching
- `phrase=True` — exact phrase matching

## Indexing Custom Models

```python
from wagtail.search import index

class Product(index.Indexed, models.Model):
    name = models.CharField(max_length=255)
    description = models.TextField()
    price = models.DecimalField(max_digits=8, decimal_places=2)

    search_fields = [
        index.SearchField('name', boost=3),
        index.SearchField('description'),
        index.FilterField('price'),
    ]
```

Then register:
```python
# In wagtail_hooks.py or AppConfig.ready()
from wagtail.search.index import register_indexed_model
register_indexed_model(Product)
```

Update indexes:
```bash
python manage.py update_index
```

## Performance

### When to switch to Elasticsearch

Switch when ANY of these are true:
- Page count > 10,000 (DB search latency crosses 200ms)
- Need autocomplete (not available on DB backend)
- Need true relevance ranking (DB backend uses Postgres `ts_rank`, which is crude)
- Search latency consistently > 200ms in production
- Users complain about search quality (DB backend can't do fuzzy matching without trigram)

### Elasticsearch sizing

| Pages | Shards | Replicas | RAM |
|---|---|---|---|
| <10K | 1 | 0 | 512MB |
| 10K–100K | 2 | 1 | 1–2GB |
| 100K–1M | 3 | 1–2 | 2–4GB |
| 1M+ | 5+ | 2+ | 4GB+ |

### The `update_index` strategy

Don't run `update_index` without `--age` in production — it reindexes everything:

```bash
# Reindex only recently changed content
python manage.py update_index --age=1h

# Reindex everything (use rarely, off-peak hours)
python manage.py update_index
```

### Query optimization

```python
# BAD: loads full page object (including StreamField JSON) for search results
results = Page.objects.search("query")

# GOOD: only load what you need for the search results page
results = Page.objects.search("query").select_related(
    'locale'
).defer_streamfields()[:20]
```

## Autocomplete

### Setup

```python
search_fields = [
    index.AutocompleteField('title'),
    index.AutocompleteField('intro'),
]
```

### Query

```python
suggestions = BlogPage.objects.live().autocomplete("wagt")  # Matches "wagtail"
```

### Frontend

```javascript
// Fetch autocomplete suggestions as user types
const input = document.getElementById('search-input');
input.addEventListener('input', async (e) => {
    const res = await fetch(`/api/v2/pages/?autocomplete=${e.target.value}&limit=5`);
    const data = await res.json();
    renderSuggestions(data.items);
});
```

For production autocomplete with low latency, use Elasticsearch — the DB backend can't do prefix matching efficiently.

## Relevance Tuning

### With Elasticsearch

```python
WAGTAILSEARCH_BACKENDS = {
    'default': {
        'BACKEND': 'wagtail.search.backends.elasticsearch8',
        'INDEX_SETTINGS': {
            'settings': {
                'index': {
                    'similarity': {
                        'default': {
                            'type': 'BM25',
                            'b': 0.75,
                            'k1': 1.2,
                        }
                    }
                }
            }
        }
    }
}
```

BM25 parameters: `k1` (term saturation, default 1.2) and `b` (length normalization, default 0.75). Tuning these is an art — start with defaults, adjust if search quality feels off.

### With Postgres

Postgres `ts_rank` is limited compared to BM25. Use `setweight()` in custom queries for more control:
```sql
SELECT setweight(to_tsvector('english', title), 'A') ||
       setweight(to_tsvector('english', body), 'C')
FROM blog_blogpage;
```

Weights A–D: A is highest, D is lowest. Wagtail's DB backend maps `boost` to Postgres weights internally.
