# Deploying a Wagtail / Wagtail CRX site

Contents: Settings split · Environment variables · Database · Static & media files · Caching · Search backend at scale · Docker (incl. Coolify/self-hosted PaaS notes) · Pre-launch security checklist

Applies to both vanilla Wagtail and CRX projects — CRX adds a couple of CRX-specific env vars (noted below) but the deployment shape is otherwise identical, since CRX is "just Wagtail."

---

## Settings split

Both `wagtail start` and `coderedcms start` scaffold a settings package (`mysite/settings/base.py`, `dev.py`, `production.py`) rather than one flat file — keep working within that shape rather than collapsing it. `production.py` should:
- Import from `base.py`, override only what differs.
- Set `DEBUG = False` unconditionally (never derive it from an env var that might silently default to truthy).
- Pull secrets from environment variables, never commit them.

`manage.py`/`wsgi.py` typically select the settings module via `DJANGO_SETTINGS_MODULE` env var — set this explicitly in the production environment rather than relying on a default.

---

## Environment variables

At minimum: `DJANGO_SETTINGS_MODULE`, `SECRET_KEY`, `ALLOWED_HOSTS` (comma-separated, no wildcards in production), `DATABASE_URL` (consumed via `dj-database-url` or `environ.Env`, both common in Wagtail scaffolds). For CRX specifically, also consider explicit env-driven overrides for `CRX_BANNER` (handy for instantly flagging a staging deploy) and the protected-media settings if uploads go to non-default storage.

---

## Database

Postgres is the de facto standard for Wagtail in production — better full-text search behavior than SQLite/MySQL, and what most Wagtail-specific tooling assumes. `psycopg` (v3) is the current standard driver. Use the same engine in development as production if at all possible; SQLite-in-dev/Postgres-in-prod is a common source of "works on my machine" search and migration surprises.

---

## Static & media files

- **Static** (CSS/JS/admin assets): `collectstatic` into a directory served by Whitenoise (simplest — no separate static file server needed) or a CDN/object-storage bucket for higher-traffic sites.
- **Media** (uploaded images/documents): do **not** rely on local disk storage in any environment where the filesystem isn't persistent across deploys (most container platforms, including Coolify, fall into this category unless a persistent volume is explicitly mounted). Use `django-storages` with an S3-compatible backend — AWS S3 itself, or any S3-compatible object store (DigitalOcean Spaces, Backblaze B2, Cloudflare R2, etc.) all work with the same `django-storages` configuration, just different endpoint/credentials.
- CRX's protected media (form file uploads, see `references/crx.md`) needs the same storage consideration — don't assume it lives safely on local disk in production.

---

## Caching

`wagtail-cache` (pip install separately) provides full-page caching keyed correctly against Wagtail's publish/unpublish/preview cycle — it knows to invalidate when a page changes, unlike a naive reverse-proxy cache. Pair with Redis as the cache backend for anything beyond a single-process toy deployment. For a marketing/content-heavy site (the CRX sweet spot), this is usually the single highest-leverage performance change available.

---

## Search backend at scale

Default database search (Postgres full-text) is genuinely fine for most sites — don't add Elasticsearch/OpenSearch infrastructure preemptively. Revisit only once there's an actual signal: large content volume, multi-language search needs, or search-relevance complaints database search can't address through field weighting (`index.SearchField(..., boost=...)`).

---

## Docker

CRX ships an official Docker how-to (`docs.coderedcorp.com/wagtail-crx/how_to/docker.html`) — follow it for the CRX-specific bits (note: as of CRX 0.19+, `coderedcms start` no longer scaffolds a boilerplate Dockerfile automatically; write one following that guide rather than expecting it pre-generated). A vanilla Wagtail Dockerfile follows the standard Django container shape: install deps, `collectstatic`, run migrations as a release step (not inside the image build), serve via `gunicorn`.

**Self-hosted PaaS platforms (Coolify and similar):** these generally expect a standard Dockerfile + exposed port + env vars — nothing CRX- or Wagtail-specific beyond what's already covered above. The one thing worth being deliberate about: if media storage isn't offloaded to S3-compatible object storage (previous section), a persistent volume must be explicitly configured for the media directory, or uploaded images/documents vanish on the next redeploy. Run `migrate` as part of the deploy step (most of these platforms support a pre/post-deploy command hook) rather than baking a stale migration state into the image.

---

## Pre-launch security checklist

- `DEBUG = False`, confirmed in the actual production environment (not just the file — check the running process's resolved settings).
- `ALLOWED_HOSTS` set to the real domain(s) only.
- `SECRET_KEY` from environment, not the scaffold-generated default, not committed to version control.
- `CSRF_TRUSTED_ORIGINS` set if the admin is accessed over a domain Django doesn't infer automatically (common with reverse proxies).
- `SECURE_SSL_REDIRECT`, `SESSION_COOKIE_SECURE`, `CSRF_COOKIE_SECURE` — all `True` once HTTPS is confirmed working end-to-end (enable after verifying TLS termination, not before, or you'll lock yourself out mid-setup).
- Default Wagtail admin path is `/admin/`/`/django-admin/` by convention — consider whether to leave as-is (most sites do; security-through-obscurity on the path isn't a substitute for real auth hardening) or restrict by IP/VPN for higher-sensitivity sites.
- Stay current on Wagtail and CRX point releases — both ship periodic CVE fixes (permission-handling and XSS issues have both occurred in past Wagtail releases); pin versions deliberately and review release notes before upgrading, but don't let "deliberately" become "never."
