# Prospectra Codex Project State

Updated: 2026-09-17

## Current phase

The requested production-readiness implementation is complete in the repository. Deployment and real-provider activation remain operational steps that require production credentials, hosting access, and quota approval.

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
