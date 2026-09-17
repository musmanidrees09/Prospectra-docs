# Prospectra Production Deployment

Updated: 2026-09-07

## Preconditions

- Back up the production database and uploaded storage before deploying.
- Use PHP 8.4, Composer 2, Node.js, and pnpm versions compatible with the lockfiles.
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

Deploy the generated Next.js application with its production start command or the hosting adapter already used by the project.

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
