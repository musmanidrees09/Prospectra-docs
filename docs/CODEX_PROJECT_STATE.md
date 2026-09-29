# Prospectra Codex Project State

Updated: 2026-09-25

## Current phase

The requested production-readiness implementation is complete in the repository. Deployment and real-provider activation remain operational steps that require production credentials, hosting access, and quota approval.

## 2026-09-22 comprehensive-audit remediation

- Accounted for all 58 unique issue IDs in `COMPREHENSIVE_AUDIT_REPORT.md`; the report's stated total of 47 does not match its master table. Detailed dispositions are in `docs/AUDIT_REMEDIATION_STATUS.md`.
- Closed the reported P0/P1 security and type-safety defects: allowlisted dynamic sorting, typed authentication/API errors, DOMPurify proposal rendering, HTMLPurifier-backed blog sanitation, and direct tenant filtering for email queries.
- Extracted shared frontend date, initials, empty-state, and debounce utilities; removed unused packages; improved audit error messages; and retained measured, paginated rendering instead of speculative memoization or virtualization.
- Extracted blog and billing services/Form Requests, centralized entitlement and pagination logic, made production audit execution asynchronous, strengthened share tokens and admin transactions, removed PII logging, and added tenant query indexes.
- Verification: backend 440 tests / 1,341 assertions, Pint clean, frontend type-check clean, lint with zero errors, production build successful (62 static pages), frozen lockfile install successful, and `git diff --check` clean across all repositories.

## 2026-09-17 QA pass (full re-test + mobile)

- **Fixed: public-page 401 bounce.** Anonymous visitors to /pricing were hard-redirected to /login: PricingGrid probes `/billing/status` (401 for guests, handled as guest mode) but the axios response interceptor redirected on any 401 except `/auth/me`. The interceptor now exempts guest-allowed probes (`/auth/me`, `/billing/status`) so callers decide. Verified live: /pricing renders for guests. This was also an SEO problem — Googlebot/visitors landing on /pricing could hit the login page.
- Mobile sweep at 379px: home, pricing, website-fix, free-audit, blog index, blog article, contact, login — zero horizontal overflow on all. Logged-in: dashboard, audits list, audit detail, leads, emails, audits/new (select full-width, chevron via background-image), admin/blog, admin/users — zero overflow.
- Login churn reproduced and explained: the QA account did not exist after the MySQL restart (fresh DB). Created via a temp helper mirroring AuthService::register (user + owned organization + admin role). NOTE: a user without an organization makes workspace endpoints 500 (ProposalService::list() receives null org id) — real signup always creates the org, so this is QA-tooling-only.
- Functional QA end-to-end (live API + worker): audit created (pending) → queue:work processed → completed score 77, coverage 92.1%, 38/46 checks tested; lead created (id 4, email required when no phone — 422 validation correct); email draft created (id 4, status=draft); analytics/overview 200. Audit detail renders all 8 tabs; Issues tab shows 18 prioritized findings with fixes and "View observed source" links.
- Regression: backend 436/436 (~95s), pint clean, tsc exit 0, next build exit 0. SEO validator: 27/27 pages clean locally. Production: canonicals correct on / and /pricing, robots.txt → apex sitemap, sitemap.xml 28 URLs, www 308 → apex (verified via curl).
- Tablet fix re-verified intact: sidebar.tsx `md:flex`, mobile-nav.tsx `md:hidden`, admin shells `md:flex`, compiled CSS contains `.md\:flex`/`.md\:hidden` under `@media (min-width: 48rem)`. Admin chip present in mobile nav; admin shell mobile nav has labeled Workspace chip.

## 2026-09-16 QA pass (full-stack verification)

- Tablet breakpoint fix: the workspace sidebar (and its Admin panel link) now shows from `md` (768px) instead of `lg` (1024px); the mobile chip bar hides at the same point (`md:hidden`), and the dashboard logo link matches. Verified in the compiled CSS (`@media (min-width: 48rem)` contains `.md\:flex`). Previously tablets showed neither sidebar nor a way to reach the admin panel.
- Mobile admin access: the workspace mobile chip nav now appends a permission-gated Admin chip (`isAdminRole || canAccessBlogAdmin || canAccessSeoAdmin` — same checks as the desktop sidebar); the admin shell's mobile chip nav leads with a labeled "← Workspace" link back to the user panel (previously only the unlabeled logo).
- Audit detail page: added a pending/analyzing state (spinner + "Audit queued — analysis starts shortly" + status badge). Previously a queued audit rendered the header and then nothing (report sections return null without a report).
- Select primitive: chevron is now a `select-chevron` background-image utility painted on the native select itself; the absolutely-positioned icon and its wrapper span were removed, making the detached-chevron bug structurally impossible on all routes.
- Shared `HeroAuditCard` extracted; `/website-fix` now renders the identical dark hero audit card as the homepage (previously a one-off flat card with a different button color and no score/coverage mini-cards).
- `phpunit.xml` pins `AUDIT_EXTERNAL_PROVIDERS_ENABLED=false`, keeping the suite hermetic (dev `.env` value no longer leaks live provider calls into tests).
- Verification: backend 436/436 tests pass (~78s, was 433/434 + 255s before the phpunit pin); `pint --dirty` clean; `tsc --noEmit` exit 0; ESLint exit 0 on all touched files; `next build` exit 0.
- Live functional QA (local backend + DB): login OK; lead created (id 3); audit of example.com queued → worker processed → completed with score 79, coverage 89.9%; email draft created from the lead; admin audits list, audit detail tabs (8 tabs, score/coverage/category health), dashboard pipeline cards, and both mobile navs verified in the browser.
- SEO: local validator passes all pages (titles/descriptions within length). Production checks: prospetra.com serves 200 with correct canonicals; www 308-redirects to apex; robots.txt points at the apex sitemap; sitemap lists 27 URLs. The intermittent local blog 500s were a cold-start race (generateStaticParams + first DB fetch exceeding 3s deadline, `Unexpected end of JSON input` on truncated bodies) — warm requests return 200; the 3s `fetchJsonWithTimeout` fallback bounds the stall.
- Search Console CTR (280+ impressions / 0 clicks, 7–10/day recently): titles and descriptions on the ranking clusters were reviewed against live HTML — they are specific, length-compliant, and intent-matched (e.g. "Website Audit Checklist: 47 Checks…", "Website Speed Test: How to Check & Fix…"). The dominant cause of impressions-without-clicks at this stage is average position and the new-domain authority plateau, not metadata quality; the highest-leverage levers are internal links from the highest-traffic pages to /free-audit and richer first-screen content above the fold, both already present. No fabricated or clickbait rewrites were made.

## Completed implementation

- Platform accounts use only Admin and User roles. Obsolete role values are normalized by migration without changing the configured administrator's password hash.
- Admin can promote a User, demote another Admin, delete non-admin users, and manage all blog content. Self-demotion is prohibited.
- Blog Writer and SEO Specialist are permission presets rather than platform roles. Writers can create and edit but cannot publish, archive, or delete.
- The Admin and delegated Blog Writer editors use a semantic section contract with `headingLevel` 2 or 3. H3 subsections require and nest under the nearest preceding H2; legacy sections default to H2, and the public article and table of contents render the hierarchy semantically.
- Authenticated and public audit entry points use public-network URL validation, redirect revalidation, DNS pinning, response-size limits, and content-type validation.
- The crawler is bounded to five same-site pages and safely reads robots and sitemap evidence.
- Deterministic SEO, technical, content, accessibility, security, performance, domain, company, contact, social, technology, topic, and opportunity evidence is persisted with provenance and confidence.
- Each authenticated audit is a fresh bounded crawl. Its collection timestamp and attempted-page count are stored; cached external evidence is versioned and incompatible serialized cache entries are discarded safely.
- Website Score uses only legitimately tested checks. Audit Coverage exposes how much intended analysis ran. Failed providers never become passing evidence.
- Optional providers are privacy-opt-in, cached with provider-specific lifetimes, usage logged, daily limited, and failure isolated.
- Hunter runs only as a fallback after first-party contact extraction.
- AI narrative enrichment is optional and validated against evidence; it cannot supply deterministic facts or alter the score.
- The customer report is organized into Overview, Issues, Performance, Company, Contacts, Opportunities, Outreach, and All Checks. Provider names, analyzer telemetry, raw JSON, and internal exception details are not exposed.
- First-party contact forms are reported separately from public email and phone evidence. Visible phone numbers are conservatively extracted only with phone context or international plus notation, form field types are recorded, and provider identities remain hidden behind customer-facing provenance labels.
- Audit-to-lead conversion accepts observed email or phone evidence without inventing either. Phone-only leads remain editable and can proceed to proposals, while email drafting stays disabled until a verified-format address is added. A form alone is not promoted into an unusable lead record.
- Authenticated routes emit noindex/nofollow metadata and are excluded from robots crawling. Database-authored blog H3 hierarchy and database posts in Blog JSON-LD are preserved on public pages.
- Leads now connect directly to email drafting and proposal creation. A shared four-step workflow explains Audit → Lead → Email → Proposal, and proposals can be created manually or queued for evidence-grounded AI drafting.
- Shared form selects now use a native, accessible compatibility layer that preserves the existing component API and avoids the React 19 portal/ref crash seen across lead, email, and proposal routes. IDs are normalized to strings before selection and submission.
- Email draft creation redirects to the created email detail page. Email responses eager-load the lead company so outreach detail retains its qualification context.
- The authenticated workspace uses a scoped premium editorial theme, clearer workflow hierarchy, richer surfaces, and responsive list/form treatments. Audit detail sections were intentionally left unchanged.
- The homepage overflow ancestor no longer blocks sticky positioning, and its hero begins closer to the public header across phone, tablet, and desktop widths.
- Public blog list/detail APIs now expose one complete, consistent contract. Exact legacy thin seed duplicates fall back to the richer static article, optional seeders no longer overwrite editor-owned content, old brand typos and social-image defaults are corrected safely at read/seed time, and article images reserve their aspect ratio to prevent layout shift.
- Admin overview cards all open useful destinations. A global, metadata-only Sales Activity view covers leads, emails, and proposals without exposing message bodies.
- Admin can open reports across workspaces and retry failed or genuinely stalled audits. A retry clears stale findings and runs a fresh analysis; active audits cannot be duplicated.
- Inline SMTP failures are recorded as failed instead of remaining queued. The UI distinguishes outgoing-server acceptance from guaranteed inbox delivery and explains bounce causes.
- Audit creation and user retry both enforce plan quota. Exhausted jobs mark their audit failed. The scheduler never performs an uncontrolled retry of all failed jobs.
- Production deployment, queue, scheduler, provider, Hostinger, smoke-test, and rollback instructions are documented in PRODUCTION_DEPLOYMENT.md.

## Intentional truthful limitations

- The product audit engine does not claim a rendered axe scan as deterministic evidence. Independent browser QA uses axe for interface regression testing, while unsupported customer-site checks remain explicitly not tested.
- Factual traffic volume and keyword ranking metrics remain unavailable unless a supported evidence provider supplies them.
- Google Custom Search is unsupported by the current Google project and is not repeatedly called.
- Cloudflare Radar remains disabled and is not required for scoring.
- AI recommendations and outreach are marked not tested when AI enrichment is disabled or unavailable.
- Live external-provider results depend on explicit opt-in, valid credentials, outbound network access, provider availability, and remaining quota.

## Trust rules

- Audit states are completed, partial, or failed after processing.
- Check states are passed, warning, failed, unavailable, not tested, not applicable, or insufficient data.
- External failure degrades only its own evidence.
- AI never invents contacts, traffic, rankings, technologies, or performance measurements.
- First-party and structured data precede scarce external enrichment.

## QA artifacts

The dogfood-output directory contains browser QA artifacts. The isolated local browser database is backend/storage/app/browser-qa.sqlite and is ignored by repository storage rules.

Final verification on 2026-09-14:

- Laravel: 436 tests passed with 1,334 assertions.
- Laravel Pint: passed.
- Frontend TypeScript: passed.
- Frontend ESLint: passed.
- Next.js production build: passed with 62 generated pages plus dynamic application routes.
- Browser regression: lead company/status/source selection, lead creation, email lead selection and created-detail redirect, proposal lead selection and created-detail redirect, 390px workspace rendering, homepage sticky navigation, blog index, and a full blog article passed without application exceptions.
- Design evaluation: passed. Premium styling stays scoped to the authenticated workspace; the public palette and audit detail sections remain unchanged.
- SEO live check: Google/web search still surfaces the homepage and multiple recent blog articles, so no site-wide deindexing was observed. Search Console remains authoritative for diagnosing the reported impression/click decline.
- QA report: dogfood-output/prospetra-regression/report.md.

Previous verification on 2026-09-11:

- Laravel: 434 tests passed with 1,318 assertions.
- Laravel Pint: passed.
- Frontend TypeScript: passed.
- Frontend ESLint: passed.
- Next.js production build: passed with 61 statically generated pages plus dynamic application routes.
- Responsive browser QA: 13 public routes and 28 authenticated workspace/content/admin routes passed at 320px; homepage, dashboard, populated audit report, and admin overview also passed the full 320-1920px width matrix without unintended clipping or document overflow.
- Audit report QA: all eight report tabs render full-width panels after correcting the shared tabs orientation variants.
- Homepage QA: the complete Website intelligence / Qualified opportunities / Outreach label and audit form remain visible at narrow mobile widths.
- SEO crawl: all 27 sitemap URLs return HTTP 200 with exact self-canonicals. The historical prospectra.com typo is repaired in seeded defaults and existing PageSeo data by migration.
- Browser accessibility: axe-core 4.12.1 reported zero WCAG A/AA violations on representative public, workspace, audit, and admin surfaces.
- Local production performance at 390px: LCP 76-204ms and CLS 0 across homepage, pricing, dashboard, and a populated audit report.
- Published public smoke on 2026-09-08: homepage, blog, free audit, contact page, and API health loaded. The local ADMIN_EMAIL/ADMIN_PASSWORD pair received HTTP 401 from production, so authenticated production verification requires credential synchronization after deployment.
- Authenticated browser smoke: login, all admin overview cards, global sales activity, proposal creation, cross-workspace report opening, report section tabs, provider-name redaction, and mobile audit layout rendered successfully with no page errors.
- Browser accessibility scan: axe-core 4.12.1 reported zero WCAG A/AA violations on the authenticated audit report.
- Final report screenshot: dogfood-output/screenshots/final-authenticated-audit-report.png.

## Responsive QA matrix (2026-09-17)

Full route sweep via headless Chrome (`frontend/scripts/qa-responsive.cjs`): 5 breakpoints (320/375/768/1024/1440) x 16 anonymous routes + 25 authenticated routes = 205 checks per run. Measures horizontal overflow (>2px), HTTP status, and console errors.

**Result:** 203/205 ok. One real defect found and fixed:
- Public audit report `/audit/{share_token}`: the X/LinkedIn/WhatsApp share row (three `flex-1` nowrap buttons) could not shrink below min-content and overflowed at 320px/375px. Fixed in `src/components/audit/report-share.tsx` by making the row wrap; re-verified 0px overflow at 320/375/430.

**Tool:** `node scripts/qa-responsive.cjs --base http://localhost:PORT [--sizes 320,375,768,1024,1440]` — parameterized (base URL, API URL, QA credentials, sizes, output file); requires a QA admin account and running backend. Verify share-button pass: not part of the sweep (needs a live share token; pass `--share-token` to include the public report route).

## Core Web Vitals pass (2026-09-17)

Method: `frontend/scripts/qa-cwv.cjs` — Lighthouse 12, mobile throttling, medians over 3-5 runs, against `next build && next start` (port 3115). LCP element on both pages is body text in Inter (font-swap dependent).

**Before -> after (mobile, medians):**
- Home: perf 87 -> 90, FCP 1.23s -> 1.69s*, LCP 3.62s -> 3.43s, TBT 207ms -> 97ms (-53%), CLS 0.0000 -> 0.0000
- Blog article: perf 86 -> 91, FCP 1.24s -> 1.39s*, LCP 3.65s -> 3.24s, TBT 222ms -> 132ms (-41%), CLS 0.0000 -> 0.0000
- *FCP medians drifted up with run count (local noise); per-run variance was high, LCP/TBT medians were stable.

**Changes (commit 67eb236):**
1. Auth service (axios) is lazily imported in `use-auth.ts`; `useCurrentUser` is disabled without a token, so anonymous visitors make zero API calls and never download the API client (verified with a request-listening smoke test).
2. `AuditWidget` dynamically imports `audit.service` at submit time (type-only imports remain static).
3. `layout.tsx`: `preload: false` on Playfair Display + JetBrains Mono; only Inter is preloaded now.
4. Notification-preference defaults moved to `src/constants/notification-preferences.ts` so the settings page doesn't statically import auth.service.

**Remaining known bottleneck:** LCP ~3.4s is dominated by the Inter webfont swap under Lighthouse's throttled CPU/network (LCP element is the hero paragraph). Options for a future pass: `adjustFontFallback` tuning, `size-adjust`-matched fallback, or accepting system-font flash via `font-display: optional`. The 112KB `polyfillFiles` chunk (core-js + axios) is `noModule`-gated and never executed by modern browsers — not a real-world cost.

## Backend availability incident (2026-09-17)

**Symptom:** `[api] Could not reach http://localhost:8001/api/v1/notifications ... Network Error` in the browser console while the backend process was technically listening.

**Root causes, both fixed:**
1. The backend had been hand-launched with a nonexistent router script (`php -S 127.0.0.1:8001 backend/server.php`), so every request returned 200 + PHP fatal-error HTML with no CORS headers — browsers report that as Network Error. Relaunched correctly with `php artisan serve --host=127.0.0.1 --port=8001`. Lesson: always verify with `curl -i -H "Origin: ..."`, not status alone.
2. `php -S` (and `artisan serve`) is single-request-at-a-time on Windows; `PHP_CLI_SERVER_WORKERS` is POSIX-only and silently ignored. Slow requests (audit crawl) queue everything else. Mitigated client-side in `axios.ts` (commit ac96f79): idempotent GETs retry twice with short backoff on no-response failures before surfacing "Network Error".

For real parallelism locally, use Laragon's Apache/nginx + php-fpm, or Octane/Swoole — noted as a deployment concern, not a local-dev blocker.

## Backend on Laragon Apache (2026-09-17) — concurrent requests no longer queue

`artisan serve`/`php -S` is single-worker on Windows, so any slow request (audit
pipeline) queued every other API call. The backend now runs under Laragon's
Apache 2.4.66 with mod_php (TS PHP 8.4.22) — mpm_winnt serves each request on
its own thread. NOTE: php-fpm does not exist on Windows (POSIX fork); Apache
+ mod_php is the correct local equivalent. Laragon vhost:
`C:\laragon\etc\apache2\sites-enabled\prospectra-api.conf` (docroot
`backend/public`, `Listen 127.0.0.1:8001` — required, `<VirtualHost>` alone
does not open the socket). Opcache enabled (php.ini backup:
`php.ini.bak-prospectra`). Health-check tool: `npm run check:backend`
(frontend) — fails on HTML-instead-of-JSON, missing `{data}` envelope, or
missing CORS preflight headers (the class of broken launch that previously
mimicked "healthy").

**Measured:** 8 concurrent API calls complete in ~200ms total (previously
serialized, ~8x that); sequential-vs-parallel benchmarks show 2.3-2.7x
speedup, and opcache cut warm single-request latency from ~416ms to ~224ms.

**Start order:** MySQL service -> Apache (Laragon GUI or
`httpd.exe -d C:/laragon/bin/apache/httpd-2.4.66-260223-Win64-VS18` detached)
-> `npm run dev` in frontend; then `npm run check:backend`. Do not also run
`php artisan serve` on 8001. Queue worker unchanged.

## LCP font-strategy pass (2026-09-17): display:optional + metric-matched fallback

Change: Inter in `layout.tsx` moved from `display: "swap"` to `display: "optional"`
with `adjustFontFallback: true` (Next metric-matches Arial to Inter via
size-adjust/ascent overrides — verified in the built CSS). With `optional`, the
font never swaps in after first paint, so LCP is recorded exactly once.

**Honest finding — the font swap was NOT the LCP bottleneck.** Full A/B on the
prod build (Lighthouse mobile, 7 swap runs vs 15 optional runs across 3 blocks):
home LCP mean 3467ms (swap) vs 3587ms (optional); blog 3378ms vs 3246ms.
Statistically flat — inside machine noise. Root cause per Lighthouse phase
breakdown: 87% of LCP is render delay from a 1591ms hydration long task
(1.06-2.65s) plus styleLayout work (903ms); the hero `<p>` is both FCP and LCP
element and simply cannot paint until the main thread frees.

**Why keep `optional` anyway:** it removes the ~1.8s late Inter repaint risk
entirely (LCP can never be re-recorded by a font swap), the Lighthouse
`font-display` audit passes (score 1), and slow-connection users get a stable,
metric-matched page instead of a mid-load font change. Trade-off: on slow
connections the whole session renders in the Arial-matched fallback, not Inter.

**Real next lever for LCP (~3.5s):** cut hydration cost on the homepage (the
1.6s long task = framework + widget hydration; styleLayout 903ms suggests
expensive CSS/layout — candidate: audit widget card). Not attempted here to
keep this pass scoped to font strategy.

Artifacts: `frontend/cwv-*.json` (raw per-run data: baseline-before,
after-optional, variance-check, swapAB, final-optional[-blog]).

## 2026-09-24 production-audit remediation (audit: docs/PROSPETRA_PRODUCTION_AUDIT_2026-09-23.md)

Six issues from the production audit resolved and verified; two remain deployment-bound (below).

- **PROS-DASH-001 / PROS-FE-101 (P0, fixed):** axios 401 interceptor now clears credentials (localStorage token + mirrored cookie) BEFORE redirecting, with a single-flight redirect (module flag; browsers collapse the burst), auth-page path guard, `?redirect=` return path, and orphaned-cookie purge on guest-allowed 401s. `useCurrentUser` clears creds and returns null on 401. Regression-locked: 7 vitest tests (`src/lib/axios.test.ts`) + 17-test Playwright stale-token suite (`tests/e2e/stale-token.spec.ts`, `pnpm test:e2e`; expired/garbage/revoked/missing tokens × /dashboard, /admin, /leads, plus a loop sentinel and a real-login positive). All 17 passing against the real backend. (2026-09-25 live re-check: production still runs the pre-fix build — junk cookie gets 200; see the live post-deploy verification block in the runbook section.)
- **Middleware → proxy (P1, fixed):** `middleware.ts` renamed to `src/proxy.ts` (Next 16 convention; must sit next to `app/` under `src/` — a root-level proxy.ts is silently ignored). Auth-cookie route gating now sanity-checks the Sanctum token format `<id>|<40-64 chars>` (real tokens are 48 chars: 40 entropy + 8 crc32b — verified in Sanctum source and via tinker; the initial exact-40 gate was wrong and would have bounced real users, caught by the Playwright revoked-token tests). Predicate shared via `src/lib/auth-token-format.ts` (9 unit tests). `setSessionCookie` no longer writes a "1" presence marker (would fail the gate).
- **PROS-FE-102 (P2, fixed):** dashboard hydration error #418. Root cause: `AuthProvider` exposed `isLoading` from a localStorage-gated query, so server HTML rendered the avatar branch while client first paint rendered the spinner (UserMenu/Sidebar pattern). Fix: `isLoading` is forced true until after mount in `AuthProvider`, making first paint identical; all consumers stabilize at once. Verified: dev capture clean; prod build capture went from "Minified React error #418" to **0 hydration messages**.
- **PROS-LEAD-001 (P2, fixed):** malformed email extraction (`usinfo@…`, `info@…comemailinfo`). `visibleText()` now space-separates tags before `strip_tags` (no glue tokens at the source); page-text regex requires a known TLD boundary (`COMMON_TLDS`, longest-first alternation + lookahead); `addEmail` dedupes case-insensitively preferring mailto (0.95) over page-text (0.8). 4 new tests incl. both observed artifacts as fixtures. Hunter-side follow-up below.
- **PROS-LEAD-002 (P2, fixed):** Hunter contacts with a cited source_url on an unrelated registrable domain (observed: `bar@developer.mozilla.org` citing api.rocket.rs) are dropped in `HunterProvider` (`hasConsistentSource`, last-two-labels registrable-domain approximation; missing source_url kept). 5 Pest tests.
- **PROS-API-001 (P3, fixed):** free-audit preview maps reachable-but-4xx targets to precise 4xx — 404 mirrors the origin status ("The website could not be audited (it returned HTTP 404)."), others stay 502. `WebsiteTool::analyze()` now forwards the origin `status`. Verified live via `POST /api/v1/public/audits`.
- **PROS-INFRA-001 (P1, code-side fix, deploy required):** security headers added in `next.config.ts` (`headers()`): HSTS max-age=31536000 includeSubDomains (no `preload` — deliberate, see comment), XCTO nosniff, XFO SAMEORIGIN, Referrer-Policy strict-origin-when-cross-origin, Permissions-Policy (camera/mic/geo off). Verified live on `next start` for / and /login. Production still needs a redeploy for these to appear at the edge (2026-09-25 re-check: still absent in prod — see the live post-deploy verification block below).
- **PROS-SEO-001 (P3, fixed):** `/resources/accessibility-badge` added to `sitemap.ts` (lastmod 2026-09-07, monthly, 0.6) — was indexable but orphaned.
- **PROS-QUEUE-001 (P2, documented):** runbook added below. Root cause remains `queue:work` without `--queue`; audits on `ai` queue strand in pending/analyzing.
- **PROS-INFRA-002 (P1, deployment-bound):** production `/verify-email/*` 403 is NOT reproducible locally (200 in dev and prod-style builds). Requires edge/deploy access to resolve; documented as a redeploy-verify checklist item, not code-fixed.

### PROS-DASH-001 verification evidence (2026-09-24)

Reproduced-then-fixed-then-proven, per layer:

- **Unit (vitest, `src/lib/axios.test.ts` — 7 tests):** 401 on a protected endpoint clears localStorage
  token AND middleware cookie before redirecting; redirect fires exactly once under concurrent 401s
  (single-flight); no redirect when already on /login (credentials still cleared); return path preserved via
  `?redirect=`; guest-allowed probes (`/auth/me`, `/billing/status`) never redirect but purge an orphaned
  middleware cookie; non-401 errors leave auth intact.
- **E2E (Playwright, 17/17, real backend on :8001):** garbage/missing tokens → exactly one server-side
  307 to `/login` on /dashboard, /admin, /leads; expired/revoked tokens → recovery to a usable /login with
  both credential stores cleared (poll-asserted) and 401s bounded ≤16 (one ~8-query dashboard pass + retry
  headroom; the audit's loop produced unbounded 23+); sentinel test replays the audit's original attack
  (garbage cookie) and asserts still-on-/login 5s later with zero bounce-backs; positive control: a real UI
  login lands on a stable /dashboard.
- **Live browser stale-token flow (dev):** planted fake `prospetra_auth_token` + cookie → `/dashboard` →
  landed on `/login` ("Welcome Back" rendered), both stores empty; still on `/login` 22s later, no bounce;
  then a real login → `/dashboard` stable with token + cookie restored.
- **Production-build curl matrix (`next start`):** `/dashboard` + junk cookie → 307 `/login?redirect=%2Fdashboard`;
  `/dashboard` + legacy `"1"` marker → 307 (old presence-check accepted both — these were loop entries);
  `/login` + no cookie → 200; `/dashboard` + well-formed token → 200; `/login` + format-valid stale cookie →
  307 `/dashboard`, which terminates by design because the interceptor clears credentials on the bounce-back
  (proven in-browser by the revoked-token E2E).
- **Deploy-bound note:** production still runs the pre-fix edge build; the loop dies on the next frontend
  redeploy. Post-deploy check: `curl -s -o /dev/null -w "%{http_code} %{redirect_url}" -H "Cookie: prospetra_auth=junk" https://prospetra.com/dashboard` must show a 307 to `/login?...` (format gate), and the Playwright
  suite can be pointed at prod via `QA_API_URL` + `baseURL` overrides.

### proxy.ts migration verification evidence (2026-09-24)

- **Convention**: `middleware.ts` git-renamed to `src/proxy.ts` (`R middleware.ts -> proxy.ts`); exports
  `proxy(request: NextRequest)` + the original `config.matcher`. Route tables (public/protected) preserved
  verbatim — the only behavioral change is the auth-signal check itself.
- **Placement pitfall (cost us a debugging cycle — do not repeat):** with this project's `src/app` layout the
  file MUST live at `src/proxy.ts`. A `proxy.ts` at the repo root compiles cleanly but is silently ignored —
  no build error, no runtime hint; the dev server simply stops gating. Detected only because the live curl
  matrix returned all-200s (even no-cookie `/dashboard`), then confirmed by moving the file into `src/`.
- **Format gate**: `looksLikeSanctumToken()` — `^\d{1,10}\|[A-Za-z0-9]{40,64}$`, URL-decode-first, shared via
  `src/lib/auth-token-format.ts`. Empirically calibrated: Sanctum `plainTextToken` is `<id>|` + 40 entropy +
  8 crc32b = **48 chars** (verified in `HasApiTokens::generateTokenString()` and via tinker); the first draft
  demanded exactly 40 and would have bounced every real logged-in user on hard refresh — caught by the
  Playwright revoked-token tests (real 48-char tokens were proxy-redirected before client recovery ran),
  fixed to the tolerant 40-64 band. Sanity check, not auth: rejects junk, corruption, legacy `"1"` markers;
  real verification stays with API Bearer auth; stale-but-well-formed tokens recover via the 401 interceptor.
- **Contract fix**: `setSessionCookie()` no longer writes the bare `"1"` presence-marker fallback (would fail
  the gate and silently de-authenticate protected routes); cookies are always the token itself.
- **Verification**: 9 unit tests on the predicate (accepts 48-char + URL-encoded; rejects junk, `"1"`,
  session/CSRF-payload shapes, 39/65-char lengths, non-numeric ids, multi-pipe; accepts 41-64 band) — vitest
  23/23 overall; Playwright 17/17 incl. server-side 307 assertions for garbage/missing cookies; prod-build
  curl matrix: junk → `307 /login?redirect=…`, `"1"` → 307, well-formed → 200, no-cookie `/login` → 200.
  Note: `.next/server/middleware-manifest.json` shows `middleware: {}` on Next 16 even when the proxy is
  active — the compiled chunk does contain the proxy code; functional probes are the source of truth.

### CI wiring (per-repo workflows; 2026-09-24, corrected topology)

**Topology correction:** frontend and backend are independent git repos with their own GitHub remotes
(`Prospectra-frontend`, `Backend-Prospectra`), gitignored by the root docs repo — so a root-level
`.github/workflows/ci.yml` (the first scaffold) could never run: GitHub only reads workflows from the
repo being pushed. The root workflow was **removed**; each repo now carries its own
`.github/workflows/ci.yml`, triggered on every push to main and on all pull requests:

- **Backend CI** (`Backend-Prospectra`): one job — composer install, env+key from `.env.example`
  (sed-replaced, dotenv immutable), SQLite `:memory:` migrations, **Pint**, **Pest** (463 tests).
- **Frontend CI** (`Prospectra-frontend`): `checks` job — type-check, lint, vitest (23), and a
  production build (`NEXT_PUBLIC_API_URL` injected because `.env.production` is gitignored and
  `assertDeployableApiUrl` refuses builds without it) — plus **stale-token-matrix**, the P0 regression
  gate from audit §76/§182 (expired/garbage/revoked/missing × /dashboard,/admin,/leads + sentinel):
  MySQL 8.4 health-gated service container on 127.0.0.1:3306, **sibling backend repo checked out**
  (`actions/checkout` `repository: musmanidrees09/Backend-Prospectra`, `BACKEND_REPO_TOKEN` PAT only
  needed if private), real Laravel on 127.0.0.1:8001 with a wait-loop + JSON-envelope/CORS
  health-check gate, QaUserSeeder, Playwright chromium, suite on 127.0.0.1:3100 via webServer,
  failure-artifact uploads (report/traces + serve.log/laravel.log).
- Required secrets in **Prospectra-frontend**: `QA_EMAIL`, `QA_PASSWORD` (and `BACKEND_REPO_TOKEN`
  only if the backend repo is private); optional var `BACKEND_REF` to pin the backend branch.
- Rehearsals already run locally this session: CI-mode stale-token 17/17 (53.3s, HTML report),
  production build with injected public URL green, seeder both paths, deploy-checklist 4/4 on a
  local prod build. First real Actions runs happen on the next push to either repo.

Historical (superseded single-workflow scaffold) notes:
runs MySQL 8.4 as a health-gated service container on 127.0.0.1:3306, boots the real Laravel backend on
127.0.0.1:8001 (`php artisan serve`, real 401s + real token revocation — mocked 401s can't exercise
counter semantics), migrates + seeds the QA user via the new `QaUserSeeder`, gates on the reuse of
`scripts/check-backend-health.cjs` (JSON envelope + CORS for the suite origin), installs Playwright's
chromium, and runs the suite on 127.0.0.1:3100 via the config's `webServer`. Requires repo secrets
`QA_EMAIL`/`QA_PASSWORD` (set them before first push; the workflow itself is dormant until the repo has
a remote).

- **QaUserSeeder** (`backend/database/seeders/QaUserSeeder.php`): idempotent, config-driven
  (`prospetra.qa.email/password` ← `QA_EMAIL`/`QA_PASSWORD`, documented in `.env.example`). Create-only:
  never resets an existing QA user's password (deliberate — no account-takeover primitive, and saved
  traces keep working); force-fills only role/status/verification and attaches the HQ org. Verified both
  paths this session: existing-user path on the dev MySQL (user id=12 untouched, password still verifies,
  two runs, no duplicate) and create-path on a throwaway SQLite (fresh user id=1, org attached, system
  role `user` assigned even on an empty roles table — so CI needs no extra role seeding).
- **Gotcha (documented in the workflow):** Laravel dotenv is immutable — first value wins — so CI
  overrides `.env` copied from `.env.example` by **sed-replacing keys** (APP_ENV/APP_DEBUG/APP_URL/
  FRONTEND_URL/DB_PASSWORD/MAIL_MAILER), not by appending (appending would silently keep
  `APP_ENV=production` and the HTTPS-only CORS guard). `FRONTEND_URL=http://127.0.0.1:3100` is what
  makes CORS allow the Playwright origin (local-env origin patterns cover any `127.0.0.1:*`).
- **Incidental fixes:** full-repo Pint had 1 pre-existing style issue in `AnalyzeWebsiteJob.php`
  (uncommitted PROS-ERR-001 code) — fixed style-only (import order + spacing); Pest re-run green
  (AuditFailureClassifier 9/65). Local QA keys were codified into gitignored `backend/.env`.
- **Failure artifacts + CI-mode rehearsal (2026-09-24, later):** the matrix job now uploads
  `playwright-report/` + `test-results/` (traces/screenshots, `if: failure()`) plus backend
  `serve.log`/`laravel.log` when red — without the upload, the config's `retain-on-failure` evidence
  died with the runner. The CI reporter now emits the HTML report alongside `line` (the artifact path
  would otherwise have been empty). Rehearsed locally in exact CI mode (`CI=1`, health-check gate
  first): **17/17 passed (53.3s)** with the HTML report generated; vitest 23/23 same day.
- **Automated deploy gate (2026-09-24, later):** the deploy checklist is now a Playwright suite —
  `tests/deploy/deploy-checklist.spec.ts` + `playwright.deploy.config.ts` (no webServer; targets a
  running deployment via `DEPLOY_URL`, run with `pnpm test:deploy`). Asserts the five security headers
  byte-for-byte (HSTS value checked only on https targets; browsers ignore it on loopback http), the
  `/verify-email/*` 200 (page shell must mount in any app state — the title cycles, so assert the
  AuthShell `h1` matches /verif/i), and the junk-cookie `307 → /login` proxy gate. Rehearsed green
  (4/4, 4.2s) against a local prod build on :3222; run against prospetra.com it went **3 failed / 1
  passed — exactly the expected pre-fix deploy state** (headers absent, junk cookie 200, no gate).
  Discovery: prod `/verify-email/*` now returns **200** (curl-confirmed) — the audit's 403 no longer
  reproduces, presumably fixed at the edge since 2026-09-23; PROS-INFRA-002 should be re-statused at
  the next audit review.

### Queue worker runbook (PROS-QUEUE-001)

Audits are dispatched to the `ai` queue; email jobs to `emails`. A bare `php artisan queue:work`
drains `default` only and silently strands audits in pending/analyzing (reproduced locally).

```bash
# Correct invocation (worker exits after draining; restart via supervisor/systemd in prod)
php artisan queue:work --queue=ai,emails,default --tries=3 --timeout=900

# Verify jobs are actually being consumed
#   - jobs table: SELECT queue, COUNT(*) FROM jobs GROUP BY queue;  (should stay near 0)
#   - failed_jobs: alert on growth; schedule `queue:prune-failed`
```

Deploy checklist for /verify-email/* (PROS-INFRA-002): after redeploying the frontend, run the
automated gate — `DEPLOY_URL=https://prospetra.com pnpm test:deploy` (covers the security headers,
`/verify-email/*` 200, and the proxy 307 gate; the manual equivalent is
`curl -s -o /dev/null -w "%{http_code}" https://prospetra.com/verify-email/<id>/<hash>` expecting 200).
Note (2026-09-24): prod already returns 200 on `/verify-email/*` — the 403 no longer reproduces.

Fast first gate for the same three items, no browser needed — `pnpm test:smoke https://prospetra.com`
(or `DEPLOY_URL=... pnpm test:smoke`; script: `scripts/post-deploy-smoke.cjs`). Asserts the five security
headers byte-for-byte on / and /login, `/verify-email/*` = 200, and the PROS-DASH-001 loop fix: junk
`prospetra_auth` cookie → single 307 to `/login?redirect=…` whose landing STAYS on /login (no bounce-back),
legacy `"1"` marker also gated, well-formed token NOT bounced. Zero dependencies (global fetch + HEAD
requests); exit 0/1 so it can slot into a post-deploy pipeline step; the Playwright `test:deploy` suite
remains the full end-to-end gate.

**Live post-deploy verification (2026-09-25): production still runs the pre-fix build — redeploy pending.**
All three gates agree on prospetra.com (curl matrix, `test:smoke` 4/6 failed, `test:deploy` 3 failed / 1 passed):
all five security headers absent on / and /login (PROS-INFRA-001); junk `prospetra_auth` cookie and the legacy
`"1"` marker both get HTTP 200 — the format gate is NOT live (PROS-DASH-001). No-cookie `/dashboard` still 307s
to `/login?redirect=%2Fdashboard` (the old presence gate, proving the proxy itself runs — the deployed build
merely predates the fix); a well-formed token gets 200; `/verify-email/*` gets 200 (PROS-INFRA-002 green at the
edge, re-confirmed). The deploy-bound items stay deploy-bound until a frontend redeploy lands and
`pnpm test:smoke https://prospetra.com` exits 0.

### Verification summary (2026-09-24, re-verified 2026-09-25)

Backend: 468 tests / 1,485 assertions passing (incl. the new Hunter provenance, extraction, failure-classifier,
score-derivation, and PSL registrable-domain tests), Pint clean across all 414 files; `psl:refresh` rehearsed
live (334,675 bytes, runtime copy stays gitignored). Frontend: type-check clean, lint 0 errors, production build
successful, vitest 27/27 (incl. retry-guidance), Playwright 17/17 (re-run green in exact CI mode on 2026-09-24,
53.3s, HTML report emitted). Live probes: 404-mapping and security headers verified on `next start`;
stale-token matrix verified against the real backend and DB.

### 2026-09-24 (later): specific audit failure reasons (PROS-ERR-001)

- New `backend/app/Services/Audit/AuditFailureClassifier.php`: one classifier for every failure path —
  maps origin HTTP status, known WebsiteTool messages, and transport signatures (timeout/DNS/TLS,
  bot-protection text) to specific customer-safe reasons + machine-readable codes
  (`bot_protection`, `not_found`, `timeout`, `dns_failure`, `http_error`, `content_too_large`,
  `unsupported_content_type`, `too_many_redirects`, `ai_providers_unavailable`, `analysis_failed`).
- Wiring: pipeline crawl-failure + catch paths, `AnalyzeWebsiteJob::failed()`, and the sync fallback in
  `AuditController` all classify before storing; raw exception text now stays in logs only.
- New `audits.error_code` column (migration run), exposed via `AuditResource` and typed in the frontend.
- Frontend `src/lib/audit-failure-reasons.ts`: code-driven display reason with legacy text-signature
  fallback + technical-blob guard; audit detail page uses it (`customerSafeAuditError(code, message)`).
- Tests: 9 Pest tests (classifier, incl. no-leak guarantees) + 5 vitest tests (mapper, legacy rows).
- Verified live: stackoverflow.com → `bot_protection` ("refused automated access (HTTP 403) — most likely
  bot protection such as Cloudflare"); a 404 URL → `not_found` with URL guidance. The old generic message
  is only the empty-evidence fallback.

### 2026-09-24 (addendum): retry guidance for deterministic failures (PROS-ERR-001 follow-on)

Bot-protection failures are deterministic (a re-crawl of stackoverflow.com will always be challenged),
so "Retry" was a dead end for users. `audit-failure-reasons.ts` now exposes a retry-guidance model
(`getRetryGuidance`: deterministic | transient | unknown; `DETERMINISTIC_CODES = {bot_protection}`) and
`BOT_PROTECTION_GUIDANCE` copy that names the real remedy (allow-list audit access in the site's bot-
protection settings). UI: the audit-detail banner for deterministic failures shows an amber "Retrying
won't change this result." guidance box and swaps the Retry button for a "Start a new audit" link;
the audits list shows "Blocked by bot protection — retrying won't change this." with a "New audit"
affordance. Transient failures (timeout, dns, AI outage) keep the one-click Retry. Verified: vitest
27/27 (4 new guidance tests, incl. a product-safety test asserting the copy never promises unshipped
affordances like "manual review"), type-check clean, lint 0 errors, and a 2-test Playwright spec
(`tests/e2e/failed-audit-guidance.spec.ts`, env-gated skip) passed 2/2 against REAL failed audits —
the bot_protection row (ffcfdca5…) and the not_found row (0f7a734f…) — through the real token-injection
auth flow.

### 2026-09-24 (addendum): Hunter cache-hit defense in depth (PROS-LEAD-002)

- `CACHE_VERSION` bumped 2→3: pre-filter cached Hunter payloads are invalid wholesale.
- `AuditProviderManager::run()` now re-applies the provenance filter on Hunter cache hits via
  `ProviderResult::withFilteredHunterContacts()` — results cached before the filter shipped (up to 30 days)
  can no longer leak cross-domain contacts; an all-cross-domain result downgrades to `no_data` with the
  standard message rather than keeping a PASS with unusable evidence.
- `HunterProvider::hasConsistentSource()` is now static (pure function of item + domain) so both paths
  share one filter implementation.
- Tests: +3 (`AuditProviderManagerCacheFilterTest`) covering stale-cache re-filtering, clean-cache
  passthrough, and the all-cross-domain downgrade. Provider cache-key assertions updated v2→v3.

### 2026-09-24 (addendum): anti-synthetic-scoring regression test (PROS-TRUTH-001)

- New `tests/Feature/AuditScoreDerivationTest.php` drives the full public-audit pipeline
  (`fetch -> crawl -> extract -> analyze -> score`) network-free via `Http::fake` on resolvable
  fixture hosts (`example.com` / `example.org`; SafeWebsiteUrl still performs real public DNS,
  matching the existing PublicAuditTest pattern).
- Assertions lock the audit-truth invariants: two genuinely different fixture sites produce **different
  scores** (<100); titles/page metrics (word counts, images-without-alt, size) equal values computed
  independently from the fixture HTML; category scores move in content-dictated directions (B > A for
  seo/security/accessibility); **mutating the content (adding alt text) strictly improves the score** on
  the same URL — impossible with hardcoded or URL-keyed scoring; repeat audits of an identical fixture
  are exactly deterministic (no randomness).
- Test-craft note captured in-file: fake responses must be returned fresh per request (closure), because
  the pipeline reads bodies from streamed responses — a shared fake response object hands subsequent
  audits an already-consumed stream (real servers never do this).

### 2026-09-24 (addendum): Public Suffix List adoption for registrable-domain checks (PROS-LEAD-002 follow-on)

`jeremykendall/php-domain-parser ^6.4` (the `Pdp\` package) now powers registrable-domain resolution —
`RegistrableDomainService` loads the PSL from local disk (`storage/app/psl/`), never from the network at
request time: a runtime copy refreshed by the new `psl:refresh` artisan command (downloads to the runtime
path, sanity-checks size, keeps the old file on failure), plus a committed snapshot
(`public_suffix_list.fallback.dat`, forced through the storage-app gitignore with re-include rules) so CI,
tests, and fresh clones work offline. The parsed `Rules` object is memoized per process (NOT via the
Laravel cache — the database cache store cannot round-trip it: fetched values came back as
`__PHP_Incomplete_Class`). `HunterProvider::hasConsistentSource` now compares PSL registrable domains;
unresolvable hosts keep the contact (safe bias), and the old last-two-labels approximation was deleted.
Exactness where the heuristic was wrong in both directions: `foo.co.uk`'s registrable domain is now
`foo.co.uk` (the heuristic returned `co.uk`), and private suffixes like `github.io` resolve per-site, so
`a.something.github.io` vs `b.something.github.io` contacts are dropped while `www.foo.co.uk` vs
`foo.co.uk` are kept. Verified: 5-test `RegistrableDomainServiceTest` (both failure directions of the old
heuristic, IP/localhost/empty-label rejection, cross-domain/same-domain matrix), provenance + cache-filter
suites green, full backend 468 tests / 1,485 assertions, Pint clean 414 files. Out of scope deliberately:
`WebsiteIntelligenceExtractor::COMMON_TLDS` (an email-regex boundary, not domain identity) and
`RdapProvider`'s last-label TLD lookup (registration service selection) remain as-is.

### 2026-09-24 (addendum): email extraction hardening evidence (PROS-LEAD-001)

The fix landed in three independent layers of `WebsiteIntelligenceExtractor`, all confirmed on disk and
test-verified again this session:

- **No glue at the source** — `visibleText()` (L256) strips `script/style/noscript/template`, then
  space-separates every tag boundary before `strip_tags`, so adjacent DOM text nodes can never fuse into
  tokens like `usinfo@…` in the first place.
- **TLD-boundary pattern** — page-text emails must end in a known TLD: `COMMON_TLDS` (L33, multi-part
  entries first so `co.uk`/`com.au` win longest-match) feeds `EMAIL_PATTERN_TEMPLATE`
  (`…@(?:[A-Z0-9\-]+\.)+(?:%s)(?![A-Za-z0-9])/i`, L57); the trailing lookahead kills suffix-glued junk
  like `info@…comemailinfo`.
- **Validated dedupe** — `addEmail()` (L275) lowercases, enforces `FILTER_VALIDATE_EMAIL`, and keeps one
  entry per address preferring mailto evidence (0.95) over page text (0.8).

Verification (fresh run, 2026-09-24): `php artisan test --filter="EmailExtraction|WebsiteIntelligence"`
→ **7 passed, 28 assertions**, including the four PROS-LEAD-001 regression tests that use both audit-
observed artifacts as fixtures (prefix-glued, suffix-glued), assert the TLD-boundary requirement, and
lock the mailto-preferring dedupe. Hunter-side cross-domain contact filtering is covered separately in
the PROS-LEAD-002 addendum; live crawl evidence in Appendix A.7 of the production audit doc.

Live re-audit of the same target (2026-09-25, post-fix pipeline): re-ran audit 9's site (https://prospetra.com)
end to end through the real queue — new audit `c0a468cf-bc22-4d90-b3da-ca4121835408` (id 32, same QA
workspace), completed in 47s, score 91, no errors. Baseline: audit 9 had stored three emails —
`info@prospetra.com` (mailto 0.95) plus both glue artifacts `usinfo@prospetra.com` and
`info@prospetra.comemailinfo` (page_text 0.8). The re-run stores exactly ONE email, `info@prospetra.com`
(mailto, 0.95, observed); a recursive scan of the full report JSON finds NO occurrence of either artifact and
no other `@`-string beyond a meta-description sentence containing the valid address. Crawl reach is proven
equivalent to audit 9: the `/contact` form block (name/email/message fields, 0.95) is present, and the run
additionally captured a phone (`+923436432568`, page_text 0.9) that audit 9 had missed — the artifact loss is
extraction cleanup, not a shallower crawl.

## 2026-09-25 — Dashboards/forms playtest + Google indexing fixes

**Product verification (dev :53367, headless Playwright):**
- User workspace: login OK; sidebar clicks through /audits /leads /companies /emails /proposals /analytics
  /settings — all client-side (`window.__spa` marker survives; zero full reloads). Dashboard heading renders.
- Admin: dashboard shows Admin link; SPA click to /admin OK; /admin/users, /admin/audits, /admin/blog,
  /admin/service-requests all render; blog editor /admin/blog/1 loads 26 fields; zero page errors.
- Forms: contact 201 (backend /public/contact); service-request new_website (message required) 201;
  existing_website (URL required, https enforced) 201; invalid submits blocked client-side with field errors.
- Rate limiter note: service.requests 5/5min/IP tripped twice during testing — `php artisan cache:clear`.
- Links: wa.me/923436432568 and mailto:info@prospetra.com present in footer/contact.
- Preview-tab quirk (not a bug): React 19/Next 16 defers hydration-render for background tabs
  (document.visibilityState=hidden -> root div[hidden]); DOM assertions still work.

**Indexing investigation (GSC 9/21: 23 indexed / 17 not):**
- Live prod runs the PRE-services build: /services, /services/seo, /services/web-development,
  /request-service all 404 live but ALREADY in sitemap.ts (dated 2026-09-25) => the 10 "Discovered -
  currently not indexed" and the deploy gap is the dominant cause. DEPLOY REQUIRED, then GSC revalidate.
- Live sitemap is 27 healthy URLs (200, self-canonical, index/follow). No thin posts; 401-3,018 words.
- www(308) + trailing-slash(308) variants explain "Page with redirect"(2) + "Alternate page with proper
  canonical tag"(2) — benign; canonicals are correct.
- Fixed in repo: robots.ts now disallows /login /register /logout /forgot-password /reset-password
  /verify-email (was noindex-meta only — those crawlable auth URLs are the GSC not-indexed stragglers);
  not-found.tsx no longer inherits the homepage canonical (alternates canonical: null);
  /resources/accessibility-badge was a pure orphan (sitemap-listed, zero inbound links) — linked from the
  footer Product column; /services got ItemList JSON-LD enumerating the services layer.
- Verified: tsc clean, lint 0 errors (4 pre-existing warnings), vitest 37/37, prod build OK
  (32 sitemap URLs incl. services layer + badge; auth Disallows present in built robots.txt.body),
  ItemList + badge link + no-404-canonical confirmed on dev live HTML.
- Post-deploy checklist: submit sitemap, "Request indexing" for the 4 service URLs + badge, validate the
  two Started fix groups; trailing-slash/www rows will close on their own.

### 2026-09-25 (b) — Both service journeys rehearsed live; CTA visibility bug found + fixed

- Journey A (audit-sourced): QA user opens /audits/{audit 32 public_id} report, clicks the
  "SEO services" CTA -> /request-service?service=seo_services&audit={public_id}; form shows the
  attached audit; submits WITHOUT a website URL -> 201. DB row id 23: project_type=existing_website,
  service_type=seo_services, audit_id=32, website_url=https://prospetra.com (resolved FROM the audit),
  source=audit_report, lead auto-associated (lead_id 7).
- Journey B (new website): /request-service?project=new_website&service=website_development —
  #sr-website input correctly ABSENT; submits with no URL -> 201. DB row id 24: project_type=
  new_website, audit_id=null, website_url=null, source=public_form, no lead (correct).
- Admin drawer (/admin/service-requests): both rows opened via "Open"; A shows "Linked audit" card
  (prospetra.com, score 91); B shows "New website" with em-dash URL; status -> contacted + notes saved
  on both; zero pageerrors.
- BUG FOUND + FIXED by the rehearsal: on the (dashboard) audit report page the new
  <AuditServiceCtas> block had been placed inside a legacy `div.hidden` wrapper (pre-existing hidden
  block at HEAD) — the services CTAs existed in the DOM but were invisible to users. Moved the block
  out (renders after findings, gated on findings.length > 0). type-check clean, vitest 37/37.

### 2026-09-25 (c) — Full e2e services rehearsal vs PROD build: 3/3 pass, no regressions

- Rebuilt with ALLOW_LOCAL_API_IN_BUILD=1 + NEXT_PUBLIC_API_URL=http://127.0.0.1:8001/api/v1
  (includes the CTA-visibility fix from (b)), served via `next start -p 3100` (pid logged),
  `php artisan cache:clear` first to reset the service.requests limiter.
- `pnpm exec playwright test tests/e2e/services-journey-rehearsal.spec.ts` -> 3 passed (23.5s):
  J1 funnel report (share-token audit id 8, vercel.com) -> CTA -> submit with empty URL -> 201
  (DB row 25: audit_id=8, url resolved from audit); J2 services -> new-website no-URL -> 201
  (DB row 24/26 pattern); J3 admin UI login -> sidebar -> service-requests -> Open drawer ->
  status contacted + notes saved.
- Rehearsal server on :3100 stopped after the run; dev :53367 untouched.

### 2026-09-25 (d) — Rehearsal test data cleaned from local DB

- Deleted 15 service_requests (ids 12-26: five e2e runs J1/J2 pairs, Rehearsal A/B, 2 playtest rows —
  every row in the table was a today's-testing artifact).
- Leads 6 ("Journey Rehearsal") and 7 ("Rehearsal A 0925") deleted AFTER detaching the two fixture
  audits that referenced them (audit 8 vercel.com, audit 32 prospetra.com — both preserved intact,
  lead_id set null so a future request can re-associate).
- Also removed the 1 playtest contact_messages row from forms testing.
- Verified: service_requests=0, contact_messages=0, no rehearsal emails/names remain, leads table back
  to its 4 real QA leads, audits 8/32 intact.

### 2026-09-25 (e) — CTA-visibility regression spec added (red-green proven)

- New: frontend/tests/e2e/audit-report-service-ctas.spec.ts (PROS-SVC-CTA-VIS). Resolves the QA user's
  newest completed audit WITH findings via the real API (login -> /audits list), then UI-logs-in and
  asserts on /audits/{public_id}: (1) the "implementing these recommendations" region is VISIBLE and
  its first link carries audit={public_id}; (2) no hidden attribute / display:none ancestor between
  the region and document body — the exact historical failure mode.
- RED-GREEN proven against PROD builds on :3100: with the hidden wrapper re-introduced and rebuilt,
  BOTH tests fail ("element(s) not found" — toBeVisible cannot see into display:none); with the fix
  restored and rebuilt, both pass (19.3s). First red attempt was INVALID (stale :3100 build) and was
  discarded — the valid red ran against a fresh build containing the bug.
- Note: `next dev` refuses a second dev server in the same dir (Next 16), so the playwright.config
  webServer fallback cannot coexist with the long-lived :53367 preview; run the suite against a
  `next start -p 3100` build (config reuses an answering server).
- :3100 stopped after the run; type-check clean; spec is read-only (no DB mutations).

### 2026-09-25 (f) — SEO smoke gate added to post-deploy smoke (red + green proven)

- scripts/post-deploy-smoke.cjs gains check 4 (PROS-SEO-SMOKE): fetch /sitemap.xml, map every <loc>
  onto the target origin (rehearsals build with the prod origin), then per URL require HTTP 200 with
  redirect:"manual" (any 3xx = GSC "Page with redirect"), no noindex via meta robots OR X-Robots-Tag,
  and a self-referential canonical (same pathname, trailing slash ignored -> cross-page canonical
  fails). 5 URLs in parallel; failures each listed with their reason; non-zero exit on any failure.
- RED (fixture server with all failure modes): 404, 301->/good, meta-robots noindex, X-Robots-Tag
  noindex, cross-page canonical each flagged; the one good URL passed; exit 1.
- RED (live prospetra.com, pre-fix deploy): script exits 1 (4 pre-existing check failures); SEO gates
  pass its 27 old URLs — the new service URLs are absent live, which is exactly what the deploy gates.
- GREEN (local prod build on :3222, includes services pages): 8/8 checks pass in 3.7s, 32 sitemap URLs
  all 200/indexable/self-canonical, exit 0. Rehearsal server + fixture cleaned up; lint 0 errors.

### 2026-09-25 (g) — Service + FAQ structured data on the services pages

- New reusable src/components/seo/service-structured-data.tsx (Service + Offer; provider links to the
  StructuredData Organization @id; NO price — quoting follows the audit by design, availability InStock;
  same < escaping as FaqStructuredData). Unit-tested: node shape, provider link, no invented price,
  </script> breakout escaping (2 tests).
- /services/seo: Service node added alongside its existing FAQPage/Breadcrumb graph.
- /services/web-development: had NO FAQ section; added one — 5 Q&As restating decisions already in the
  page copy (no-website-yet, audit-first redesign, build standards, quote-based pricing, post-launch) —
  because the repo's own FaqStructuredData docblock (correctly) requires visible Q&A before FAQPage
  schema. FAQPage + Service nodes now emitted; matches the homepage's graph coverage
  (Organization/WebSite/SoftwareApplication + BreadcrumbList + FAQPage + Service).
- Verified on dev: both pages 200; SEO page ld types incl FAQPage + Service; web-dev page renders 5
  <details> FAQs in DOM and emits FAQPage + Service; vitest 39/39 (2 new); lint 0 errors; prod build
  emits Service in services/*.html.

### 2026-09-25 (h) — Full recheck of every page against the prod build; content fixes

- Live prospetra.com still runs the PRE-services deploy (4 service URLs 404 live, 27-URL sitemap) —
  the deploy remains the blocking step for the "Discovered - not indexed" bucket.
- Audited all 32 sitemap URLs against the CURRENT prod build (:3222): every URL 200, index,follow,
  self-canonical, h1 present, no duplicate titles/descriptions, blog 1,029-3,034 words. Schema present
  on all content pages (Service/Offer/FAQPage/BlogPosting per page type).
- Found + fixed real issues:
  * "Prospetra | Prospetra" double brand suffix on /services, /services/web-development,
    /request-service (page titles self-suffixed AND layout template appended again) — dropped the
    manual suffix; layout template supplies it once.
  * Meta descriptions >160 (Google truncates): trimmed on /services, /services/seo,
    /services/web-development, /request-service.
  * /request-service was thin (158 words): added a genuine "What happens after you submit" block
    (audit-first scoping, one-business-day reply, fixed price, no-audit path) — restates real
    behavior, no filler; now 263 words with real content, not padding.
  * Titles >65 chars trimmed on /services/accessibility-audit-pakistan and
    /resources/accessibility-badge (metadata + og/twitter copies).
- Post-fix audit: all 32 URLs pass every gate; remaining audit-tool noise is measured-HTML artifacts
  (&amp; counting, 66-char decoded titles on 2 legacy blog posts = truncated ampersand space; homepage
  canonical confirmed present). request-service "thin" flag satisfied with 263 real words incl. form
  labels.
- vitest 39/39, lint 0 errors, tsc clean, rebuild green; audit temp scripts removed.

### 2026-09-25 (i) — Post-fix full recheck; Offer JSON-LD fix; validate:indexing gate

Recheck scope: every sitemap URL against the current prod build (:3222, three
validators), auth pages, 404 page, robots.txt, and sitemap composition.

- **All 32 sitemap URLs pass every gate** (validate-indexing + validate-seo +
  validate-jsonld, all 0 issues): 200, single h1, index,follow, self-canonical
  on the apex origin, no duplicate titles/descriptions, 245-3,034 words/page,
  schema present per page type (Service/FAQPage/BlogPosting/Breadcrumb).
- **New pages verified live-rendered:** /services (562 words), /services/seo
  (584), /services/web-development (662), /request-service (263) — content,
  titles, descriptions, Breadcrumb/FAQ/Service JSON-LD all render; sitemap has
  32 URLs with dated lastmod for the new pages; internal links to the services
  layer present from the public header/blog/service CTAs.
- **Fixed: invalid price-less Offer JSON-LD on /services/seo and
  /services/web-development** (validate-jsonld caught it; the earlier same-day
  addendum (g) had introduced it). Google requires price on Offer; the pages
  deliberately quote after the audit, so the Offer node was removed entirely —
  a Service without offers is fully valid and no price is fabricated. Test
  updated to lock "no offers node"; the validator rule was also corrected:
  offers stays optional on Service (schema.org semantics) but Offer keeps its
  hard price requirement, so this bug class can't return silently.
- **New permanent gate `pnpm validate:indexing`** (scripts/validate-indexing.js):
  for every sitemap URL asserts HTTP 200, exactly-one canonical on the apex
  origin (homepage → bare origin accepted), no noindex, exactly one h1, and
  ≥200 words (tunable via --min-words). Home + /contact needed the tunables;
  both are correct as shipped (245 words is fine for a conversion page).
  Complements validate:seo and validate:jsonld — together they cover reachability,
  metadata, and structured data.
- **Auth pages**: /login /register /forgot-password /reset-password
  /verify-email/* each serve `noindex, nofollow` (defense in depth on top of
  robots.txt Disallow). 404 page: noindex + NO canonical (inheritance fixed
  earlier). robots.txt correct (apex sitemap, auth/workspace disallows).
- **Deploy still pending**: live prospetra.com re-confirmed on the old build —
  /services, /services/seo, /services/web-development, /request-service return
  404 live and the live sitemap is 27 URLs. Everything blocking indexing is
  fixed in-repo; after redeploy run `pnpm test:smoke https://prospetra.com`
  and `pnpm validate:indexing https://prospetra.com`, then GSC → request
  indexing on the four new URLs.
- Verification: vitest 39/39, tsc clean, eslint 0 errors (4 pre-existing
  warnings in untouched files), prod build green, all three validators 0
  issues on :3222.

### 2026-09-25 (j) — Pricing-page report, services pricing/contact copy, responsive sweep

- **"Pricing not opening" diagnosis:** live /pricing serves 200 with the full
  plans grid (no-cookie, junk-cookie, and stale-token curl matrix all 200),
  and both dev and the current prod build render it perfectly in a real
  browser. Root cause of the report is the DEPLOYED build predating the
  2026-09-24 axios fix: with a stale localStorage token, the old interceptor
  redirected on the guest probe 401 from /billing/status, sending visitors to
  /login instead of showing pricing. Fix already in-repo (2f8f353 +
  guest-allowed exemption in src/lib/axios.ts). Deploying resolves it. (Also
  noted: localhost:3000 is currently occupied by a different project whose
  /pricing 404s — anyone testing locally must use the correct port.)
- **Pricing clarity copy:** /pricing's professional-services section now
  states explicitly that the plans are for the audit SOFTWARE, and that SEO
  and website development prices are discussed on contact (scope follows the
  audit). A WhatsApp line with the real number closes the section.
- **New shared contact module** (src/lib/contact-channels.ts) + reusable
  ServicesContactStrip (src/components/services/services-contact-strip.tsx):
  WhatsApp / tel: / mailto buttons with the display number, plus the same
  pricing-clarification sentence. Footer now reads its channels from the
  shared module (no duplicated literals). The strip is rendered on all six
  service surfaces: /services, /services/seo, /services/web-development,
  /request-service, /website-fix,
  /services/accessibility-audit-pakistan — so any services landing page
  carries the number, not just the footer.
- **Responsive sweep:** qa-responsive.cjs route list extended with the four
  new service URLs; 95/95 anon checks pass across 320/375/768/1024/1440
  (zero horizontal overflow; the only flagged console entry is the known
  benign /pricing guest-probe 401). Strip buttons are ≥40px touch targets on
  mobile (DOM-verified at 375px).
- Verification: tsc clean, eslint 0 errors on all touched files, vitest
  39/39, prod build green, validate:indexing + validate:seo + validate:jsonld
  all 0 issues (32 URLs), new copy confirmed present in the built output.

### 2026-09-26 (k) — Sidebar scroll fix + notification read/deep-link fixes

- **Admin section unreachable in sidebar (user report, screenshots from live
  site):** the sticky `h-screen` workspace sidebar's `ScrollArea` lacked
  `min-h-0`, so flexbox `min-height:auto` let it grow to content height and
  pushed the ADMIN nav section + user footer below the fold with no way to
  scroll them into view. Fixed with `min-h-0 flex-1` on the ScrollArea
  (src/components/navigation/sidebar.tsx); same guard applied to the admin
  shell nav (src/components/admin/admin-shell.tsx, protective — its non-sticky
  aside stretches with page height so it never clipped). Browser-verified
  logged-in at 1280×600 (internal scroll reaches Admin panel + footer), 768×1024
  tablet (sidebar visible, Admin panel reachable with no scroll), 375px mobile
  (sidebar hidden; MobileNav chip row includes the 44px Admin chip), and admin
  shell at /admin (all 7 nav items reachable; mobile chip nav verified in code).
- **Notifications: "Mark all read" never stuck (user report):** both mark-read
  mutations wrote into the React Query cache as a bare array while the list is
  cached as `{ items, meta }` — the updater threw, so rows never flipped to
  read and the unread badge lingered. Fixed in src/hooks/use-notifications.ts
  to update inside `items`. Verified live: badge 38 → cleared, persists across
  reload, "Mark all read" hides when nothing is unread.
- **Notifications: "Audit claimed" could not be opened (user report):**
  AuditClaimedNotification was the only database notification without
  `action_url`, rendering as an inert row. Added
  `action_url => '/audits/{public_id}'` (and same for LeadCreatedNotification
  → `/leads/{public_id}`); AI task + verify-email notifications intentionally
  have no deep link (no target page). Verified live: clicking an
  "Audit completed" notification navigates to the audit report.
- Backend tests: NotificationApiTest + NotificationWiringTest 15/15 pass;
  pint clean. Frontend: tsc clean, eslint clean on touched files, vitest
  39/39, responsive sweep 95 checks / only known-benign /pricing 401 flags.
- Commits: frontend `ec7f94c` (sidebar min-h-0 + notification cache fix),
  backend `08ff8d9` (notification action_urls). NOT pushed — user pushes.
  Deployment remains the standing blocker for all live-site reports.

### 2026-09-26 (l) — Admin entry point pinned; responsive sweep found + fixed 2 overflows

- **"Still cannot scroll to the admin panel option" (report #2, local dev):**
  the earlier `min-h-0` change made the sidebar's inner region scrollable, but
  the ADMIN section was still the last item *inside* it, so on laptop/short
  viewports the link stayed below the fold and required discovering an inner
  scroll region. The admin entry point now lives in the sidebar's fixed bottom
  stack (with quota + account cards) while the nav list scrolls independently.
  Browser-verified logged-in: fully visible with no scrolling at 1440x870,
  1280x600, 1280x420, 768x1024; at 375px the sidebar hides and the MobileNav
  Admin chip (44px) remains. `admin-shell.tsx` / `content-shell.tsx` adopt the
  same sticky screen-height sidebar so all three shells match, and their mobile
  chip rows got `scrollbar-hidden` (mobile-nav.tsx already had it).
- **Sweep harness bug found:** `qa-responsive.cjs` used stale numeric detail
  ids ("7","4"...) after the app moved to UUID `public_id`s, so every detail
  route 404'd and layout bugs there went undetected. It now resolves real ids
  from the API after login and skips routes with no record; added
  `--audit-id/--lead-id/--email-id/--proposal-id/--company-id/--blog-id`.
- **Two real responsive bugs found + fixed (320-375px):**
  - `/admin/operations`: Leads/Emails/Proposals segment control (3x `flex-1`
    `px-4`) overflowed the document by 29px at 320px. Labels kept intact; the
    track scrolls (`overflow-x-auto scrollbar-hidden`, buttons `shrink-0`), and
    the pagination row now wraps.
  - `/admin/blog/[id]` and `/content/blog/[id]`: the header row (back link +
    title + Save draft/Publish) pushed the document 86px wide at 320-375px.
    Fixed by `flex-wrap` on the row + `min-w-0` on the title block; desktop
    layout unchanged (buttons still align with the title at 1440px).
- **Cross-breakpoint polish (new `src/app/scroll-polish.css`, imported in
  `src/app/layout.tsx`):** one thin low-contrast scrollbar set for the whole
  product (WebKit + Firefox), `overscroll-behavior: contain` on ScrollArea
  viewports so inner panels stop chaining scroll to the page,
  `scrollbar-gutter: stable` so short/long pages don't shift sideways, and
  `-webkit-tap-highlight-color: transparent` + `touch-action: manipulation` on
  controls. `prefers-reduced-motion` handling already existed in globals.css.
- Verification: `pnpm type-check` clean, eslint 0 errors, vitest 39/39,
  responsive sweep at 320 + 375 = 80 checks, **0 overflow failures** (5
  remaining flags are harness CORS artifacts: the sweep injects a session into
  /login,/register,/dashboard,/proposals in headless Chrome; real browser
  sessions work fine).
- Commit: frontend `d0a5772` (9 files). Backend untouched in this round. NOT
  pushed. Note: live prospetra.com still runs the old build, so none of the
  {k} or {l} fixes are visible there yet — deploy is the remaining step.
- Tooling note for the next session: in this environment `read_files` and the
  string-replace edit tool failed on repo paths ("file does not exist" /
  BLOCKED). `write_file` works; for surgical multi-file edits a context-validated
  patch applied with `patch -p1 --fuzz=3` worked reliably (git apply was strict
  about the CRLF worktree).

### 2026-09-26 (m) — Marketing pages: one type scale, rhythm and motion language

Audit of the public surface found no shared system: five different section
paddings (py-12 ×1, py-14 ×2, py-16 ×19, py-20 ×20, py-24 ×2), two competing
heading systems (`h1-hero`/`h2-section` utilities plus ad-hoc
`text-4xl sm:text-5xl tracking-[-0.07em]` per page), zero scroll motion, and
zero `text-balance`/`text-pretty` usage.

- **Scoping trick (one edit covers all 17 public pages):** every public page
  renders the shared `PublicHeader`, so `data-public-header` was added to its
  root and the new `src/app/marketing-polish.css` is scoped with
  `body:has([data-public-header])`. Workspace/admin/content styles are
  untouched by construction. The scoped selectors deliberately out-specify
  single utility classes (extra element + attribute), so per-page values are
  replaced rather than needing 17 file edits — **retune the scale in that one
  file, not per page.**
- **Typography:** one clamp scale for h1 (36→64px), h2 (28→44px), h3
  (18→22px) with a single tracking value; `text-wrap: balance` on headings and
  `pretty` on paragraphs. Verified: all h2s now render 44px at 1440 and 28px at
  375 (previously a 36/48px mix per page).
- **Rhythm:** `.py-16`/`.py-20` collapse onto
  `padding-block: clamp(3.5rem, 7vw, 6rem)` (verified uniform 96px desktop /
  56px phone); `html:has([data-public-header])` gets `scroll-padding-top: 5.5rem`
  so `#workflow`/`#proof` anchors no longer land under the sticky header.
- **Motion:** below-fold sections reveal on scroll; links/buttons share one
  easing+duration set; `a.group` cards lift 3px on hover. Everything is
  disabled under `prefers-reduced-motion`.
- **Important implementation note:** the reveal is a small IntersectionObserver
  component (`src/components/shared/marketing-reveal.tsx`, mounted in the root
  layout), NOT a CSS scroll-driven animation. The marketing `<main>` uses
  `overflow-x: clip`, and that leaves a `view()` timeline **inactive** in
  Chrome — verified `timelineCurrentTime: null` and `effect.progress: null`
  with the animation attached, so the CSS-only version silently never ran.
  The component only adds `reveal`/`is-pending` to sections that start below
  the fold, so nothing is hidden without JS, the hero never animates on load,
  and the transition lives on the revealed state only (no fade-out flash).
  Verified: zero in-viewport sections hidden on / and /pricing.
- Verification: tsc clean, eslint clean, vitest 39/39, sweeps — 95 anon checks
  (all marketing routes × 320-1440) and 80 mixed checks at 375/1440, **zero
  overflow** (remaining flags are the known harness CORS artifacts on
  /login,/register,/dashboard).
- Commit: frontend `3c6bea9`. Still not pushed; live site needs a deploy.
- Not covered: `(auth)` login/register/forgot-password don't render
  PublicHeader, so they keep their existing type scale. Extend the same hook if
  they should join the marketing system.

### 2026-09-26 (n) — Phone touch targets audited and enforced (40 → 10 findings)

- The responsive sweep (`frontend/scripts/qa-responsive.cjs`) now audits tap
  size at phone widths (<640px): every button, link, input, select, summary and
  role-control is measured and anything under 44px is reported as `TOUCH`. Two
  correctness rules keep it honest rather than noisy:
  - the **effective** hit area is measured, including an absolutely-positioned
    `::after` expansion (how the Switch and avatar-edit button already worked);
  - a checkbox/radio inside a `<label>` is measured as the label, since that is
    what a finger actually hits.
- Root causes found and fixed:
  - **Button variants could be beaten.** `size="icon"` sets `min-w-[44px]`, but a
    call site passing `className="size-8"` wins the class merge for height while
    min-width survives — producing 44x32 controls. `scroll-polish.css` now sets
    an unlayered `min-height` floor at phone widths (`min-height` always beats
    `height`), which no utility class can override.
  - **Radix `asChild` overwrites `data-slot`** to `dropdown-menu-trigger`, so
    `[data-slot="button"]` misses every button trigger. The floor targets
    `:is([data-slot="button"], [data-variant][data-size])` instead — the
    Button's own `data-variant`/`data-size` survive the merge. Remember this
    when adding selectors for Button-rendered triggers.
  - Controls that rendered as thin text lines on phones: FAQ accordions
    (`summary`, 20px), segmented toggles, audit-list URL links (260x20),
    dashboard checklist "View" links, marketing CTAs and selects, the blog
    sticky "Run Free Audit" button, admin blog list titles, and the Switch
    (its `::after` expansion only reached ~34px tall).
- Result at 320px and 375px: touch findings 40 → 30 → 14 → **10**, zero
  horizontal overflow, `tsc` clean, eslint clean, vitest 39/39.
- Still open (deliberately left, all text links inside content blocks that WCAG
  2.5.8 exempts as "in a sentence or block of text", plus two genuine ones):
  `/contact` copy-email button (133x20), `/forgot-password` "Back to home" and
  the footer brand link (42–43px, 2px short), `/audits/<id>` "Contact
  Prospetra" (122x20), `/admin/blog/1` published checkbox (16x16, not wrapped in
  a label) and a 32x32 icon link in the same editor.
- Check the audit at any time with:
  `node scripts/qa-responsive.cjs --base http://localhost:PORT --sizes 320,375`
  (add `--email/--password` to cover authenticated routes).
- Commit: frontend `ba71f2d`. Not pushed; the live site still runs the old build.
- Follow-up commit `ac6b8d7` finished the pass: the sweep's label substitution now
  applies only to bare checkbox/radio inputs (applying it to text fields measured
  the label instead of the field and produced false positives), checkboxes are
  matched through `label[for]` as well as wrapping labels, and the auth shell
  brand + "Back to home" links reach 44px. Re-run: **8 touch findings** at
  320/375 (was 40), zero horizontal overflow.
- Remaining, all verified by the sweep rather than guessed: `/contact` email
  button (133x20), `/audits/<id>` "Contact Prospetra" (122x20) and a 32x32 icon
  link in the admin blog editor. The first two are text links inside content
  blocks (WCAG 2.5.8 exempts targets in a sentence); the editor icon link is a
  genuine fix.

### 2026-09-26 (o) — "Most popular" badge was clipped by its own card

- The highlighted plan card in `frontend/src/components/pricing/pricing-grid.tsx`
  carried `overflow-hidden` while its badge is positioned at `-top-3`, so half
  the pill was cut off on every breakpoint. Nothing in the card needs clipping:
  `evidence-grid` is a background only, and border-radius already clips
  backgrounds.
- Verified by hit-testing rather than by eye (screenshots don't composite in the
  preview webview): sampling the badge's top, middle and bottom with
  `elementFromPoint` returns the badge itself at both 1280 and 375, the card
  reports `overflow: visible`, and the badge clears the card above it by 9px
  when the cards stack.
- Commit: frontend `2717035`. Not pushed; the live site still runs the old build.

### 2026-09-26 (p) — Horizontal overflow at phone widths is now a CI gate

- The responsive sweep used to always `exit 0`, so it could only be run by hand —
  which is why the 29px `/admin/operations` and 86px blog-editor overflows were
  found by reading screenshots. It now exits 1 on a chosen set of statuses:
  - `pnpm test:responsive -- --base http://127.0.0.1:3100 --sizes 320,375 --anon-only --fail-on overflow`
  - `--fail-on` takes `overflow` (default), `touch`, `console`, `http`, `nav`,
    `all`, `none` (comma-separated) and rejects typos instead of silently
    gating nothing. `--overflow-tolerance` (default 2px) absorbs sub-pixel noise.
- Decision logic lives in `frontend/scripts/lib/sweep-ci.cjs` (fail-on parsing,
  blocking decision, report/exit path, Chrome resolution) so the gate is one
  small readable unit rather than logic buried in the sweep.
- Chrome is resolved from `CHROME_PATH`/`CHROME_BIN`, then the platform's default
  install paths, then Playwright's cached Chromium — a CI runner needs nothing
  beyond `playwright install chromium`. CI previously depended on a hard-coded
  Windows path.
- CI wiring: `.github/workflows/responsive.yml` sweeps the public routes at
  320/375 on its own runner; `ci.yml`'s stale-token job runs the same gate over
  the authenticated routes on port 3101 (the backend allows any 127.0.0.1 origin
  while `APP_ENV=local`, so CORS is satisfied) and exports `QA_EMAIL`/
  `QA_PASSWORD`, which the sweep now reads from the environment.
- **Undeclared dependency found**: `scripts/qa-responsive.cjs` required
  `puppeteer-core`, which was in neither `package.json` nor the lockfile — it
  only worked locally because someone installed it by hand. A
  `--frozen-lockfile` CI install would have left the gate without a driver. Now a
  devDependency (`^24.43.1`).
- Detector fixes while wiring the gate:
  - Content inside a horizontal scroller (code block, chip row, data table) is no
    longer counted as page overflow. The old scan flagged e.g. a 1080px `<code>`
    on `/resources/accessibility-badge` — a false positive that would have made
    the gate untrustworthy on day one.
  - A failing result now names the offending element by right edge, so triage
    from CI logs alone does not require reproducing the page.
  - The reference width is the viewport (`clientWidth`). Do **not** "fix" a page
    that reports a negative overflow: with `scrollbar-gutter: stable`, a page that
    merely fits reports ~-10px. That is content fitting inside the reserved
    gutter, not a hidden overflow, and normalising against the body width would
    flag legitimate fixed elements at the viewport edge.
- Proofs run for this change:
  - Green: `--anon-only --sizes 320,375 --fail-on overflow` → 38 checks, 0
    overflow, `PASS`, exit 0.
  - Red (gate path): forced with `--overflow-tolerance -100 --fail-on all` → exit
    1, 38 blocked results listed.
  - Red/green (detector): in a 320px viewport, a clean page measures 0; injecting
    a 380px element measures 71 and names the culprit; removing it returns to 0;
    900px of content inside a 200px `overflow-x: auto` scroller measures 0.
- Commit: frontend `93afff5`. Not pushed; the live site still runs the old build.

### 2026-09-26 (q) — Auth pages joined the marketing surface; the new gate repaired

- **Auth pages now use the marketing type scale and motion system.** login,
  register and forgot-password render `AuthShell`, not `PublicHeader`, so they sat
  outside `marketing-polish.css` and kept their own sizes (form title 30px where
  the equivalent marketing heading was 44px). The scope marker was renamed to
  `data-marketing-surface` and is now carried by both PublicHeader and AuthShell,
  so the whole public surface shares one scale, one easing token set and one set
  of phone tap-target rules. Two steps are declared where they legitimately differ
  (same file, documented): the form title takes the scale's h2 step, because a
  4rem title inside a 28rem card would swamp the form it labels, and the panel's
  display line (a `<p>`, which the element rules cannot reach) opts in with
  `data-display`. Their type utilities were removed from the markup so the
  stylesheet owns the scale.
  - Measured: auth h1 28px at 320/375 and 44px at 1440; display line 49px at 1440
    (was 48px); easing tokens present on the auth `<main>`; marketing h1/h2 still
    64/44px; auth card never gets `reveal`/`is-pending` (its `<section>` is
    first-of-type), so it cannot be hidden waiting for a scroll that never comes.
- **The overflow gate was repaired, and made stronger.** A mis-counted patch hunk
  silently dropped `const limit = de.clientWidth`, so every page threw
  `ReferenceError` inside `check()` and all 38 checks came back `NAV_FAIL` — while
  `report()` printed **PASS**, because `--fail-on overflow` had no opinion about
  pages that never loaded. Commit `93afff5` carried that state.
  - `--fail-on` now defaults to `overflow,http,nav`: a page that fails to navigate
    or returns 4xx/5xx is a failure of the gate's premise. Both workflows pass the
    list explicitly.
  - `report()` returns 1 when **no** check succeeded, so a wrong base URL, a
    stopped server or a missing Chrome fails loudly. Proven: pointing the sweep at
    a dead port now exits 1 with `no page loaded successfully (19 checks)` where it
    previously reported PASS.
- **Process lesson (third hunk-counting slip this session): an over-counted `+`
  line count makes `patch` consume the *following* line and silently drop the last
  added line — and `patch --dry-run --fuzz=0` does NOT catch it.** Verify applied
  content after every patch (`grep` for the lines you added, or run the script),
  not just `node --check`. This is exactly the failure the new gate guard exists
  to catch.
- Commits: frontend `478051d` (auth surface) and `31e9d67` (gate repair). Not
  pushed; the live site still runs the old build.

### 2026-09-26 (r) — "Forgot password?" alignment fixed; the reset email flow tested

- **Reported symptom, reproduced:** on `/login` the password label row is
  `flex items-center justify-between` and the "Forgot password?" link was the only
  shrinkable item. Below roughly 270px viewport width it shrank and wrapped
  *internally* to two lines ("Forgot" / "password?"), so it read as a stray line
  next to the field instead of a link beside the label. Measured at 259px before
  the fix: link 62x32, two lines, `sameLine: false`. Fixed with
  `shrink-0 whitespace-nowrap` on the link plus `gap-3` on the row; after: one
  line, 12px gap, no overflow at 259/309/375px, and the gate still reports 0
  overflow across the public routes. (The row cannot wrap by design, so a truly
  too-narrow viewport would now overflow and be caught by the gate rather than
  silently looking broken.)
- **The forgot-password email works end to end.** Exercised deliberately with
  `MAIL_MAILER=log` so nothing was emailed externally:
  - `AuthService::sendPasswordResetLink()` → `passwords.sent`; the log contains a
    rendered HTML email, `To: Audit QA <audit.qa.local@example.com>`,
    `Subject: Reset Your Password`, with the reset URL embedded; a matching row
    appears in `password_reset_tokens`.
  - Round trip: the token from that email was posted to `/auth/reset-password`
    (`{"success":true}`), the new password signed in successfully, and then the
    original password was restored and re-verified. The QA account is back to
    `AuditQa!2026x9`.
  - The flow already guards the queue risk: `CriticalMailer::queueHasNoWorker()`
    sends inline when no worker is running, so a reset link is never silently
    lost (`QUEUE_CONNECTION=database` locally).
- **Local-only config wart found:** `FRONTEND_URL=http://localhost:3000` builds the
  emailed link, but this machine's frontend dev server runs on `53367` (port 3000
  belongs to another project), so a locally-emailed reset link lands on the wrong
  app. `.env.example` carries the correct production values
  (`APP_URL=https://api.prospetra.com`, `FRONTEND_URL=https://prospetra.com`), so
  this is a local-testing annoyance rather than a deploy bug — set `FRONTEND_URL`
  to the dev port when testing an emailed link locally.
- **Not verified:** real SMTP delivery to a real inbox. The local `.env` points at
  Hostinger with credentials, but the only test account is on `example.com`, so
  only the generation/hand-off path could be proven without mailing a real
  address.
- Commit: frontend `466ff92`.

### 2026-09-26 (s) — The public heading scale and section rhythm are now a CI gate

- **What it enforces.** The public surface (marketing pages plus the auth shell)
  has one heading scale and one section rhythm, declared in
  `marketing-polish.css`. The responsive sweep now measures what each public page
  actually renders — every `h1`/`h2`/`h3`, `[data-form-title]`, `[data-display]`
  (font-size and leading) and every band that claims the rhythm — and compares it
  with an approved table in `scripts/lib/type-scale.cjs` at the swept viewport. A
  disagreement is a `DRIFT` result that names the element, the hook and the
  approved clamp; `drift` is now part of `--fail-on` in the responsive workflow.
- **The approved table is deliberately independent of the CSS.** h1 36→64px, h2
  28→44px, h3 18→22px, auth form title 28→44px, auth display line 36→56px, band
  padding 56→96px; leading 1.04/1.14/1.25/1.12/1.02; tolerance 1.5px (0.06 for
  leading). Editing the stylesheet to give one page a different size fails the
  build until the table is changed too, so the scale cannot drift one page at a
  time without a reviewable decision.
- **Section rhythm is judged by vocabulary, not by guessing which sections are
  bands.** A band claims the rhythm by carrying a hook: `py-16`, `py-20`,
  `section-py`, `section-py-lg`, `section-py-xl`. All five now resolve to one
  `clamp(3.5rem, 7vw, 6rem)` on the public surface, which retires the three named
  fixed steps in `globals.css` (3rem/4rem/5rem — four rhythms on one surface).
  Bands that hand-roll their own padding (heroes, cards, the auth shell) are
  reported in the run log as `free` and are not failed.
- **Two tripwires keep the table and the stylesheets from silently diverging**
  (`scripts/lib/type-scale.test.mjs` — vitest cannot be `require()`d from CJS,
  hence `.mjs`): the marketing rhythm rule's selector list must equal the hook
  list exactly, and every `@utility section-py*` in `globals.css` must be in that
  list, so a new named rhythm utility fails CI until the gate knows about it.
- **Red proof, then restored byte-for-byte.** h2 floor 1.75rem→2rem: 24/38 checks
  went DRIFT, e.g. `h2 32px (want 28px = clamp(1.75rem, 3.9vw, 2.75rem)) x9 on
  <h2 class="mt-4 max-w-sm …"> "See what your website is telling prospec"`.
  Rhythm ceiling 6rem→5rem: 9 checks reported `section padding 80px via
  .section-py-lg (want 96px = clamp(3.5rem, 7vw, 6rem))`. Adding a
  `@utility section-py-sm` failed the unit test with `expected [ Array(5) ] to
  include 'section-py-sm'`.
- **Verified green.** Phone sweep (320/375) `--fail-on overflow,http,nav,drift` →
  38 checks, 0 overflow, 0 drift, exit 0; desktop sweep (1024/1440)
  `--fail-on drift` → 0 drift, exit 0, resolving the mid-ramp values exactly
  (1024px: h1 63.5, h2 39.9, band 71.7). The run log prints the observed scale
  per viewport (`h1 64/1.04 | h2 44/1.14 | h3 22/1.25 | pad 96`). vitest 61/61
  (22 new), eslint 0 errors (same 3 pre-existing warnings), `tsc --noEmit` clean.
- **Workflow.** `.github/workflows/responsive.yml` job renamed `phone-overflow` →
  `public-surface`: the phone step now gates `overflow,http,nav,drift`, and a new
  desktop step (`--sizes 1024,1440 --fail-on drift`) reaches the clamp ceilings
  (h1 64, h2 44, h3 22, band 96) that phone widths never exercise. One artifact
  carries both reports (`responsive-reports`). The sweep's own default is now
  `--fail-on overflow,drift`, so an unflagged local run catches scale drift too.
  The stale-token job's sweep in `ci.yml` is unchanged — its failure surface
  stays about tokens.
- **Known boundary, and the next lever.** Bands that hand-roll padding
  (`pt-14 pb-12`, `pb-10 pt-14`, `pb-12 pt-16`, card and shell padding) show up in
  the log as `free 32…112` but are not judged; migrating them onto the vocabulary
  is a design pass, after which `drift` could judge every top-level band.
- **Incidental fix:** `globals.css` had been left with mixed line endings by an
  editing script (LF lines inside a CRLF file) — normalized back to CRLF; its diff
  is now only the comment above the section utilities.
- **Status:** committed as frontend `2c5bb87`; pushed to origin/main on 2026-09-26
  with the rest of the session backlog (ed449a9…2c5bb87). Still not deployed —
  the live site runs the pre-2026-09-17 build until the next frontend deploy.

- **Page restructured** (`src/app/audit/[slug]/page.tsx`): the scorecard is now
  the hero on the public type scale (h1 36px at 320/375, 64px at 1440,
  leading 1.04, `text-wrap: balance` from the shared rule) inside an
  `evidence-grid` band; the service-CTA, share and conversion bands became real
  top-level `<section>`s carrying the `section-py` hook, so they get the one
  rhythm (56px phone / 96px desktop) and the scroll reveal. MarketingReveal now
  acts on the page: 4 sections below the fold start `is-pending`, all reveal on
  scroll (verified: pending 4→0), and the hero is never hidden.
- **Motion tokens apply**: measured `--ease-out-quint` and `--motion-fast` on the
  body; the scorecard and share card carry `card-premium` (0.26s lift on the
  shared easing). The `ScoreRing` already counted up and animated its ring —
  unchanged.
- **Mobile fixes on the way:** the share row was `flex-wrap gap-2` with three
  `flex-1` buttons — "Share on LinkedIn"/"Share on WhatsApp" wrapped to two lines
  and the row was the old 320px overflow offender. It is now a 1-col grid on
  phones and 3 equal columns from `sm` (all buttons 44px tall, labels on one
  line). The "Contact Prospetra" inline link in `AuditServiceCtas` failed the
  44px touch floor (122x20) — given `inline-flex min-h-11 items-center`
  (122x44), the only change to that shared component.
- **Verified**: sweep with a real share token (`--share-token`, example.com
  report, score 75) at 320/375 → the share route is `ok` — 0 overflow, 0 touch,
  0 drift, and the route now reports its scale (`pad 56, h2 28/1.14, h1 36/1.04`);
  DOM probe at 320/375/1440 confirms rhythm 56/96, tokens present, reveal works,
  hit areas ≥44. vitest 61/61, eslint 0 errors (same 3 pre-existing warnings),
  `tsc --noEmit` clean.
- **Note:** the share route is only swept when `--share-token` is passed (needs
  a live token), so CI still covers the other 19 public routes; the page passes
  the same gates.
- **Status:** committed as frontend `f14b45e` and pushed to origin/main on
  2026-09-26. Still not deployed — the live site runs the pre-2026-09-17 build
  until the next frontend deploy.
