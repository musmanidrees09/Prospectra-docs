# Prospectra Production Deployment

Updated: 2026-09-29

## Preconditions

- Back up the production database and uploaded storage before deploying.
- Use PHP 8.4, Composer 2, Node.js 22, and the pnpm version pinned in `frontend/package.json` (`packageManager`).
- **Frontend installs must use pnpm with the committed lockfile — `pnpm install --frozen-lockfile` from the `frontend/` directory. Never run `npm install` or `yarn`:** the repo has no `package-lock.json`, so npm resolves the caret ranges to the newest release. This is exactly how a deploy drifted onto an untested Next.js and failed its production build while CI stayed green (2026-09-29). A `preinstall` guard (`npx only-allow pnpm`) now aborts any non-pnpm install.
- Keep all credentials in the hosting environment. Never copy a development environment file to production.
- ADMIN_EMAIL may identify an existing administrator. AdminSeeder preserves that account's password hash.
- Set AUDIT_SYNC_PROCESSING=false and run a durable queue worker in production.

## Backend release

Run these commands from the backend directory:

    composer install --no-dev --prefer-dist --optimize-autoloader
    php artisan down --render="errors::503"
    php artisan migrate --force
    php artisan db:seed --class=PermissionSeeder --force
    php artisan db:seed --class=RoleSeeder --force
    php artisan db:seed --class=AdminSeeder --force
    php artisan storage:link
    php artisan optimize:clear
    php artisan config:cache
    php artisan route:cache
    php artisan view:cache
    php artisan queue:restart
    php artisan up

Never run migrate:fresh against production. Do not automatically run queue:retry all: audit retries must use the application retry endpoint so plan quota is reserved again.

## Frontend release

Set NEXT_PUBLIC_API_URL to the production API origin, then run from the frontend directory:

    pnpm install --frozen-lockfile
    pnpm build

The install must report pnpm and succeed silently on the frozen lockfile — if it prompts, warns about lockfile drift, or a `preinstall` guard error appears, stop: the checkout or the tooling on the box is wrong. Verify the built binary before starting it:

    pnpm list next --depth=0   # must print the version pinned in package.json

Deploy the generated Next.js application with its production start command or the hosting adapter already used by the project. If the host runs its own install step (dashboard builds, Git-push deploys), configure it to use pnpm with the lockfile — a host that defaults to npm will drift again.

### Hostinger Node.js deployment (hbuilds pipeline)

Hostinger's Git deployment pipeline runs `npm install` by default (then retries with `--legacy-peer-deps`), which the repo's `only-allow pnpm` preinstall guard now blocks — by design; the 2026-09-29 log shows exactly that block. Configure the deployment in hPanel → Websites → prospetra.com → Deployments → edit the build settings:

- Install command: `npx --yes pnpm@11 install --frozen-lockfile`
- Build command: `npx --yes pnpm@11 build`
- Start command: unchanged

`npx pnpm` needs no global install and satisfies the preinstall guard (pnpm's user agent is what `only-allow` checks). Do not remove the guard to let npm through — if the install/build command fields are not editable on the plan, the fallback is to drop the guard and commit a generated `package-lock.json` for determinism, which is a deliberate decision to record here first.

## Post-deploy verification

After each deploy:

    pnpm test:smoke https://prospetra.com

The smoke script gates security headers, the verify-email edge rule, the dashboard auth loop, and every sitemap URL (status, noindex, canonical). A failed item is a rollback signal, not a follow-up ticket.

## 2026-09-29 build-failure post-mortem

A deploy failed with a Turbopack panic (`node process exited before we could connect to it` inside `parse_css` on `globals.css`). Cause: the box ran `npm install` (no `package-lock.json` in the repo), so `next: ^16.2.12` floated to the untested 16.3.6; CI built 16.2.12 with pnpm and stayed green. Fixes: `next` is now pinned exactly, `packageManager` pins pnpm, `preinstall` blocks non-pnpm installs, and this runbook is the authority for install commands.

Resolution order for a repeat of the panic (2026-09-29, CI evidence attached):

1. **Prove the checkout is current.** The failing log showed npm installing 858 packages — impossible on code containing the `only-allow pnpm` preinstall guard. The box was building a stale checkout. Run `git fetch origin && git log --oneline -1` and require it to match `origin/main` before anything else.
2. **Install and build with pnpm** exactly as the Frontend release section says. CI's ubuntu build job is the reference: on commit `166d302` it built this same tree green on Linux with next 16.3.6, so a genuine fresh checkout has no known build failure.
3. **If Turbopack still panics on the box**, build with webpack: the panic occurred on a clean checkout (`b0a1289`, current pnpm pin) so `build` now runs `next build --webpack` by default (frontend `26508f4`); `build:turbopack` keeps the Turbopack path for local use. Report the panic log upstream, then treat the box's environment (memory limits, /tmp space, Node build) as the suspect.
4. Confirm what the box actually runs: `pnpm list next --depth=0` must print the version pinned in `package.json`.

## 2026-09-29 corepack pin mismatch (resolved)

After Hostinger switched to pnpm, the deploy failed with `[ERROR] This project is configured to use 11.15.1 of pnpm. Your current pnpm is v11.21.0`. Cause: pnpm invoked through corepack does not self-switch versions, so the box's corepack pnpm 11.21.0 hard-failed against the strict `packageManager` check. Fix: `packageManager` bumped to `pnpm@11.21.0` (frontend `b0a1289`). Local direct-pnpm invocations auto-switch to the declared version, so keep the pin aligned with the deploy box's corepack pnpm if Hostinger upgrades it again. The `only-allow pnpm` preinstall guard stays.

## 2026-09-29 runtime blank page: @swc/helpers not resolvable on the box (resolved)

The first green deploy (frontend `26508f4`) still served blank pages: every route returned 200 with 0 bytes and metadata routes (robots/sitemap) returned plain-text 500. Runtime log: `Cannot find module '@swc/helpers/_/_interop_require_default'` from inside next's own require stacks — the box's staged pnpm tree exposes the transitive package only via the virtual store, which Node cannot resolve there (local direct installs can hoist a top-level copy, masking it). Fix: `@swc/helpers` pinned exact `0.5.23` (the version next 16.3.6 declares) as a direct dependency so pnpm materializes a real top-level copy (frontend `991f668`). Diagnostic signature to remember: static `/_next/*` assets fine + dynamic routes empty-200 = the Node server crashes per-request while static file serving still works; check runtime logs before blaming the build.

Update (same day, frontend `0f2749c`): the direct-dependency pin alone was NOT sufficient — after it deployed (commit `932840e`, Completed/Current), the box failed identically. Root cause: Hostinger stages the built app into `versions/<id>/nodejs`, and pnpm's symlink-based tree does not survive that copy, so next-server cannot resolve packages even when a top-level copy exists. Fix: `nodeLinker: hoisted` in `pnpm-workspace.yaml` — pnpm builds an npm-style tree of real directories with zero symlinks, which is staging-proof. Verify after any change to dependency settings: `node -e "const e=require('fs').readdirSync('node_modules',{withFileTypes:true});console.log(e.filter(x=>x.isSymbolicLink()).length)"` must print 0.

## Queue and scheduler

Run a supervised worker and restart it on each deployment:

    php artisan queue:work --queue=ai,default --sleep=2 --tries=3 --timeout=210 --max-time=3600

Run the Laravel scheduler once per minute:

    php artisan schedule:run

The scheduler intentionally does not retry every failed job. AnalyzeWebsiteJob marks exhausted audits failed, and user-requested retries remain quota controlled.

## Audit provider configuration

- AUDIT_EXTERNAL_PROVIDERS_ENABLED is the global privacy and cost opt-in. It defaults to false.
- Enable only providers with valid credentials and acceptable quota: PageSpeed, CrUX, Open Page Rank, IPinfo, RDAP, Observatory, and Hunter.
- Hunter is a fallback after first-party extraction and must remain scarce-use.
- Cloudflare Radar is not a production dependency and remains disabled.
- Google Custom Search is unsupported by the current project and is not called.
- AUDIT_AI_ENRICHMENT_ENABLED is optional. When disabled or unavailable, deterministic audit facts and scoring still work.
- Provider cache durations are configurable independently. Do not lower them without reviewing quota and cost impact.

## Hostinger checks

- Point the API virtual host document root to backend/public, never the repository root.
- Give the web and worker processes write access only to storage and bootstrap/cache.
- Set APP_URL, FRONTEND_URL, CORS, Sanctum domains, secure cookies, database, cache, mail, and queue values for the real domains.
- Confirm outbound HTTPS and DNS are available for opted-in providers and public website audits.
- Use the hosting process manager for the queue worker. If unavailable, use a guarded cron worker that cannot overlap.

## Smoke test

1. Sign in as the existing administrator and confirm the password was unchanged.
2. Confirm the platform role choices are only Admin and User.
3. Promote a test user to Admin, then demote or delete only a non-current account.
4. Grant Blog Writer and confirm create/edit works while publish, archive, and delete remain blocked.
5. Run audits against one strong and one deliberately weak fixture and confirm scores and findings differ.
6. Confirm localhost, private IPs, unsafe redirects, oversized responses, and invalid content types are rejected.
7. Confirm queued audits move through pending/running to completed, partial, or failed.
8. Disable or invalidate one optional provider and confirm only that evidence is unavailable, never passed.
9. Repeat a provider-backed audit and confirm a cache hit rather than duplicate billable usage.
10. Confirm first-party contact evidence prevents a Hunter lookup.
11. Review application, worker, scheduler, mail, and web-server logs without exposing secrets.

## Rollback

Restore the previous application release and its matching environment configuration. If schema rollback is required, restore the pre-deployment database backup after assessing data written since deployment. The normalization and evidence migrations intentionally avoid unsafe automatic down operations.
