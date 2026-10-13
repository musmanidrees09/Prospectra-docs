# Technical SEO and Audit Experience Implementation Plan

## Decisions

- Canonical origin is `https://prospetra.com`.
- Retain the Pakistan accessibility service until real GSC, backlink/referrer, conversion, and service-availability evidence supports retirement.
- Keep terms and privacy indexable.
- Use only `frontend/public/robots.txt`; delete the generated robots route.
- Keep the durable database queue internal. `pending` means “Preparing”; the worker alone sets `analyzing`.
- Restore existing published blog content only. Do not invent posts or social profiles.
- Add no third-party packages.

## 1. Audit persistence and transactional delivery

Add additive migrations for audit idempotency, truthful processing stage, heartbeat, dispatch attempts/timestamps, stale-work indexes, and an `audit_dispatches` outbox table with one row per audit. Update `Audit`; add `AuditDispatch`.

Introduce `AuditSubmissionService` to lock the organization, replay the same tenant/user/idempotency key, enforce `PlanLimitService`, and create the audit plus outbox atomically. Introduce `AuditDispatchService` to claim due outbox rows, enqueue safely, retry transient failures with bounded backoff, and mark only still-pending audits failed after terminal dispatch failure.

Update `StoreAuditRequest`, `AuditController`, and `AdminAuditController` to use those services. Return pending resources promptly with “Preparing” messages. Preserve tenant authorization and retry quota rules.

Harden `AnalyzeWebsiteJob` and `WebsiteAuditPipeline`: atomically claim pending work; no-op terminal or duplicate delivery; set analyzing, start time, heartbeat, and truthful stages at real pipeline boundaries; guard terminal writes. Add scheduled `audits:dispatch-pending` and `audits:recover-stale` commands with bounded batches/dry-run, then schedule without overlap.

Files include:

- `backend/database/migrations/2026_10_13_000001_add_processing_reliability_to_audits_table.php`
- `backend/database/migrations/2026_10_13_000002_create_audit_dispatches_table.php`
- `backend/app/Models/Audit.php`, `AuditDispatch.php`
- `backend/app/Services/Audit/AuditSubmissionService.php`, `AuditDispatchService.php`
- `backend/app/Console/Commands/DispatchPendingAudits.php`, `RecoverStaleAudits.php`
- `backend/app/Http/Requests/Audit/StoreAuditRequest.php`
- `backend/app/Http/Controllers/Api/V1/AuditController.php`, `Admin/AdminAuditController.php`
- `backend/app/Jobs/AI/AnalyzeWebsiteJob.php`
- `backend/app/AI/Pipelines/WebsiteAuditPipeline.php`
- `backend/app/Http/Resources/AuditResource.php`
- `backend/routes/console.php`, `config/audit.php`, `config/queue.php`, `.env.example`, `DEPLOYMENT_RUNBOOK.md`

Tests: idempotent replay and concurrency, tenant isolation, quota once-only behavior, atomic outbox, transient/terminal dispatch failure, duplicate worker no-op, pending→analyzing claim, heartbeat/stage, stale recovery, retries, and terminal persistence.

## 2. Accessible audit UX

Generate one UUID idempotency key per deliberate frontend submission and reuse it for request retries. Navigate immediately to the detail page after acceptance. Replace audit queue wording with “Preparing your audit” and worker-start wording with “Analyzing the website.” Map only real backend stages; never show invented percentages.

Use `role="status"`, `aria-live="polite"`, stable focus, and visible text. Continue polling only for pending/analyzing.

Files:

- `frontend/src/services/audits.service.ts`
- `frontend/src/hooks/use-collections.ts`, `use-audit-detail.ts`
- `frontend/src/app/(dashboard)/audits/new/page.tsx`
- `frontend/src/app/(dashboard)/audits/[id]/page.tsx`
- `frontend/src/components/shared/status-badges.tsx`

## 3. Canonical host, robots, and indexing

Keep application fallback 301s in `frontend/next.config.mjs`, testing HTTP apex and HTTP/HTTPS www directly to HTTPS apex while preserving path/query. Document authoritative Cloudflare one-hop rules and staging access/noindex controls.

Change private auth/workspace/admin/utility metadata and headers from `noindex, nofollow` to `noindex, follow`; keep authorization unchanged. Keep terms/privacy canonical and indexable.

Create `frontend/public/robots.txt` with the apex sitemap and narrow API exclusions; delete `frontend/src/app/robots.ts`. Assert exactly one robots source, 200 output, and no internal www links.

Files include `frontend/next.config.mjs`, private route layouts, `frontend/public/robots.txt`, SEO/config tests, smoke scripts, and `frontend/docs/deployment/canonical-host-and-indexing.md`.

## 4. Sitemap

Update `public-pages.ts`, `sitemap.ts`, and `published-blog-sitemap.ts` so only canonical, indexable, successful public URLs appear. Use real curated modification dates. Add change frequency and priority only as descriptive metadata, explicitly documenting that they do not improve rankings.

Keep last-known-good cached published-blog inventory on temporary backend failure instead of silently deleting all blog entries; still exclude redirects, noindex, off-origin/query URLs, unsafe slugs, missing articles, and future publication dates.

## 5. Blog restoration and structured data

Add `frontend/src/lib/public-blog.ts` for the real API contract, strict slug/image/link validation, and explicit 404 versus temporary-failure behavior.

Restore `frontend/src/app/blog/page.tsx` and `[slug]/page.tsx` as server-rendered published content. A confirmed missing/unpublished post returns 404; timeout/5xx uses a temporary error path. Render stored content semantically as React values, not raw unsanitized HTML. Add visible breadcrumbs, one H1, self-canonical, matching article Open Graph metadata, and an error boundary.

Add BlogPosting/Article and BreadcrumbList JSON-LD using real author, publisher Organization, dates, canonical, and optional validated image. Extend backend public-blog publication/serialization/sanitization tests. Do not publish new content.

## 6. Internal links and Pakistan service

Keep Organization schema single-source and emit only verified configured `sameAs` profiles. Add restrained contextual links among existing services, free audit, website fix, and pricing. Improve the accessibility badge journey from explanation to free audit and then account creation after value is shown.

Retain `/services/accessibility-audit-pakistan`, its canonical, sitemap entry, and useful links. Review unsupported claims and CTAs. Record the future decision test: retain when offered/valuable; 301 only to a genuinely equivalent replacement; 410 only after permanent retirement with no equivalent.

Provide metadata, long-tail cluster, branded entity, internal-link, CRO, and measurement recommendations in the final SEO report. Update existing CMS metadata only when the published article actually supports it; create no placeholders.

## 7. Verification

Run fresh:

- backend: focused tests, `php artisan test`, `vendor/bin/pint --test`;
- frontend: `pnpm test`, `pnpm type-check`, `pnpm lint`, `pnpm build`, JSON-LD/SEO/indexing validators;
- responsive/browser checks for headings, focus, labels, status announcements, blog pages, and mobile overflow;
- cURL matrix for one-hop redirects, robots 200, sitemap XML, canonical/robots/schema, retained Pakistan 200, published blog 200, absent blog 404, and private-route headers.

Production deployment order:

1. back up the database;
2. deploy additive backend migrations/code;
3. configure audit thresholds and `DB_QUEUE_RETRY_AFTER` greater than job timeout;
4. supervise the low-concurrency `ai` worker and run scheduler every minute; restart workers after deployment;
5. deploy frontend with exact production origins;
6. install Cloudflare one-hop and staging protection rules, then purge affected cache;
7. run smoke/Rich Results checks, inspect representative Search Console URLs, and resubmit sitemap.

Rollback keeps additive migrations initially, disables outbox drain/recovery through config if needed, and restores application releases without deleting pending audit data. Cloudflare rules roll back independently while application fallbacks remain. Robots rollback must restore exactly one implementation. Never make valid blog content return a false permanent 404 during an outage.
