# wagtail-cms

[![skills.sh](https://skills.sh/b/dannkkie/wagtail-cms)](https://skills.sh)

A Claude Code / Claude.ai skill for building on **[Wagtail](https://wagtail.org)** (the Django CMS) and **[Wagtail CRX](https://www.coderedcorp.com/cms/)**, formerly known as CodeRed CMS.

Covers the full lifecycle: deciding between vanilla Wagtail and CRX for a given project, scaffolding new sites, building custom page models and StreamField blocks, snippets and settings, headless/REST API setups, multi-language sites, and production deployment.

## Install

### Any AI agent (skills.sh CLI)

```
npx skills add dannkkie/wagtail-cms
```

### Claude Code (plugin marketplace)

```
/plugin marketplace add dannkkie/wagtail-cms
/plugin install wagtail-cms-development@wagtail-cms
```

Then just work as normal — Claude reaches for the skill automatically any time Wagtail, StreamField, CRX, or CodeRed CMS comes up.

## Skills

| Skill | Description |
|---|---|
| **wagtail-cms-development** | Core skill — page models, StreamField blocks, CRX, snippets, settings, deployment. Decision framework for CRX vs. vanilla Wagtail. |
| **wagtail-localize-i18n** | Multi-language sites — wagtail-localize, modeltranslation, PO file workflows, locale tree structure, migration between strategies. |
| **wagtail-headless-api** | Headless/API-first — REST API v2, wagtail-grapple GraphQL, Next.js/Nuxt integration, preview URLs, image renditions over API. |
| **wagtail-performance** | Production performance — N+1 patterns catalog, template caching, wagtail-cache + Redis, image CDN, query-count regression testing. |
| **wagtail-custom-admin** | Custom admin — ModelAdmin/SnippetViewSet, hooks, custom panels & widgets, telepath/React, moderation workflow, custom choosers. |
| **wagtail-search** | Search backends — Postgres FTS/trigram, Elasticsearch setup, autocomplete, faceted search, relevance tuning, `search_fields` configuration. |
| **wagtail-ecommerce** | Ecommerce integration — Wagtail + Saleor/Shopify/Snipcart/Oscar product pages, content-driven commerce, caching product API data. |
| **wagtail-migrations** | Safe migrations — StreamField data migration patterns, zero-downtime deploys, page tree restructuring, squashing, CI testing. |
| **wagtail-accessibility** | A11y — accessible templates, ARIA patterns, image alt text enforcement, form labels, keyboard navigation, admin a11y testing. |

## License

MIT
