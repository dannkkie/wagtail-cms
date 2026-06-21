---
name: wagtail-cms-development
description: 'Build and extend websites on Wagtail (the Django CMS at wagtail.org) and Wagtail CRX, formerly known as CodeRed CMS (coderedcorp.com/cms) — a pre-built component layer on top of Wagtail. Use this any time the user mentions Wagtail, StreamField, Wagtail pages/snippets/blocks, CodeRed CMS, CodeRedCMS, Wagtail CRX, CRX, or asks to scaffold, customize, or deploy a Django-based CMS site, even if they just say "build me a website in Django" and a content-managed site is implied. Covers full lifecycle: choosing CRX vs. vanilla Wagtail, scaffolding new projects, writing custom page models and StreamField blocks, snippets and settings, and production deployment.'
---

# Wagtail & Wagtail CRX (CodeRed CMS) Development

## Orientation

**Wagtail** is a Django CMS. Pages are Django models (`Page` subclasses) arranged in a tree, content editors get a rich admin UI, and flexible content lives in `StreamField` — a sequence of typed, reorderable blocks stored as JSON.

**Wagtail CRX** (the project at coderedcorp.com/cms, renamed from "CodeRed CMS" in 2023) is not a fork or a competing product — it's a pip package of pre-built abstract page models, StreamField blocks, and Bootstrap 5 templates that sit on top of stock Wagtail. The project's own framing: *"CRX is to Wagtail what Wagtail is to Django."* Everything that applies to Wagtail applies to a CRX site too; CRX just pre-builds the common stuff (hero sections, card grids, forms, blog/event/location page types, SEO meta tags) so you don't hand-roll it per project.

Because they're genuinely different trade-offs, **don't default to one** — see Step 0.

## Step 0: vanilla Wagtail or CRX? Ask every time.

The user has asked to be consulted per-project rather than have this skill assume an answer — always raise this explicitly for a new project (an existing project's `requirements.txt`/`pyproject.toml` settles it instantly: look for `coderedcms` or `wagtail-crx`).

Lean **CRX** when:
- It's a marketing/business/brochure site — landing pages, blog/news, events, a locations/store-finder, contact forms — the exact territory CRX pre-builds.
- Bootstrap 5 as the CSS foundation is fine (CRX's blocks and templates are tightly coupled to it — this is a real constraint, not a footnote).
- Time-to-launch matters more than pixel-level design control. SEO meta tags, Open Graph/structured data, Google Analytics config, and a multi-step form builder all come free.

Lean **vanilla Wagtail** when:
- The design system is custom (Tailwind, a component library, anything that isn't Bootstrap 5) — fighting CRX's templates costs more than it saves.
- It's headless/API-first (Wagtail's REST API feeding a separate Next.js/Flutter/etc. frontend) — CRX's value is almost entirely in its server-rendered templates, so it buys little here. See `references/headless-and-i18n.md`.
- The content model is unusual enough that CRX's opinionated page types (Article/Event/Location/Form) don't fit and would be fought rather than used.

If it's genuinely ambiguous, ask the user directly rather than guessing — the two paths diverge early (different `start` commands, different base page classes) and switching later is real rework.

## Core conventions (apply to both)

- **Abstract base → concrete subclass.** Both Wagtail and CRX favor abstract base page models (`Page` in vanilla; `Codered*Page` classes in CRX) that you subclass concretely in your own app's `models.py`. Don't edit framework code — extend it. This keeps migrations sane and upgrades survivable.
- **Migrations are real version-controlled artifacts.** `makemigrations` → read the generated file → `migrate` → commit. Never hand-edit a migration unless you specifically know why; flag migration squashing as a deliberate, separate step before production deploys, not something to do casually.
- **Images and documents: never hardcode the model.** Use `get_image_model_string()` / `get_document_model_string()` (and `get_image_model()` / `get_document_model()` for querysets) instead of importing `wagtail.images.Image` directly. This is what makes swapping in a custom Image/Document model later (common in production — see references) possible without a painful migration.
- **StreamField is for *content*, not for data you need to query, validate, or expose via API cleanly.** Page titles, SEO metadata, publish dates, prices, SKUs, important URLs — these belong in real model fields, not buried in StreamField JSON. Reach for StreamField when content structure genuinely varies page-to-page or editors need to reorder/repeat blocks; don't reach for it by default.
- **`content_panels`/`body_content_panels` extend, they don't replace** — when adding fields to a CRX page model, the pattern is `body_content_panels = CoderedWebPage.body_content_panels + [FieldPanel("my_field")]`, not redefining the whole list from scratch (you'd lose the base functionality).

## Workflow: scaffolding a brand-new site

After Step 0 settles vanilla-vs-CRX, confirm Python/Django versions and whether Postgres is wanted from the start (strongly recommended even in dev — SQLite full-text search behaves differently and you'll hit friction migrating later).

**Vanilla Wagtail:**
```bash
python -m venv .venv && source .venv/bin/activate
pip install wagtail
wagtail start mysite
cd mysite
pip install -r requirements.txt   # generated requirements.txt, adjust as needed
python manage.py migrate
python manage.py createsuperuser
python manage.py runserver
```

**Wagtail CRX:**
```bash
python -m venv .venv && source .venv/bin/activate
pip install coderedcms
coderedcms start mysite --template pro --sitename "Site Name" --domain "www.example.com"
cd mysite
pip install -r requirements-dev.txt   # pro template only
python manage.py migrate
python manage.py createsuperuser
python manage.py runserver
```
Default `basic` template is bare-bones; `pro` template (recommended for anything beyond a throwaway prototype) adds custom Image/Document/User models, a custom navbar/footer, SCSS compilation via Python (no Node needed), and ruff/mypy/pytest pre-wired. `--sitename`/`--domain` just pre-populate settings, not load-bearing.

Either way, the scaffold already gives you a `base.py` / `dev.py` / `production.py` settings split — work within it rather than collapsing back to one `settings.py`.

## Workflow: ongoing feature work

This is where most time actually goes. Route to the right reference based on what's being built:

- **New page type** (e.g. "add a product page", "add a careers page") → read `references/vanilla-wagtail.md` (page models section) for plain Wagtail, or `references/crx.md` (custom page types section) when extending a `Codered*Page` base.
- **New StreamField block or editing an existing one's behavior** → `references/vanilla-wagtail.md`, StreamField section — covers `StructBlock`/`ListBlock`/`StreamBlock`, custom `Block` subclasses, validation, and previews. If working in CRX, check `references/crx.md`'s block catalog first — there's a good chance a content or layout block already does most of what's wanted.
- **Reusable content across pages (navbar, footer, testimonials, a "person" card used in multiple places)** → snippets. Both references cover this; CRX additionally ships ready-made Navbar/Footer/Classifier/Reusable Content snippets.
- **Site-wide config (social links, tracking IDs, a logo)** → settings models (`BaseSiteSetting`/`BaseGenericSetting` in vanilla; CRX's `LayoutSettings`/`AnalyticsSettings` plus `CRX_DISABLE_*` flags if you want to turn off CRX's own UI for these and roll your own).
- **Forms** → vanilla Wagtail's `wagtail.contrib.forms`, or CRX's considerably more featureful `CoderedFormPage` (multi-step, conditional logic, MailChimp) — see `references/crx.md`.
- **Headless/API consumption by a separate frontend, or a multi-language site** → `references/headless-and-i18n.md`. Both are decisions to make early (they shape the project from the start) rather than retrofit — flag this if either comes up mid-project rather than treating it as a routine feature add.

Before writing model code, briefly state the plan (field types, panel placement, parent/child page-type restrictions) so it's easy to course-correct before migrations exist.

## Testing & performance

Use `wagtail.test.utils.WagtailPageTestCase` for page-model tests (it gives you `assertPageIsRoutable`, `assertPageIsRenderable`, etc. instead of hand-rolling client requests). For anything serving real traffic, watch for the classic Wagtail N+1: looping over StreamField chooser blocks (images, pages) in a template without prefetching — covered in the performance section of `references/vanilla-wagtail.md`.

## Deployment

Read `references/deployment.md` before taking either a vanilla or CRX site to production — settings split, static/media storage, caching, and a pre-launch security checklist. Both `wagtail start` and `coderedcms start` scaffolds are dev-server-only by default; don't assume they're production-ready as generated.

## Reference files

- **`references/vanilla-wagtail.md`** — page models, StreamField & custom blocks (the deepest section — this is the part of Wagtail that rewards real mastery), panels, snippets, images/documents, search, settings, routable pages, querysets, permissions, testing, performance.
- **`references/crx.md`** — installation, the abstract→concrete page hierarchy and parent/child rules, the full content/layout block catalog, built-in page types (Article/Event/Form/Location), snippets, settings models, SEO, forms, writing custom page types, the Bootstrap 5 coupling and when it becomes a real constraint, and the `CRX_*` Django settings.
- **`references/deployment.md`** — settings split, environment variables, Postgres, static/media storage (including S3-compatible object storage), caching, search backend choice, Docker (including notes for Coolify and similar self-hosted PaaS platforms), and a pre-launch security checklist.
- **`references/headless-and-i18n.md`** — wiring up Wagtail's REST API for a headless/separate-frontend setup, and multi-language sites: vanilla Wagtail's native i18n plus `wagtail-localize`, versus CRX's different recommended approach (`wagtail-modeltranslation`) and why they aren't interchangeable.
