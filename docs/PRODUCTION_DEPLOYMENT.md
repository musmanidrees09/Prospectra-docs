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

## Post-deploy verification

After each deploy:

    pnpm test:smoke https://prospetra.com

The smoke script gates security headers, the verify-email edge rule, the dashboard auth loop, and every sitemap URL (status, noindex, canonical). A failed item is a rollback signal, not a follow-up ticket.

## 2026-09-29 build-failure post-mortem

A deploy failed with a Turbopack panic (`node process exited before we could connect to it` inside `parse_css` on `globals.css`). Cause: the box ran `npm install` (no `package-lock.json` in the repo), so `next: ^16.2.12` floated to the untested 16.3.6; CI built 16.2.12 with pnpm and stayed green. Fixes: `next` is now pinned exactly (no caret), `packageManager` pins pnpm, `preinstall` blocks non-pnpm installs, and this runbook is the authority for install commands.

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
