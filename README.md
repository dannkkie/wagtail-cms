# wagtail-cms

A Claude Code / Claude.ai skill for building on **[Wagtail](https://wagtail.org)** (the Django CMS) and **[Wagtail CRX](https://www.coderedcorp.com/cms/)**, formerly known as CodeRed CMS.

Covers the full lifecycle: deciding between vanilla Wagtail and CRX for a given project, scaffolding new sites, building custom page models and StreamField blocks, snippets and settings, headless/REST API setups, multi-language sites, and production deployment.

## Install

```
/plugin marketplace add dannkkie/wagtail-cms
/plugin install wagtail-cms-development@wagtail-cms
```

Then just work as normal — Claude reaches for the skill automatically any time Wagtail, StreamField, CRX, or CodeRed CMS comes up.

## What's included

- **Decision framework** — when to reach for CRX vs. vanilla Wagtail on a new project, and the signals that mean you should switch mid-project.
- **`vanilla-wagtail.md`** — page models, StreamField & custom blocks, panels, snippets, images/documents, search, settings models, routable pages, querysets, testing, performance.
- **`crx.md`** — installation, the abstract→concrete page model pattern, parent/child page rules, the full content/layout block catalog, built-in page types, SEO, forms, writing custom page types, `CRX_*` Django settings, and the Bootstrap 5 coupling.
- **`deployment.md`** — settings split, environment variables, storage, caching, Docker (incl. Coolify notes), and a pre-launch security checklist.
- **`headless-and-i18n.md`** — wiring up Wagtail's REST API for headless setups, and multi-language sites (vanilla Wagtail's `wagtail-localize` vs. CRX's different recommended approach, `wagtail-modeltranslation`).

## License

MIT
