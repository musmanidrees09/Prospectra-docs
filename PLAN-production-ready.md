# Prospectra — Production-Ready Execution Plan

**Goal:** Take the current ~35% complete project to a deployable v1.0 (green tests, demo data, full feature UI, SEO content, CI/CD, deployment runbook).

**Current state (evidence, audited Aug 13):**
- Backend: 189 PHP files, real layered architecture (controllers → services → repositories → policies → resources), AI layer (4 agents, 3 providers, crawler), 27 test files. **102 pass / 20 fail.**
- Frontend: landing + auth pages + dashboard shell. **All dashboard numbers are hardcoded mocks; feature routes (`/leads`, `/audits`, `/emails`, `/proposals`) don't exist → 404.** Only `auth.service.ts` exists; it calls backend routes that don't match the API (`GET /auth/user`, `PUT /auth/profile`, `PUT /auth/password` — none exist in `backend/routes/api.php`).
- Data: `DatabaseSeeder` seeds only roles/permissions/super-admin. **No demo data.** 5 factories missing.
- SEO: `sitemap.ts` has 1 URL; no blog/docs/content pages. Landing page is polished.
- Infra: no CI, no Docker, no deploy config. Frontend deps not fully installed; `@types/react ^18` vs React 19 mismatch; both `package-lock.json` and `pnpm-lock.yaml` present.
- Brand: 3 names in play — folder `Prospectra`, docs `Prospetra/ClientHunterAI`, frontend `IntentQuanta`. **Decide one before Phase 4.**

---

## Phase 0 — Green backend build (Day 1)

1. **Add 5 missing factories** (fixes ~17 test failures). Mirror `database/factories/LeadFactory.php`/`CompanyFactory.php`:
   - `database/factories/EmailFactory.php`, `AuditFactory.php`, `ActivityFactory.php`, `ContactFactory.php`, `OpportunityFactory.php` — fields must match migrations (`2026_07_31_100003/4/5`).
2. **Fix `tests/Unit/Services/CompanyServiceTest.php`** (line 48): add `use function Pest\Laravel\assertDatabaseMissing;`.
3. **Fix Proposal `public_id` failure**: replace manual `creating`-event UUID generation with `Illuminate\Database\Eloquent\Concerns\HasUuids` trait on all models that use `public_id` (`Proposal`, `Email`, `Audit`, `Activity`, `Contact`, `Company`, `Lead`, `AiTask`, `Opportunity`). Remove the `booted()` UUID closures. This removes the `Event::fake()` fragility (root cause of `ProposalServiceTest` failure).
4. **Fix `tests/Unit/Services/UserServiceTest.php`**: capture the pre-update hash from a fresh query before calling `updatePassword()` (the service mutates the passed instance in memory; password hashing itself is correct — `User` model has the `hashed` cast, `Hash::check` in login).
5. **`composer.json`**: set real `name`/`description` (currently untouched `laravel/laravel` skeleton). Rewrite `backend/README.md` (still Laravel boilerplate).
6. **Verify:** `php artisan test` → **all green** (currently 122 tests); `vendor/bin/pint --test` clean.

## Phase 1 — Demo data (Day 2–3)

7. **New `database/seeders/DemoSeeder.php`** (register in `DatabaseSeeder::run()`): 1 organization, 1 admin + 1 demo user, ~20 companies, ~50 leads across statuses/sources with scores, 10 audits with `AuditFinding` rows (scores 40–90), 5 proposals, 15 emails, CRM contacts/opportunities/activities, timestamps spread over the last 90 days so analytics charts have data.
8. Reuse the new factories; keep it idempotent (`firstOrCreate` / wipe-and-reseed).
9. **Verify:** `php artisan migrate:fresh --seed` on MySQL (set `DB_*` in `.env`); API returns non-empty lists.

## Phase 2 — Frontend foundations (Day 4–6)

10. **Package manager**: adopt pnpm (workspace + `engines` say pnpm). Delete `frontend/package-lock.json`, run `pnpm install`, `pnpm type-check`, `pnpm build` until clean. Fix `@types/react ^18 → ^19` and `@types/react-dom` in `frontend/package.json`.
11. **Align auth contract** (frontend `auth.service.ts` ↔ `backend/routes/api.php`):
    - Frontend fixes: `getCurrentUser` → `GET /auth/me`; `updateProfile` → `PATCH /users/me`; `verifyEmail` → path params `GET /auth/verify-email/{id}/{hash}`; drop `GET /auth/user`, `PUT /auth/profile`, `PUT /auth/password`.
    - Backend additions: `POST /auth/password` (uses existing `UserService::updatePassword` + validate current password), `POST /auth/email/verification-notification` (resend, rate-limited).
12. **New API service modules** following the `auth.service.ts` pattern (axios + `ApiResponse`): `frontend/src/services/{lead,company,audit,proposal,email,crm,analytics,settings}.service.ts`.
13. **Types**: `frontend/src/types/{lead,company,audit,proposal,email,crm,analytics}.ts` mirroring backend `app/Http/Resources/*.php` field names.
14. **Auth flow wiring**: `auth-provider.tsx` loads `/auth/me` on mount; verify `middleware.ts` cookie check (`laravel_session`) works with the backend session config; document `SANCTUM_STATEFUL_DOMAINS` + `SESSION_DOMAIN` for `localhost:3000` and production.

## Phase 3 — Feature UI (Day 7–21, the core)

Build each as list + detail + create/edit form (react-hook-form + zod mirroring backend Form Requests), with loading/empty/error states and react-query hooks. Add routes under `frontend/src/app/(dashboard)/…` and entries in `components/navigation/navigation-config.ts`.

15. **Leads** (`/leads`): table with filters (status, search, assigned_to, company_id — backend supports), create form (POST `/leads`), detail with audit/proposal/email shortcuts, lead-score display.
16. **Companies** (`/companies`): list + detail + create.
17. **Audits** (`/audits`) — **the differentiator, do first**: URL input → `POST /audits` → poll `GET /audits/{id}` → score + findings list rendered from `AuditFindingResource`. Verify backend `AuditService`/`AnalyzeWebsiteJob` wiring and hook up.
18. **Proposals** (`/proposals`): list, create, AI generate (`POST /proposals/{id}/generate`), status workflow (draft → sent → viewed/accepted/rejected), PDF/export if cheap (else mark future).
19. **Emails** (`/emails`): list, create, AI generate (`POST /emails` + generation), send (`POST /emails/{id}/send`), status filters.
20. **CRM** (`/crm/contacts`, `/crm/opportunities`, `/crm/activities`): basic tables + create; activities feed.
21. **Analytics** (`/analytics`): replace **all hardcoded dashboard mock numbers** with `GET /analytics/overview`, `/analytics/leads`, `/analytics/conversions`; activity feed from activities API.
22. **Settings** (`/settings`): profile (PATCH `/users/me`), password (POST `/auth/password`).
23. **Verify:** full happy path in browser — register → login → dashboard shows real data → create lead → run audit → generate proposal → send email.

## Phase 4 — SEO & content (Day 22–28)

24. **Brand decision (from naming research) — pick ONE name** and apply to `frontend/src/constants/index.ts` (`APP_NAME`), `frontend/src/lib/site.ts` (`url`, keywords), `manifest.ts`, metadata, `composer.json`, docs.
25. **Landing page**: add pricing section (matches backend plans), FAQ with `FAQPage` JSON-LD, testimonials/proof, product screenshots, footer links.
26. **New content pages** with per-page metadata + OG images: `/blog` (index + ≥3 keyword-targeted posts), `/docs`, `/use-cases`, `/alternatives`, `/pricing`, `/privacy`, `/terms`.
27. **`sitemap.ts`**: include every public route (currently 1 URL). Keep `robots.ts`; extend `structured-data.tsx` per page.

## Phase 5 — Production hardening (Day 29–35)

28. **CI** (none exists): `.github/workflows/backend.yml` (composer install, `pint --test`, `php artisan test`) and `frontend.yml` (pnpm install, type-check, lint, build).
29. **Docker**: `docker-compose.yml` (mysql, redis, backend, frontend, nginx) + `Dockerfile` for backend and frontend + `nginx.conf` (reverse proxy, gzip, static cache). Add `docs/deployment/PRODUCTION.md` runbook (Vercel frontend + VPS backend per `docs/`; env table; `migrate --force` + seed on deploy; queue worker/supervisor).
30. **Mail**: configure `RESEND_API_KEY`; test register→welcome, forgot-password, verify flows end-to-end (dev: mailtrap).
31. **Observability**: daily log channel or Sentry (`sentry/sentry-laravel`); add `LOG_*`/`SENTRY_*` to `.env.example`.
32. **Security pass**:
    - `SESSION_DRIVER=cookie`, `SESSION_SECURE_COOKIE=true` behind HTTPS; `SANCTUM_STATEFUL_DOMAINS` + `FRONTEND_URL` set.
    - `config/cors.php` allowed origins = `FRONTEND_URL`; TrustProxies for nginx in `AppServiceProvider`.
    - **Rate limits beyond auth**: add `throttle:` to `leads`, `audits`, `proposals/*/generate`, `emails/*/send` (paid AI features — highest abuse risk).
    - Re-run `tests/Security/MultiTenantIsolationTest`; verify 419/401 handling in axios interceptor.
33. **Final verification (release checklist):**
    - `php artisan test` 100% green · `pint --test` clean · `pnpm type-check` + `pnpm build` + `pnpm lint` clean.
    - `/sitemap.xml` lists all routes; each page has unique title/description/OG.
    - E2E: register → verify email → login → dashboard real data → lead → audit → proposal → email.
    - Domain (see naming research) registered, DNS + HTTPS live, `.env` production values set.

---

## Critical files
- Backend: `database/factories/*` (5 new), `database/seeders/DatabaseSeeder.php` (+ new `DemoSeeder`), `app/Models/{Proposal,Email,Audit,Activity,Contact,Company,Lead,Opportunity,AiTask}.php`, `app/Services/User/UserService.php`, `routes/api.php`, `composer.json`, `tests/Unit/Services/{CompanyServiceTest,UserServiceTest,ProposalServiceTest}.php`
- Frontend: `src/services/auth.service.ts` (+ 8 new), `src/lib/axios.ts`, `src/app/(dashboard)/**` (new pages), `src/components/dashboard/dashboard-overview.tsx`, `src/components/navigation/navigation-config.ts`, `src/constants/index.ts`, `src/lib/site.ts`, `src/app/sitemap.ts`, `package.json`, `middleware.ts`
- Infra (new): `.github/workflows/*.yml`, `docker-compose.yml`, `Dockerfile*`, `nginx.conf`, `docs/deployment/PRODUCTION.md`

## Naming (checked Aug 13 via DNS-over-HTTPS NS lookup; NXDOMAIN = unregistered)
Available `.com` shortlist — **pick one and register immediately** (availability changes daily):
1. **prospetra.com** — matches folder + docs; respelling of taken `prospectra.com`; memorable, brandable.
2. **winstria.com** — short, premium feel (win + -stria); strong logo/wordmark potential.
3. **intentquanta.com** — already the frontend brand; fits intent-data positioning.
4. **winloom.com** — win + loom (weave wins); softer brand.
Checked & taken: prospectra, prosparo, leadient, winsight, closera, signalo, leadlens, quantle, outboundly, hunterly, scoutly, leadvy, pipelens, prospectiva, leadnova, clientium, winweave, leadweave, prospark, leadnook, winscale, dealmuse, clientara, leadara, leadera, intentify, winorbit, intentlens, dealvista, sellora, nurturely, winstem, quantora, winqube, winvault, leadvault, prospendo, winely, leadorbit, leadpilot, winlyra, salestel, winaq, leadaxy, winpilot, leadwisp, winova, dealara, leadvera, winlume, funnnelly, leadspire, dealqube, prospell, dealora, leadquanta, leadspire (taken or parked-for-sale).
