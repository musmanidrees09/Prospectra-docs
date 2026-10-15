# Prospetra Blog Visual and SEO Upgrade — Implementation Plan

## Outcome

Deliver a polished, responsive blog library; expose Blog throughout public navigation; replace blurry accessibility illustrations with reproducible HD assets without altering factual screenshots; and publish substantial, intent-specific content for all five current guides through Laravel's CMS authority.

## 1. Shared public navigation

- Update `frontend/src/components/shared/public-header.tsx` to normalize caller-supplied navigation links, insert `/blog` exactly once immediately before Contact (or before Sign in/end when Contact is absent), and use the normalized list on desktop and mobile.
- Add Blog to mobile essentials and keep deduplication, active states, dropdown behavior, touch targets, and keyboard controls intact.
- Add focused tests for insertion order, deduplication, and nested blog-route active state.

## 2. Editorial blog hub

- Refactor `frontend/src/app/blog/page.tsx` into a server-rendered editorial hub with a concise hero, compact audit CTA, one featured article, and a responsive one/two-column grid for the remaining posts.
- Add reusable components under `frontend/src/components/blog/` for cards and crisp code-native article visuals. Use a valid featured image when available; otherwise use a deterministic, accessible vector/CSS visual keyed by topic with a neutral fallback.
- Use semantic heading levels that do not inherit the oversized global `main h2` marketing style, equal-height cards, readable metadata, clamped hub copy, visible keyboard focus, and no nested interactive elements.
- Add component/pure-helper tests where practical and verify 375px, 768px, and 1440px layouts in a browser.

## 3. HD accessibility imagery

- Generate three separate 1536px-or-larger Prospetra-branded editorial masters for the hero, form-label comparison, and accessibility checklist using minimal/no generated text.
- Inspect each generated master before adoption and retain reproducible source masters under `frontend/assets/blog/website-accessibility-audit/`.
- Update `frontend/scripts/process-accessibility-audit-images.py` to export the same six public WebP filenames. Export illustrations from their HD masters; enlarge the three factual screenshot crops with Lanczos resampling and restrained sharpening only.
- Preserve all factual scores, labels, measurements, domain redaction, and source crop boundaries. Never generatively recreate evidence.
- Update intrinsic image dimensions in `backend/app/Services/Blog/ReviewedBlogArticlePublisher.php` (and compatibility publisher behavior), plus article hero rendering so the browser reserves the correct aspect ratio.
- Add deterministic image validation for decodeability, expected dimensions, and minimum size; update deployment documentation.

## 4. Reusable reviewed-article publishing pipeline

- Extract Markdown parsing from `backend/app/Services/Blog/AccessibilityAuditArticlePublisher.php` into `backend/app/Services/Blog/BlogMarkdownParser.php`, accepting an optional safe image-marker map.
- Add `backend/app/Services/Blog/ReviewedBlogArticlePublisher.php` with a catalog for all five articles. It will validate sources before publication, preserve existing author and publication date, update only reviewed editorial fields on explicit publish, and provide `publish`, `publishAll`, and `publishMissing` operations.
- Keep `AccessibilityAuditArticlePublisher` as a backward-compatible wrapper for `website-accessibility-audit`.
- Add an explicit `blog:publish-reviewed-articles` Artisan command that accepts selected slugs or `--all`; reject ambiguous no-argument runs.
- Simplify `backend/database/seeders/BlogPostSeeder.php` to create missing reviewed articles only, ensuring deployment seeding never overwrites admin edits.

## 5. SEO-focused article sources

- Preserve the complete accessibility article and add reviewed Markdown sources for:
  - `website-audit-checklist`
  - `seo-audit-step-by-step`
  - `core-web-vitals-speed-score`
  - `website-health-score`
- Give each guide a distinct query intent, concise answer, practical workflow/checklist, prioritization guidance, pitfalls, FAQs, official references, useful related-guide links, and contextual paths to `/free-audit`, `/website-fix`, and `/pricing`.
- Apply the approved keyword map naturally in metadata and useful headings. Do not invent search volumes, certifications, audit facts, or ranking guarantees.
- Keep titles/descriptions unique and concise, retain existing canonical/Article/Breadcrumb handling, and interlink all five topic clusters.

## 6. Automated verification

- Backend tests: parser behavior; all five publisher definitions; idempotence; preserved author/date; explicit refresh; create-only seeding; selected/all command behavior; API/schema regression.
- Frontend tests: header normalization/deduplication/active state, blog-card fallback/image behavior, and sitemap regressions.
- Run `php artisan test`, `vendor/bin/pint --test`, frontend unit tests, `pnpm type-check`, `pnpm lint`, and `pnpm build`.
- Decode and dimension-check every public WebP. Visually inspect factual screenshots against their source crops and generated illustrations at native size.
- Browser-check `/blog` plus all five article routes at phone/tablet/desktop widths; verify one H1, unique metadata, self-canonical, BlogPosting and BreadcrumbList JSON-LD, crawlable server-rendered copy, commercial/internal links, sitemap admission, and desktop/mobile Blog navigation.

## 7. Deployment handoff

- Document the explicit reviewed-content publication command and post-deploy cache/storage steps.
- Provide production cURL checks for status, canonical, robots, sitemap, and redirect behavior. Treat local completion as deployment-ready, not already deployed.
