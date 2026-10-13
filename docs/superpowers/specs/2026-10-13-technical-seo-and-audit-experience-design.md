# Technical SEO and Audit Experience Design

## Purpose

Create one consistent, indexable `https://prospetra.com` search presence, remove low-value and retired URLs from Google's indexable inventory, restore search-ready blog templates, and replace the customer-facing audit queue experience with immediate analysis progress.

Success means:

- every public canonical and sitemap URL uses the non-www HTTPS origin;
- HTTP and www requests permanently redirect in one hop at the edge, with an application fallback;
- utility, authentication, workspace, and admin pages are fetchable but `noindex, follow`;
- the sitemap contains only canonical, indexable, successful public pages;
- `/services/accessibility-audit-pakistan` is removed from navigation, content, and the sitemap and returns `410 Gone`;
- published blog pages emit accurate Article and BreadcrumbList structured data;
- a submitted authenticated audit immediately appears as analyzing and opens its progress page without user-facing queue language;
- technical SEO and audit behavior have automated regression coverage and deployment verification commands.

## Current State

The frontend already normalizes `NEXT_PUBLIC_SITE_URL`, generates self-canonicals for registered public pages, has dynamic robots and sitemap routes, emits global Organization/WebSite JSON-LD, and defines Next.js host redirects. Authenticated production audits are dispatched to a dedicated `ai` queue, while synchronous processing is restricted to local/testing environments.

Important gaps:

- auth, workspace, and admin metadata/header rules currently use `nofollow`;
- terms and privacy remain indexable;
- the retired Pakistan service is linked and included in public-page inventory;
- the dynamic blog route currently returns a branded 404 for every slug;
- global Organization schema does not yet contain all requested, verified profiles;
- the audit UI labels the durable background execution state as queued;
- application redirects are useful fallback protection, but the production edge must be the first redirect authority.

## Architecture

### Canonical origin and redirects

`siteConfig.url` remains the single frontend origin source and must resolve to `https://prospetra.com` in production. Public page metadata, Open Graph URLs, JSON-LD identifiers, sitemap URLs, and absolute internal URLs derive from it.

Cloudflare will be the authoritative production redirect layer:

1. redirect `http://prospetra.com/*` to `https://prospetra.com/*`;
2. redirect both HTTP and HTTPS `www.prospetra.com/*` directly to `https://prospetra.com/*`;
3. preserve path and query string;
4. return HTTP 301 and avoid a two-hop HTTP-to-HTTPS-to-non-www chain.

The existing Next.js redirect rules remain as defense in depth and receive regression tests. Deployment documentation will include equivalent Nginx and Apache snippets, but only the configuration matching the actual host should be installed.

### Crawl and index controls

A static `frontend/public/robots.txt` will replace the generated robots route so the file is always available as a simple 200 response. It will allow crawling, disallow only non-page API paths, and advertise `https://prospetra.com/sitemap.xml`.

Robots exclusions will not block auth or portal routes because crawlers must be able to read their `noindex` directive. Route metadata and `X-Robots-Tag` headers will consistently use `noindex, follow` for:

- login, registration, logout, password reset, and verification routes;
- terms and privacy;
- dashboard/workspace routes;
- admin/content-management routes;
- tokenized audit reports and service-request utility routes.

Canonical tags remain limited to indexable public pages. Non-indexable application pages do not need self-canonicals because they are not candidates for search indexing.

### Sitemap

The existing Next.js sitemap generator remains dynamic. It will emit only indexable public routes and published blog posts whose canonical resolves to the production non-www origin. Entries include evidence-based `lastModified`, `changeFrequency`, and `priority` fields.

The generator excludes auth, legal utility pages, portal routes, admin routes, tokenized reports, query variants, redirects, unpublished posts, the retired Pakistan service, and any backend-provided page marked noindex. Tests will reject off-origin, www, parameterized, redirected, or noindex entries.

### Retired service page

`/services/accessibility-audit-pakistan` will be removed from page inventory, footer navigation, the services page, the accessibility badge resource, and the sitemap. The route will return `410 Gone`, not redirect to an only loosely equivalent page. This explicitly reports intentional removal and avoids transferring users or signals to an irrelevant destination. Any useful accessibility language will link contextually to the free audit or website-fix page instead.

### Structured data and blog restoration

Organization schema stays globally available because it identifies the site publisher. It will use the existing fetchable logo and only real, verified social profiles. LinkedIn is already configured. Product Hunt, GitHub, and X will be added only when a verified canonical profile URL exists in configuration; missing profiles will be omitted instead of fabricated.

The published blog detail route will fetch its post through the existing public blog API, return 404 for missing/unpublished posts, sanitize/render the stored content through existing components, and generate:

- self-referencing canonical and matching Open Graph URL;
- unique title and description from approved post fields with safe fallbacks;
- Article JSON-LD with headline, description, canonical URL, image when present, published/modified dates, author, and the Organization publisher;
- BreadcrumbList JSON-LD for Home, Blog, and the visible article;
- visible breadcrumbs matching the structured data.

No ratings, reviews, or unsupported facts will be synthesized.

### On-page and internal-link improvements

Metadata for the three supplied blog topics will be aligned with their observed search intent:

- website accessibility audit: how to run, checklist, tools, and remediation;
- ADA website compliance: practical requirements, risk reduction, and testing limits;
- website speed test: how to test, interpret Core Web Vitals, and fix bottlenecks.

The implementation will prefer backend-managed post metadata where available, with code fallbacks for resilience. Contextual calls to action will connect informational articles to `/free-audit`, `/website-fix`, and `/pricing`. The accessibility badge resource will add a clear path from understanding the badge to running an audit and creating an account, without turning the footer into a keyword-link block.

### Audit submission experience

The HTTP request must not remain open while DNS, fetching, deterministic analysis, and optional external providers run. Holding the request open would expose users to browser, Cloudflare, PHP-FPM, and upstream timeouts and would make retries ambiguous.

Instead:

1. quota is reserved transactionally using the existing `PlanLimitService` rules;
2. the audit record is created and immediately moved to `analyzing` when work is accepted;
3. the API returns the audit resource immediately;
4. the frontend navigates directly to the audit detail page;
5. the detail page polls the existing show endpoint while status is analyzing and displays concrete progress copy;
6. the dedicated durable audit worker performs the pipeline and publishes completed, partial, or failed status;
7. retries use the same state transition and quota reservation rules.

The queue remains an internal delivery mechanism for reliability, but it is not presented as the user experience. Customer copy will say “Starting your audit” and “Analyzing the website now,” never “added to queue” or “queued for retry.” Operationally, production must run a dedicated worker for the `ai` queue with low concurrency appropriate to outbound audit limits. Monitoring will alert on stale analyzing records so a stopped worker is visible to administrators.

This design preserves DNS pinning, `SafeWebsiteUrl`, response size/type limits, provider isolation, and tenant-scoped authorization.

## Error Handling

- Redirect and canonical helpers reject credentials, unexpected origins, queries, and fragments where canonical URLs are expected.
- Blog API failure falls back safely for listings/sitemaps and returns an appropriate not-found or unavailable response for article routes rather than emitting invented content.
- Audit dispatch failure changes the audit to failed with a safe classified message rather than leaving it indefinitely analyzing.
- Existing partial-audit behavior remains intact when optional provider data is unavailable; missing evidence never becomes Passed.
- Stale analyzing audits remain recoverable through the existing admin retry workflow.

## Verification

Automated checks will cover:

- 301 redirect destinations for www and HTTP host/protocol combinations;
- one canonical per indexable page using the non-www production origin;
- `noindex, follow` metadata/headers for every utility route family;
- robots output and sitemap URL;
- sitemap exclusions and metadata fields;
- removal/410 behavior for the retired service;
- valid Organization, Article, and BreadcrumbList JSON-LD;
- immediate audit analyzing response, dispatch failure, retry, tenant authorization, quota reservation, and final state transitions;
- frontend navigation and progress copy without queue language.

Fresh completion checks will include backend tests and Pint plus frontend tests, type-check, lint, SEO validators, and a production build. Post-deployment cURL checks will inspect redirect hops, status codes, canonical/robots markup, robots.txt, sitemap entries, and the retired URL. Browser tools will validate rendered head tags, JSON-LD, live progress behavior, and internal links.

## Growth Measurement

The SEO rollout will be measured as a funnel rather than a ranking guarantee:

- canonical/indexable URL count and duplicate-host exclusions;
- sitemap submitted versus discovered URLs;
- branded query impressions, position, and CTR;
- page/query performance for the three target blog clusters;
- accessibility-badge organic landings to free-audit starts, registrations, and completed audits;
- audit submission-to-start latency, completion rate, failure rate, and stale-analyzing count.

The 14-day branded goal is a target, not a guaranteed outcome. Search Console annotations should mark deployment and content-update dates so subsequent changes are not attributed without evidence.
