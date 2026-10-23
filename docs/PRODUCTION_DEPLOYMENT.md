# Prospectra Production Deployment

Updated: 2026-10-23

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

### Canonical production origin

The only indexable production origin is `https://prospetra.com`. Keep `http://prospetra.com`, `http://www.prospetra.com`, and `https://www.prospetra.com` reachable only as permanent redirect sources; do not remove the `www` DNS record or block these variants in `robots.txt`, because crawlers and old links must be able to reach the canonical URL.

The application has explicit 301 rules for HTTP apex and both `www` schemes, preserves the path/query, and emits only HTTPS-apex canonicals, internal links, structured data, robots sitemap references, and sitemap URLs. Hostinger's CDN can still run before Next.js. On 2026-10-23 the live behavior was:

| Request | Live result | Required result |
| --- | --- | --- |
| `http://prospetra.com/path` | one 301 to HTTPS apex | pass |
| `https://www.prospetra.com/path` | one 301 to HTTPS apex | pass |
| `http://www.prospetra.com/path` | 301 to HTTPS www, then 301 to HTTPS apex | one 301 directly to HTTPS apex |
| `https://prospetra.com/path` | 200 | pass |

To remove the remaining CDN-level chain, configure a **permanent 301 at the Hostinger edge** for `www.prospetra.com` to `https://prospetra.com`, preserving the path and query. In hPanel, use Websites → Dashboard → Redirects and select the `www` host as the source. If that rule cannot run before Hostinger's automatic HTTP-to-HTTPS upgrade, ask Hostinger support to apply the equivalent CDN/edge rule; changing Next.js cannot pre-empt a redirect already returned by the CDN. Do not disable the CDN or change DNS as a shortcut without a separate availability review.

Acceptance test: all three alternate origins must return `301` with an exact `Location: https://prospetra.com/<same-path>?<same-query>` in a single response, while the canonical origin returns `200`. The post-deploy smoke command below enforces this behavior.

## Post-deploy verification

### 2026-10-08 Hostinger SWC/config compatibility

The owner-provided build log for frontend `13fceda` failed while loading `next.config.ts`: native SWC required a missing `GLIBC_2.29` symbol, WASM was selected, and a generated config import was unresolved. The frontend configuration is now `next.config.mjs`, which Node loads directly without TypeScript/SWC config compilation. Existing settings are preserved and checked against the original resolved configuration. See [the compatibility report](../frontend/docs/HOSTINGER_CONFIG_COMPATIBILITY_2026-10-08.md) for evidence and the retry procedure.

Keep Node 22, pinned pnpm and `pnpm run build` (`next build --webpack`). A local full WASM-fallback build passes; production still needs a fresh checkout/build of the compatibility commit. The native GLIBC warning may remain until the hosting Linux image is upgraded, but the configuration no longer depends on native or WASM TypeScript transformation.

After each deploy:

    pnpm test:smoke https://prospetra.com

The smoke script gates one-hop canonical-origin redirects, security headers, the verify-email edge rule, the dashboard auth loop, and every sitemap URL (status, noindex, canonical). A failed item is a rollback signal, not a follow-up ticket.

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

The audit pipeline uses two dedicated queues. `audit-fast` publishes the deterministic initial report; `audit-enrichment` runs optional external providers and grounded AI after that report is visible. A worker that only consumes `default` or the legacy `ai` queue will not process audit jobs.

### Hosting capability gate

Do not assume that an hPanel website has Supervisor access. Hostinger documents port `65002` and home-directory-restricted SSH for Web/Cloud hosting, and documents cron as the supported scheduler there. A VPS provides the system-level control needed for Supervisor/systemd. Confirm the actual plan and process manager in hPanel before changing production; an SSH endpoint on port `65002` is evidence of Web/Cloud hosting, not proof that `sudo supervisorctl` is available.

The sub-30-second initial-report objective requires continuously supervised `audit-fast` consumers. A once-per-minute cron invocation can leave a new job waiting nearly a minute and is not an equivalent production configuration. If the current plan exposes only cron, use cron for Laravel's scheduler, but move the queue workers to a VPS or another persistent process service before declaring the rollout complete. Do not install a long-running `nohup` worker or overlapping cron-worker workaround without a separate resource and reliability review.

Hostinger references:

- [SSH access on Web/Cloud hosting is restricted to the account home directory](https://support.hostinger.com/en/articles/1583645-how-to-enable-ssh-access)
- [Hostinger Web/Cloud SSH uses port 65002](https://support.hostinger.com/en/articles/1583736-what-ports-are-open-at-hostinger)
- [hPanel cron job configuration](https://support.hostinger.com/en/articles/1583465-how-to-set-up-a-cron-job-at-hostinger)

Before installing process definitions, record the real application path and available manager from the production shell. Do not substitute the example `/var/www` path below until these checks pass:

```bash
pwd -P
test -f artisan
php -v
composer --version
command -v supervisorctl || command -v systemctl || true
git rev-parse HEAD
```

Run all three worker groups under Supervisor, systemd, or an equivalent process manager. The following Supervisor configuration is the production baseline; replace `/var/www/prospetra/backend`, `www-data`, and log paths for the host:

```ini
[program:prospetra-audit-fast]
process_name=%(program_name)s_%(process_num)02d
command=php artisan queue:work database --queue=audit-fast --sleep=1 --tries=1 --timeout=45 --max-time=3600
directory=/var/www/prospetra/backend
user=www-data
numprocs=2
autostart=true
autorestart=true
stopasgroup=true
killasgroup=true
redirect_stderr=true
stdout_logfile=/var/log/prospetra/audit-fast.log
stopwaitsecs=55

[program:prospetra-audit-enrichment]
process_name=%(program_name)s_%(process_num)02d
command=php artisan queue:work database --queue=audit-enrichment --sleep=1 --tries=1 --timeout=130 --max-time=3600
directory=/var/www/prospetra/backend
user=www-data
numprocs=3
autostart=true
autorestart=true
stopasgroup=true
killasgroup=true
redirect_stderr=true
stdout_logfile=/var/log/prospetra/audit-enrichment.log
stopwaitsecs=140

[program:prospetra-general]
process_name=%(program_name)s_%(process_num)02d
command=php artisan queue:work database --queue=ai,emails,default --sleep=2 --tries=3 --timeout=210 --max-time=3600
directory=/var/www/prospetra/backend
user=www-data
numprocs=1
autostart=true
autorestart=true
stopasgroup=true
killasgroup=true
redirect_stderr=true
stdout_logfile=/var/log/prospetra/general-worker.log
stopwaitsecs=220
```

For the database queue, `DB_QUEUE_RETRY_AFTER` must remain greater than the longest worker timeout. The production default of 240 seconds is valid for the 130-second enrichment worker. If Redis is selected instead, set `REDIS_QUEUE_RETRY_AFTER` to the same safe value.

After installing or changing the process definitions:

    sudo supervisorctl reread
    sudo supervisorctl update
    sudo supervisorctl restart prospetra-audit-fast:*
    sudo supervisorctl restart prospetra-audit-enrichment:*
    sudo supervisorctl restart prospetra-general:*

Do not use `queue:retry all` as recovery. Audit retries through the API reserve quota correctly; the scheduled audit recovery command handles jobs stranded before terminal state.

Run the Laravel scheduler once per minute (cron) or as a supervised `php artisan schedule:work` process:

    * * * * * cd /var/www/prospetra/backend && php artisan schedule:run >> /dev/null 2>&1

The scheduler performs four audit control-plane duties: dispatching pending audit outbox rows, dispatching due enrichment tasks, recovering stale work every five minutes, and sending queue heartbeat jobs every minute.

Monitor the pipeline with:

    php artisan audits:queue-health --json
    php artisan queue:failed
    php artisan schedule:list

`audits:queue-health` must report `healthy: true`, fresh positive heartbeat ages for both queues, no unexplained queue-depth growth, and zero stale pending audits. Alert when it exits non-zero, when `failed_jobs` grows, or when either required queue depth rises continuously. The recovery command is safe to inspect manually with `php artisan audits:recover-stale --dry-run`; the scheduler runs the bounded repair without `--dry-run`.

Required audit environment values are listed in `backend/.env.example`. Keep `AUDIT_SYNC_PROCESSING=false`; leave external providers and AI disabled until their explicit privacy/cost opt-ins and credentials are approved. `AUDIT_CORE_BUDGET_SECONDS=25` keeps the fast worker inside its 45-second process timeout under supported network conditions.

Deployment order for these migrations:

1. Back up the database and current environment.
2. Deploy backend code and run `composer install`.
3. Run `php artisan migrate --force` to add audit lifecycle fields, enrichment tasks, execution/quota reservations, and nullable editorial byline fields.
4. Apply the new environment values, then run `php artisan optimize:clear && php artisan config:cache`.
5. Install/update the three worker groups and the scheduler, then run `php artisan queue:restart`.
6. Run `php artisan audits:queue-health --json`; allow up to one scheduler minute for the first heartbeat.
7. Submit a controlled audit and confirm initial publication, background enrichment, history visibility, lead conversion, and Print/PDF before opening normal traffic.

The reviewed article sources are not applied by the migration. After the owner has supplied and approved the final Prospetra GSC case-study figures and named byline, take a database backup and run this explicit content command during the approved release window:

    php artisan blog:publish-reviewed-articles --all

The command updates only the five known slugs, validates every requested slug before the transaction, preserves each record's slug, publication date, author relation, named byline, bio, and reviewer, and does not create any additional article slug. Verify all five public URLs, title/H1 metadata, author box, reviewer line when present, related-article cards, and sitemap timestamps after publication.

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
