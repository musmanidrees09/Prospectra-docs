# Prospectra — Work Report

_Generated: 2026-08-31_

---

## 0. Read this first (honest status)

- **I did not commit or push anything.** Every change below is sitting **uncommitted** in your working tree (`frontend` repo, branch `main`).
- **All changes are in `frontend/`.** The `backend/` repo has **no uncommitted changes** (`git status` is clean).
- **I could not actually log in / test features end-to-end**, because your **local stack is not running** (details in §4). So "test every feature, check the dashboard updates" is **not done** — it's blocked on the environment, not on code.
- **Production backend is healthy** — I verified `api.prospetra.com` directly (see §3). The login "network error" is a **frontend-build / local-environment** problem, not a backend outage.

---

## 1. TL;DR — what the "network error" actually is

There are **two separate** "network error" situations. They look identical to a user but have different causes:

### A) Production (the reported bug: "login/register shows network error" on the live site)
**Root cause:** the deployed frontend bundle had `NEXT_PUBLIC_API_URL` baked in as `http://localhost:8001/api/v1`.
`NEXT_PUBLIC_*` values are compiled into the client JS **at build time**. The local URL had been set in `frontend/.env.local`, and Next.js loads `.env.local` in **every** environment — including `next build` — where it **outranks `.env.production`**. Result: the production bundle told **every visitor's browser to call `localhost:8001`**, which their machine refuses → axios reports a bare **"Network Error."**

**Status: FIXED in code (uncommitted).** See §2, cluster 1. Still needs a **rebuild + redeploy** to take effect on the live site (§5).

### B) Local (what you hit when testing on your own machine right now)
**Root cause:** the **local backend is not running** on port `8001` (connection refused), and **MySQL is not reachable** on `3306`/`3307`/`3308` (all refused) — even though MySQL 8.4.3 is installed via Laragon at `C:\laragon\bin\mysql\mysql-8.4.3-winx64\`.
So the browser has nothing to talk to → "network error."

**Status: NOT resolved — this is a runtime/environment issue, not a code bug.** Nothing to edit; the services need to be started and pointed at each other (§4, §5).

> ⚠️ **Discrepancy to confirm:** you said "even mysql is running," but a direct TCP test to `127.0.0.1:3306`, `:3307`, and `:3308` was **refused**, and no `mysqld` process was detected. MySQL may be stopped, bound to a named pipe only, or on a non-standard port. Please confirm it's actually listening on TCP `3306` (§5, step 1).

---

## 2. Files I changed and why

**Location:** all paths are under `D:\Personal Projects\Prospectra\frontend\`.
Totals: **22 files modified, 6 new files/dirs added, +554 / −126 lines.**

### Cluster 1 — Fix the production "Network Error" (the core bug)

| File | Change | Why |
|---|---|---|
| `next.config.ts` (+40) | Added `assertDeployableApiUrl()` that **throws during a production build** if `NEXT_PUBLIC_API_URL` points at localhost/127.0.0.1/0.0.0.0. Escape hatch: `ALLOW_LOCAL_API_IN_BUILD=1`. | Makes the bug **impossible to ship silently again**. A stray local URL now fails the build loudly instead of shipping and breaking every visitor. |
| `.env.development` (**new**) | Holds `NEXT_PUBLIC_API_URL=http://localhost:8001/api/v1` (+ app name/env/site URL). | Next.js loads this file **only for `next dev`**, never for `next build`. This is the correct home for local URLs. |
| `.env.local` (untracked, gitignored) | Removed the `NEXT_PUBLIC_API_URL` line; left a comment explaining why local URLs must not live here. | `.env.local` is loaded for production builds too and outranks `.env.production` — that's what caused the baked-in localhost URL. |
| `src/lib/axios.ts` (+24) | In the response interceptor, detect **"no response at all"** errors and return a human message ("Can't reach the server…" / timeout variant) with a **dev-only console hint** ("Is the backend running, and is `NEXT_PUBLIC_API_URL` correct?"). | Turns axios's opaque "Network Error" into something a user and a developer can act on. |

### Cluster 2 — Auth form accessibility / UX

| File | Change | Why |
|---|---|---|
| `src/app/(auth)/login/page.tsx` (+11) | Password show/hide toggle → 44px tap target (`size-11`), `aria-label`, `aria-pressed`, `title`, focus ring; icons `aria-hidden`. | Meets minimum touch-target + screen-reader guidance. |
| `src/app/(auth)/register/page.tsx` (+22) | Same treatment for **both** password and confirm-password toggles. | Same as above. |
| `src/components/auth/auth-shell.tsx` (+18) | Fixed heading hierarchy (marketing line demoted from `<h1>` to `<p>`; form title promoted `<h2>`→`<h1>`); made the "terms / privacy policy" footer text into **real links** to `/terms` and `/privacy`. | Correct single-h1 per page (SEO/a11y); wires up the new legal pages. |

### Cluster 3 — Error resilience (so failures aren't a blank page)

| File | Change | Why |
|---|---|---|
| `src/app/error.tsx` (**new**) | Route-level error boundary with a `reset()` retry button. | Unhandled render errors previously dropped users on React's bare screen / a blank page. |
| `src/app/global-error.tsx` (**new**) | Root error boundary; renders its own `<html>/<body>` with **inline styles**. | Only fallback if the root layout itself throws or the stylesheet fails to load. |
| `src/app/not-found.tsx` (**new**) | Branded 404 with links to audit/guides. | Replaces the default 404; `robots: noindex`. |
| `src/components/audit/audit-widget.tsx` (+27) | Distinguishes **network errors** (no response) from **API errors** (4xx/5xx JSON body) and shows a friendlier "temporarily unreachable" message for the former. | Same "network error" clarity, applied to the public audit tool. |

### Cluster 4 — SEO / marketing / content / legal

| File | Change | Why |
|---|---|---|
| `src/app/privacy/page.tsx` (**new**) | Full Privacy Policy page + metadata/canonical. | Legal page + linked from auth shell/footer. |
| `src/app/terms/page.tsx` (**new**) | Full Terms page. | Same. |
| `src/app/sitemap.ts` (+21) | Registered new routes (privacy, terms, etc.). | Keeps sitemap in sync for SEO. |
| `src/app/layout.tsx` (+1) | Metadata tweak. | SEO. |
| `src/components/shared/public-header.tsx` (+153) | Navigation restructure. | Marketing nav / links to new pages. |
| `src/components/shared/public-footer.tsx` (+183) | Footer restructure (legal links, sections). | Marketing footer / legal links. |
| `src/app/blog/[slug]/page.tsx`, `src/app/blog/page.tsx`, `src/content/blog/posts.ts`, `src/components/blog/blog-cta.tsx`, `src/components/blog/sticky-audit-button.tsx`, `src/components/seo/article-structured-data.tsx` | Blog + structured-data updates. | Content/SEO polish. |
| `src/app/contact/page.tsx`, `src/components/contact/contact-form.tsx`, `src/app/free-audit/page.tsx`, `src/app/pricing/page.tsx`, `src/app/website-fix/page.tsx`, `src/app/page.tsx` | Marketing-page copy/layout/form updates. | Content/SEO polish. |

---

## 3. What I verified (production backend is fine)

Direct requests to the live API (safe probes — nothing created):

- `POST https://api.prospetra.com/api/v1/auth/login` → responds correctly (rejects bad creds).
- `POST https://api.prospetra.com/api/v1/auth/register` with empty body → **HTTP 422** with proper validation JSON. ✅
- `GET https://api.prospetra.com/sanctum/csrf-cookie` → **HTTP 204**, sets `XSRF-TOKEN` + session cookie, and returns `Access-Control-Allow-Origin: https://prospetra.com`. ✅ (CORS is correct.)

**Conclusion:** the backend and its CORS config are healthy. The production login failure is on the **frontend bundle** side (cause A above), so redeploying the frontend fix should resolve it.

> I could **not** inspect the *currently deployed* frontend bundle — the in-app browser preview is sandboxed to `localhost` and won't load `prospetra.com`. So confirm the fix after redeploy by checking the Network tab (§5, step 5).

---

## 4. What is NOT done / still needs attention

1. **Nothing is committed.** 22 modified + 6 new files are uncommitted in `frontend`. (§5, step 3)
2. **Production not yet fixed live** — the code fix exists but must be **rebuilt and redeployed**. (§5, step 4)
3. **Feature testing not performed** — I never logged in or checked the dashboard/features, because the local stack is down. This still needs to be done once the environment is up.
4. **Local environment is not running / misconfigured:**
   - **MySQL** not reachable on `3306` (or `3307`/`3308`). Must be started + confirmed listening on TCP.
   - **Backend** not running on `8001`.
   - **Port mismatch:** the frontend expects the API on **`8001`**, but `composer dev` runs `php artisan serve` which defaults to **`8000`**. These won't line up unless you serve on 8001.
   - **Backend `.env` holds production values** (`APP_ENV=production`, `APP_URL=https://api.prospetra.com`, `SESSION_DOMAIN=.prospetra.com`). Cookie-based auth **will not work on localhost** with a `.prospetra.com` cookie domain. A local `.env` (or overrides) is needed to test locally.

---

## 5. Recommended next steps (exact commands)

**Step 1 — Start MySQL and confirm it's actually listening**
Start it from the Laragon UI (or run `mysqld`), then verify:
```bash
curl -s -o /dev/null --max-time 3 telnet://127.0.0.1:3306 && echo "3306 OPEN" || echo "3306 refused"
```

**Step 2 — Run the backend on the port the frontend expects (8001), with local settings**
The frontend points at `localhost:8001`. Serve there, and use local env values (either a local `.env` with `APP_ENV=local`, `APP_URL=http://localhost:8001`, `SESSION_DOMAIN=localhost` / blank, and DB pointed at your local MySQL; or override inline):
```bash
cd "D:/Personal Projects/Prospectra/backend"
php artisan migrate
php artisan serve --port=8001
```
(Alternatively, change `frontend/.env.development` to `:8000` and use the default `composer dev`. Pick one — just make both sides agree on the port.)

**Step 3 — Commit the frontend fixes** (only after you've reviewed them)
```bash
cd "D:/Personal Projects/Prospectra/frontend"
git add -A
git commit -m "fix(web): stop shipping localhost API URL in prod bundle; clearer network errors; a11y + legal pages"
```
`.env.development` is safe to commit (only public `NEXT_PUBLIC_*` values). `.env.local` stays gitignored.

**Step 4 — Rebuild + redeploy the frontend** so production stops calling localhost. With the guard in place, a bad build now fails loudly:
```bash
cd "D:/Personal Projects/Prospectra/frontend"
npm run build
```

**Step 5 — Verify the live fix after deploy**
On `https://prospetra.com/login`, open DevTools → Network, attempt login, and confirm the request goes to `https://api.prospetra.com/api/v1/auth/login` (not `localhost`).

---

## 6. File-change index (quick reference)

**Modified (22):** `next.config.ts`, `src/app/(auth)/login/page.tsx`, `src/app/(auth)/register/page.tsx`, `src/app/blog/[slug]/page.tsx`, `src/app/blog/page.tsx`, `src/app/contact/page.tsx`, `src/app/free-audit/page.tsx`, `src/app/layout.tsx`, `src/app/page.tsx`, `src/app/pricing/page.tsx`, `src/app/sitemap.ts`, `src/app/website-fix/page.tsx`, `src/components/audit/audit-widget.tsx`, `src/components/auth/auth-shell.tsx`, `src/components/blog/blog-cta.tsx`, `src/components/blog/sticky-audit-button.tsx`, `src/components/contact/contact-form.tsx`, `src/components/seo/article-structured-data.tsx`, `src/components/shared/public-footer.tsx`, `src/components/shared/public-header.tsx`, `src/content/blog/posts.ts`, `src/lib/axios.ts`

**New / untracked (6):** `.env.development`, `src/app/error.tsx`, `src/app/global-error.tsx`, `src/app/not-found.tsx`, `src/app/privacy/page.tsx`, `src/app/terms/page.tsx`

**Backend:** no changes.
