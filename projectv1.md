# MOTPO v1 — Codex Implementation Specification

**Project:** MOTPO.com  
**Tagline:** Your Anime Obsession.  
**Status:** V1 build specification  
**Stack:** Go, SQLite, server-rendered HTML, CSS, minimal JavaScript, Nginx, systemd  
**Repository:** Connect to the GitHub repository provided by the owner. Do not assume a repository name or push without authorization.

## 1. Mission

Build a fast, maintainable anime/manga discovery website with an integrated browser-based CMS. V1 has exactly three principal public sections:

1. `/adapted/` — searchable index of manga, manhwa, manhua and optionally light novels **already adapted into anime or with an officially confirmed anime adaptation**.
2. `/category/` — anime discovery by genre, mood, theme, trope, and story characteristics.
3. `/compare/` — human-reviewed manga/source-material vs anime comparison articles, including verified adaptation coverage where available.

A work can appear in multiple sections, all powered by one normalized catalog. Build foundations compatible with future full `/anime/` and `/manga/` directories, but **do not implement those directories, accounts, social features, or a general unadapted manga index in V1**.

### Definition of eligible adaptation

Include source works with an anime adaptation that is released, airing, or officially announced. Exclude rumors, unconfirmed leaks, and fan adaptations. Keep anime release status separate from source publication status and from how completely the source has been adapted. Only surface verified announcements as confirmed. The internal data model must allow unadapted works later, without publishing them in the V1 adaptation index.

## 2. Operating constraints and non-goals

- One deployable Go modular monolith; one SQLite database. No microservices, Redis, Elasticsearch, React SPA, or paid SaaS required.
- Server-rendered, mobile-first, accessible HTML with progressive enhancement; minimal JavaScript.
- Deployment compatible with Debian 13, Nginx reverse proxy, systemd, TLS terminated at Nginx/Cloudflare.
- Work comfortably on a modest shared VPS; avoid heavyweight recurring background tasks.
- V1 must be functional with **manual data entry alone**. External metadata APIs are optional adapters and must never block the basic site.
- Do not scrape copyrighted pages or bypass API limits. Do not assume any API permits persistent caching, republishing, cover-image reuse, or commercial use. Provide a documented provider policy configuration and keep unsupported providers disabled until verified.
- No AI-generated comparison claims published without review. No fabricated manga chapter/episode mappings, release dates, or announcement sources.
- Never perform destructive migrations, overwrite existing production data, or remove the old MOTPO museum application without explicit approval and a verified backup.

## 3. Routes and information architecture

| Route | V1 behavior |
|---|---|
| `/` | Homepage: search, three feature entrances, featured adaptations/categories/comparisons |
| `/adapted/` | Paginated adaptation index, search and filters (source type, adaptation status, year, genre) |
| `/adapted/{slug}/` | Adaptation overview with original source, related anime installments, statuses, source citations, and comparison link if published |
| `/category/` | Browse genre/mood/theme/trope taxonomy |
| `/category/{slug}/` | Paginated eligible anime/adaptation results with clear filtering |
| `/compare/` | Published comparison listing |
| `/compare/{slug}/` | Spoiler-labeled, evidence-backed comparison article |
| `/search/?q=...` | Unified title/alternate-title search, with paginated results |
| `/admin/login` | Protected login |
| `/admin/` | CMS dashboard |
| `/admin/works` | Work catalog CRUD, bulk operations, duplicate review |
| `/admin/adaptations` | Manage source-to-anime relationships and announcements |
| `/admin/tags` | Category CRUD and assignment |
| `/admin/comparisons` | Comparison editor and publication workflow |
| `/admin/imports` | Provider configuration, job status, errors and review queue |
| `/admin/trash` | Restore or safely purge deleted content |

Reserve `/anime/`, `/manga/`, `/novel/`, `/people/`, and `/user/` as future namespaces; do not publish empty indexable pages. Use stable canonical URLs and 301 redirects when slugs change. Add sitemap.xml and robots.txt. Do not expose drafts, trashed records, private admin content, or empty placeholders in sitemap.

## 4. Data architecture

Use internal stable integer IDs, foreign keys, migrations, explicit constraints, indexes, and UTC timestamps. Avoid using third-party IDs as primary keys. Design `works` for distinct catalog entities and installments; do not put multiple seasons/remakes in a single free-text field.

### Required tables

- `works`: `id`, `work_type` (`anime`, `manga`, `manhwa`, `manhua`, `light_novel`, `visual_novel`, `other`), `canonical_title`, `slug`, `synopsis`, `visibility` (`draft`, `published`, `archived`, `trashed`), timestamps. Do not overload this with a universal status.
- `work_titles`: alternate titles, language, script/title type, normalized search representation; unique constraints where sensible.
- `anime_details`: `work_id` unique FK, format (TV/movie/OVA/ONA/etc.), episode count, release/airing status, first/last air dates, optional studio data.
- `publication_details`: `work_id` unique FK, source format, publication status, chapter/volume counts, publication dates.
- `work_relations`: source work FK, target work FK, typed relation (`adaptation`, `sequel`, `prequel`, `remake`, `spinoff`, `other`), verification state, timestamps, no self-relations. Define directional semantics clearly (source → adaptation for `adaptation`).
- `adaptation_announcements`: relationship FK, announcement date, official URL, confirmation state, verification timestamp, notes.
- `tags`: name, slug, type (`genre`, `mood`, `theme`, `trope`), description, visibility.
- `work_tags`: work FK, tag FK, attribution/provenance, optionally confidence and manual override.
- `comparisons`: source work FK, anime work FK (or relationship FK), unique slug, title, spoiler level, body in sanitized Markdown, editorial state, timestamps, publication date.
- `adaptation_coverage`: relationship FK, anime installment/episode range, source chapter/volume range, verification state, source evidence; ranges may be unknown/null. Do not infer mappings from anime season number.
- `external_ids`: work FK, provider, provider ID, sync metadata; unique (`provider`, `external_id`).
- `source_evidence`: linked entity kind/id, URL, source title, evidence type, verification date and notes. Prefer explicit typed relationships or validate polymorphic references at the application layer.
- `media_assets`: linked work, URL/path, image attribution/license/rights notes, status. Do not hotlink or download unlicensed art by default.
- `slug_redirects`: old path, new path, unique old path, timestamps.
- `admin_users`, `admin_sessions`: protected local admin auth and revocable sessions.
- `content_revisions`, `audit_log`, `import_jobs`, `import_candidates`, `provider_cache`: recoverability, accountability, staged import and policy-limited cache.

Implement migrations as numbered SQL files with a schema version table. Enable `PRAGMA foreign_keys=ON` per connection and WAL mode; configure busy timeout and a bounded connection pool. Add appropriate indexes, especially relationships in both directions, work type/visibility, tags, publication state, and unique provider identifiers. Use SQLite FTS5 for title/alternate-title search with a documented reindex strategy; fallback gracefully if FTS5 unavailable.

### Important modeling invariants

- A manga may relate to many anime installments, movies, OVAs, and remakes; an anime may have multiple related source works.
- Manga publication status, anime release status, adaptation announcement state, and coverage completeness are separate fields.
- An adaptation relationship is not proof that every chapter has been adapted.
- Store external provider provenance and rights/retention metadata. Separate temporary API cache from legally retained catalog records.
- Imported values must never silently overwrite manually locked/corrected fields.
- Keep editorial comparisons separate from API metadata so they survive provider changes.
- Preserve stable IDs and redirects when titles/slugs change; deduplicate cautiously and support admin-reviewed merges.

## 5. Public features — acceptance criteria

### `/adapted/`

- Shows only published, eligible source works with at least one verified released/airing/officially announced anime relationship.
- Search works with canonical and alternate titles; filters can combine; paginated URLs are shareable.
- Detail page clearly identifies original source format, original work publication status, each anime installment, release/announcement status, and evidence for planned adaptations.
- Link to published comparisons; no fake chapter data. If coverage is unknown, show “Not yet verified”.
- For multiple adaptations, show each installment separately and link their relationships correctly.

### `/category/`

- Separate genre, mood, theme, and trope categories, each with an editable description.
- Category pages list eligible anime titles with server-side pagination, relevant linked works, and meaningful empty states.
- Support multiple tags per work and basic combined filters; manually curated tags should remain after API sync.

### `/compare/`

- Editor supports Markdown with safe rendering, spoiler warnings, sources, drafts, previews, publish/unpublish.
- Comparisons can describe skipped/changed scenes and verified coverage, but absent evidence show unknown rather than inventing specifics.
- Only published comparisons have public URLs or appear in sitemap.

### Search, performance, and accessibility

- Server-side search with pagination and normalized alternate-title matching.
- Semantic headings, keyboard navigation, visible focus, labels, alt text where appropriate, readable mobile layout.
- Canonical URLs, sensible metadata, Open Graph tags, sitemap, robots, structured data only where accurate.
- Cache public responses conservatively; invalidate relevant pages after admin edits. Avoid cache leakage from `/admin/`.

## 6. Admin CMS — core V1 requirement

**All normal content maintenance must work in the browser without editing code or SQL.**

- Dashboard counts: works, eligible adaptations, published pages, pending reviews, import errors.
- Create/edit/delete/archive/restore works, anime details, source details, relationships, announcements, categories/tags, and comparisons.
- Search/filter/sort tables, paginate, select multiple records; bulk tag, publish/unpublish, archive and move to trash.
- Show relationship dependencies and affected public pages before delete/merge. Prefer soft deletion; allow recovery from trash.
- Support permanent purge with typed confirmation and a fresh verified backup; **never cascade-delete unrelated source works by default**.
- Support a special **Delete All Imported Data** operation only as an explicitly gated maintenance action: preview exact counts, distinguish imported records from manually authored records, block if dependencies are unresolved, create/verify backup, require typed phrase, and record audit log. Do not put this beside ordinary Delete buttons.
- Provide a comparison editor with draft, preview, publish, revisions, and spoiler level.
- Maintain provenance/field locks; provide a diff/review screen before applying API updates that conflict with manual edits.
- Record meaningful audit events (create, update, merge, delete, restore, import, login) and allow restoring prior revisions.
- Single admin account is sufficient for V1; structure authorization middleware so future roles can be added.

### Admin security

- Hash passwords with Argon2id or bcrypt, secure session cookies (`HttpOnly`, `Secure` under HTTPS, appropriate `SameSite`), session expiration and logout.
- CSRF protection for all state-changing browser requests, parameterized SQL, output escaping, sanitized Markdown, strict file validation if uploads are implemented.
- Rate-limit login, do not expose provider secrets, protect admin routes and disable indexing.
- No default hardcoded credentials. Bootstrap admin via secure CLI/env setup, not a public install endpoint.
- Require POST/DELETE-style mutations rather than destructive GET URLs. Maintain audit logs and backup before high-impact operations.

## 7. Imports and API strategy

Implement a provider interface, not a hard-coded dependency:

```go
type MetadataProvider interface {
    Search(ctx context.Context, query string) ([]Candidate, error)
    GetWork(ctx context.Context, externalID string) (*Candidate, error)
    GetRelations(ctx context.Context, externalID string) ([]Relation, error)
}
```

- A working manual-entry workflow is mandatory even with no API keys.
- Start with an interface, a disabled/mock provider for tests, and import candidate review screens.
- Do not automatically enable AniList, Jikan, Kitsu, or other providers for persistent commercial imports until usage, licensing, caching, attribution, rate limits, and image rights are checked and documented.
- Search local catalog first. External lookup only if enabled and allowed. Rate-limit, deduplicate, log jobs, retry transient errors with backoff, and expire temporary cache per provider policy.
- Imported candidate must pass eligibility checks and verification before public publication. Unverified announcements remain drafts/pending.
- Provider outages must not take down existing public catalog pages.
- Do not silently generate pages from every API search; stage and review candidates.

## 8. Suggested repository layout

```text
motpo/
  cmd/motpo/main.go
  cmd/motpoctl/main.go             # admin bootstrap / maintenance commands
  internal/
    config/
    database/
    catalog/
    adaptations/
    discovery/
    comparisons/
    search/
    providers/
    importer/
    admin/
    auth/
    web/
  migrations/
  templates/
    layouts/
    pages/
    components/
    admin/
  static/css/
  static/js/
  static/images/
  tests/
  deploy/
    motpo.service.example
    nginx.conf.example
  docs/
    architecture.md
    data-sources.md
    operations.md
  .env.example
  go.mod
  README.md
```

Use idiomatic Go and standard library where reasonable; choose a small, maintained router and SQLite driver if helpful. Keep handlers thin, put business logic in services, and keep SQL isolated in repositories. Do not introduce frameworks without a clear need.

## 9. Deployment, backups, operations

- Read config from environment; `.env.example` has placeholders only. Never commit secrets, SQLite files, backups, API keys, or media without rights.
- Run app as a dedicated non-root service user; bind to `127.0.0.1` behind Nginx.
- Provide example systemd unit with hardening, restart policy, health endpoint, and graceful shutdown.
- SQLite database on persistent storage outside release directory; use safe SQLite online backup API or `VACUUM INTO` / tested consistent backup approach, not unsafe raw copies during writes.
- Document backup schedule, retention, restore rehearsal, log inspection, migrations, rollback approach, and release steps.
- Never deploy to production or modify existing MOTPO service without explicit owner approval.

## 10. Implementation milestones for Codex

Build incrementally; after each milestone, run tests and report changed files, commands, results, and blockers.

**M0 — Repository audit and plan:** Inspect repository, existing code, and deployment constraints. Identify anything worth preserving. Present proposed file tree, migrations, dependencies, and risks. Do not delete or overwrite existing content.

**M1 — Foundation:** Go app, configuration, SQLite migrations, core models, basic routing, templates, CSS, tests, health endpoint, README. Seed a few clearly labeled sample records in development only.

**M2 — Catalog and public pages:** `works`, titles, relationships, adaptation eligibility query, `/adapted/` list/detail, search and pagination.

**M3 — Discovery and comparisons:** taxonomy, `/category/`, `/compare/`, comparison drafts/publishing, cross-links, SEO basics.

**M4 — Admin CMS:** authentication, CRUD, validation, revisions, trash/restore, safe bulk actions, dependency-aware delete/merge, audit log.

**M5 — Import foundation:** provider interfaces, policy settings, staged candidates, deduplication, review queue, import logs; no unlicensed mass import.

**M6 — Production readiness:** security review, migration tests, backup/restore test, performance sanity checks, accessibility review, systemd/Nginx examples, operational docs.

Keep commits focused by milestone. Do not declare a milestone complete if tests fail or required features are stubs.

## 11. Tests and definition of done

Include automated tests for:

- Migration up on a clean database; data preservation on subsequent runs.
- Alternate-title search, pagination, filters, and stable URLs.
- One manga linked to multiple anime installments, and multiple related source works.
- Only officially confirmed or released adaptations appear in public V1 index; rumors do not.
- Separate publication, airing, and coverage statuses.
- Draft/archived/trashed records are not publicly accessible or indexed.
- CRUD, bulk operations, trash/restore, revisions, and relationship-safe deletion.
- Manual field locks survive import updates; duplicate candidate detection.
- CSRF, authentication, authorization, safe HTML/Markdown rendering, login throttling.
- Backup and restore against a populated test database.
- API disabled/unavailable behavior does not break the site.

**V1 is done when** an administrator can log in, create a source manga and linked anime, mark the adaptation as released or verify its official announcement, assign moods/categories, publish the adaptation page, write and publish a comparison, find it through public search, edit it, unpublish it, move it to trash and restore it — all without touching source code or the SQLite CLI.

## 12. Immediate instruction to Codex

1. Read this specification and inspect the connected GitHub repository.
2. Report whether the repository is empty or already contains an application; do not assume it is safe to overwrite.
3. Propose the exact M0/M1 implementation plan and initial schema with any ambiguities or license dependencies.
4. Begin implementation in small tested commits once the owner approves the plan.
5. Preserve the three V1 public sections and future-proof catalog model; do not expand scope into user accounts, full anime/manga indexes, or unverified automated content generation.
