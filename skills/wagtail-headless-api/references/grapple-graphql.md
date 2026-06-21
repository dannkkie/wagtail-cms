# Wagtail Grapple (GraphQL) — Deep Reference

## Architecture

Grapple wraps Wagtail models as Graphene types. Each registered page model gets a GraphQL type with fields you declare in `graphql_fields`. The `pages` query returns an `Interface` that resolves to the correct type.

## Registration

### Page models

```python
from grapple.models import (
    GraphQLString,
    GraphQLRichText,
    GraphQLImage,
    GraphQLForeignKey,
    GraphQLStreamfield,
    GraphQLCollection,
)

class BlogPage(Page):
    body = RichTextField()
    header_image = models.ForeignKey('wagtailimages.Image', ...)
    author = models.ForeignKey('blog.Author', ...)

    graphql_fields = [
        GraphQLString('intro'),
        GraphQLRichText('body'),
        GraphQLImage('header_image'),
        GraphQLForeignKey('author', 'blog.Author'),
    ]
```

### Image renditions

```python
class BlogPage(Page):
    graphql_fields = [
        GraphQLImage('header_image', rendition='fill-1200x600'),
    ]
```

Query becomes:
```graphql
{ header_image { url, width, height, alt } }
```

### StreamField

```python
GraphQLStreamfield('body')
```

Returns a raw JSON representation. For typed StreamField access, define a custom type per block.

### Collections/FK lists

```python
GraphQLCollection(
    GraphQLForeignKey, 'related_posts', 'blog.BlogPage',
    source='related_posts'
)
```

## Snippet Types

```python
from grapple.types.snippets import SnippetType
from grapple.registry import register_snippet_type

@register_snippet_type
class AuthorType(SnippetType):
    model = Author
    fields = ['name', 'bio', 'photo', 'website']
```

## Custom Resolvers

```python
class BlogPage(Page):
    graphql_fields = [
        GraphQLString('reading_time'),
        GraphQLString('related_posts_formatted'),
    ]

    def resolve_reading_time(self, info, **kwargs):
        words = len(self.body.split())
        return f"{max(1, words // 200)} min read"

    def resolve_related_posts_formatted(self, info, **kwargs):
        return [
            {'title': p.title, 'url': p.url, 'date': str(p.first_published_at)}
            for p in self.related_posts.all()[:5]
        ]
```

## N+1 and Dataloaders

Grapple's default resolvers can cause N+1 queries. Use Django's `select_related`/`prefetch_related`:

```python
class BlogPagesAPIViewSet(PagesAPIViewSet):
    def get_queryset(self):
        return super().get_queryset().select_related(
            'header_image', 'author'
        ).prefetch_related('tags', 'related_posts')
```

For more complex N+1 patterns, use `graphene-django`'s `DataLoader`:

```python
from promise.dataloader import DataLoader

class RelatedPostsLoader(DataLoader):
    def batch_load_fn(self, page_ids):
        posts = BlogPage.objects.filter(
            related_from__page__in=page_ids
        ).select_related('header_image')
        return [[p for p in posts if p.id == pid] for pid in page_ids]
```

## Rich Text Serialization

```python
GraphQLRichText('body')  # Returns raw HTML
```

For structured rich text, write a custom serializer:

```python
def resolve_body_blocks(self, info, **kwargs):
    """Return rich text as structured blocks."""
    from wagtail.rich_text import expand_db_html
    html = expand_db_html(self.body)
    # Parse HTML into structured blocks
    return parse_rich_text_to_blocks(html)
```

## Mutations (Experimental)

Grapple doesn't ship mutations by default. For content updates via GraphQL, you'd add a custom mutation:

```python
import graphene
from wagtail.models import Page

class PublishPageMutation(graphene.Mutation):
    class Arguments:
        page_id = graphene.Int(required=True)

    success = graphene.Boolean()

    def mutate(self, info, page_id):
        page = Page.objects.get(id=page_id).specific
        revision = page.save_revision()
        revision.publish()
        return PublishPageMutation(success=True)
```

This is unusual — most headless setups use Wagtail's admin for content editing and only read via GraphQL.

## Performance Tips

1. **Always `select_related`/`prefetch_related`** in the page queryset override
2. **Use `GraphQLString` with custom resolvers** instead of `GraphQLStreamfield` when you only need specific StreamField values
3. **Cache image renditions** at the CDN level — Grapple returns rendition URLs that are static per spec
4. **Limit the number of items in `GraphQLCollection`** — large collections slow down queries for ALL consumers
