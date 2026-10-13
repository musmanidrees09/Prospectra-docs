# Prospetra Blog Visual and SEO Upgrade

## Scope

Upgrade the public blog hub and its five current articles. This design treats
“all pages SEO” as the blog index plus the five published blog pages, not the
authenticated application or unrelated service pages.

## Safety rulings

- Regenerate the hero, form comparison, and checklist as new high-resolution
  editorial illustrations.
- Do not regenerate audit screenshots. Generative recreation could change
  scores, labels, or measurements and would turn evidence into a mockup.
  Instead, use deterministic enlargement, restrained sharpening, and a
  click-to-zoom presentation. If standalone original screenshots become
  available later, they should replace the collage crops.
- Preserve the redaction on the audited domain and never publish the supplied
  contacts panel.
- Keyword targets are based on observed search-result language and the supplied
  Search Console context. No search-volume numbers will be invented.

## Visual direction

Use Prospetra's white, warm-neutral, ink, and dark-green palette. The blog hub
will feel like an editorial resource library rather than a grid of oversized
text boxes.

- Add a restrained blog hero with an eyebrow, concise value proposition, and
  a compact “Run free audit” action.
- Feature the newest or priority article in a wide card with image, category,
  reading time, title, summary, and arrow action.
- Render remaining articles as equal-height cards with thumbnail media,
  smaller responsive typography, metadata, two-line title clamping, concise
  descriptions, generous whitespace, subtle borders, and visible focus/hover
  states.
- Use one column on phones, two on tablets and desktop. Avoid a dense
  three-column layout because titles are long.
- Add Blog to the shared public navigation before Contact and preserve the
  mobile menu's touch targets and keyboard behavior.

## Image outputs

New illustrative masters will be generated at 1536×768 or larger, inspected,
then exported to responsive WebP with stable dimensions. The three evidence
images will be enlarged to approximately twice their current dimensions with
high-quality resampling and restrained unsharp masking only. No text, score,
or diagnostic fact may change.

The final article keeps the same six filenames and URLs, so CMS records and
Open Graph metadata remain stable.

## Keyword and intent map

### Website accessibility audit

Primary: `website accessibility audit`.

Supporting: `WCAG 2.2 checklist`, `web accessibility audit checklist`,
`accessibility testing`, `how to audit a website for accessibility`, and
`manual accessibility testing`.

### Website audit checklist

Primary: `website audit checklist`.

Supporting: `how to audit a website`, `site audit checklist`, `technical
website audit`, `website health check`, `SEO speed security accessibility
checklist`.

### SEO audit guide

Primary: `how to do an SEO audit`.

Supporting: `SEO audit checklist`, `technical SEO audit`, `on-page SEO audit`,
`crawl and index audit`, `SEO audit tools`, `prioritize SEO issues`.

### Core Web Vitals

Primary: `Core Web Vitals`.

Supporting: `LCP INP CLS`, `how to improve Core Web Vitals`, `PageSpeed
Insights`, `web performance optimization`, `LCP 2.5 seconds`, `INP 200 ms`,
and `CLS 0.1`.

### Website health score

Primary: `website health score`.

Supporting: `site health score`, `website health check`, `technical website
health`, `improve website performance`, and `website audit score`.

These phrases will be integrated into useful headings, summaries, metadata,
and explanatory copy. They will not be dumped into keyword lists or repeated
unnaturally.

## Content changes

- Preserve all strong existing article material and URLs.
- Expand the four short seeded articles into substantive CMS content with a
  quick-answer section, practical steps/checklists, prioritization guidance,
  FAQs, authoritative references, and contextual links to `/free-audit`,
  `/website-fix`, and related guides.
- Keep the accessibility article's complete supplied editorial content.
- Give every article a unique title, description, H1, canonical, Article
  schema, Breadcrumb schema, and relevant related-article links.
- Keep claims conservative: automated scores do not certify WCAG conformance,
  Core Web Vitals are one input to page experience, and rankings are not
  guaranteed.

## Architecture

- Keep Laravel `blog_posts` as the content authority.
- Extend the existing article publisher/import pattern rather than hardcoding
  articles in Next.js.
- Add an idempotent publisher for the five reviewed articles, preserving
  existing author and publication dates.
- Normal database seeding may create missing reviewed articles but must never
  overwrite later editor-owned revisions.
- Keep the existing dynamic sitemap admission checks.

## Verification

- Visually inspect every generated/enlarged image and confirm evidence text is
  unchanged.
- Verify blog hub layouts at phone, tablet, and desktop widths.
- Verify header Blog links on public pages and mobile navigation.
- Confirm all five pages return crawlable server-rendered content with unique
  metadata, one H1, self-canonical, BlogPosting and BreadcrumbList JSON-LD.
- Confirm sitemap eligibility and internal links.
- Run frontend tests, type-check, lint, production build, backend tests, Pint,
  image decoding checks, and diff checks.

## Success criteria

- The blog hub no longer presents oversized, sparse cards.
- All five articles have clear, distinct search intent and substantial useful
  content.
- Illustrations are genuinely high resolution; evidence remains factual.
- Blog is reachable from the shared public header.
- No fabricated keyword volumes, audit evidence, ratings, certifications, or
  ranking promises are introduced.
