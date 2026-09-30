# Public Website Improvements Design

## Purpose and scope

Make Prospetra's public website clearly SaaS-first: a free, evidence-based
website audit and prospecting workflow for freelancers, small agencies and
consultants. Professional implementation services remain a separate,
secondary conversion path. Blog publishing is explicitly out of scope: no new
posts, no reactivation of blog navigation or sitemap entries, and no blanket
redirects for previous article URLs.

Success means a visitor can immediately understand the product category, run
an audit without being misled about coverage, see unambiguous software pricing,
and choose an appropriate service path when they need implementation help.

## Verified implementation baseline

- `backend/config/billing.php` is the authoritative runtime catalog: Free,
  Starter, Pro and Max; 3, 50, unlimited and unlimited AI requests/day;
  $0, $9.99, $29.99 and $99.99 monthly; 35% annual discount represented as
  monthly-equivalent annual prices.
- The public pricing page and Terms still use an obsolete Pro/Unlimited
  taxonomy, so they contradict that catalog.
- The contact component already POSTs to `/public/contact`; contact-page copy
  incorrectly describes a mail-client-only form.
- The sitemap deliberately excludes the empty blog shell. Keep that decision.
- The website-fix page contains unverified numeric before/after claims. The
  accessibility badge needs narrower automated-check wording.

## Design decisions

### Information architecture and navigation

Keep the current URLs. Public navigation prioritizes Free Website Audit,
How It Works (the substantive home-page workflow anchor), Pricing, Services,
Contact and Sign in, with a persistent `Run Free Audit` CTA. Services stay
below the product story and remain a grouped, secondary route family. Blog
and the accessibility resource are not promoted in main navigation until they
have substantive, accurate content.

### Homepage and audit funnel

The homepage metadata becomes `Website Audit & Prospecting Software for
Agencies | Prospetra`; the H1 and immediate support identify the audit,
evidence-based opportunity and pipeline workflow in plain language. The
primary CTA is `Run a Free Website Audit`, while account creation is an
explicit tertiary action labelled `Create Free Account`.

The audit page starts with the job visitors came to do: enter a public URL,
receive a free preview without registration, understand what the report
contains, and distinguish Website Score from Audit Coverage. A representative,
clearly labelled sample-report explanation replaces abstract technical-first
copy. The page will never say it proves WCAG conformance, real-user Core Web
Vitals, private Search Console data or checks unavailable provider data.

### Pricing, entitlements and payment

The frontend will consume the existing billing API/catalog representation
rather than maintain a second hard-coded set of plan facts. A small derived
copy helper may calculate and format annual total (`monthly price * 12 * .65`),
effective monthly value and activation language from those values. Cards,
feature matrix, FAQs, payment CTA parameters and Terms receive the same plan
taxonomy.

Until the owner changes commercial configuration, use the backend values above.
Paid CTAs say `Request [Plan] Activation`, not instant checkout. They explain
that activation is currently arranged by WhatsApp and avoid unverified
cancellation, tax, refund, turnaround or fair-use promises. Pro differentiates
from Max as priority support versus dedicated support, guided onboarding,
custom integrations and API access; no "most popular" label without evidence.

### Services and website-fix pages

The services hub begins with two paths: an existing website can be audited or
reviewed for help; a new website goes directly to a development quote. It uses
three concise, nonduplicated service cards, a scope-dependent engagement
process and a single quote CTA.

The website-fix page uses repair-specific metadata, an honest `Fix website
speed, usability and technical issues` H1, quote-first CTA and audit-secondary
CTA. Delete numerical customer-outcome examples and replace them with an
explicitly illustrative issue-to-fix walkthrough without invented outcomes.
Replace the SaaS outreach/pipeline sequence with diagnose, scope, approve,
repair and verify. Any timelines are stated as scope-dependent.

### Accessibility badge and contact

The badge can only acknowledge specific automated HTML/public-page signals
that completed. It will clearly state that it is neither WCAG conformance
certification nor a substitute for manual keyboard, contrast and assistive-
technology testing. It is not framed as a backlink programme.

The contact page presents the live direct submission flow, response/error
states and email/WhatsApp only as alternatives.

## Affected modules

- `frontend/src/app/page.tsx`, `free-audit/page.tsx`, `pricing/page.tsx`,
  `terms/page.tsx`, `services/page.tsx`, `website-fix/page.tsx`,
  `resources/accessibility-badge/page.tsx`, `contact/page.tsx`
- shared public navigation/footer and pricing components/content as required
- `backend/config/billing.php` only if an existing canonical API endpoint needs
  more display-safe plan data; no entitlement values change in this work.
- public sitemap metadata dates, only for changed indexable routes; blog stays
  excluded.

## Error handling, accessibility and tracking

Preserve existing direct form submission validation and failure fallback. Use
semantic sections/headings, visible focus, keyboard-accessible dropdowns and
44px touch targets. Reuse existing tracking for contact. Add audit funnel event
hooks only where the project already provides consent-aware tracking; do not
invent analytics collection or send personal data in event payloads.

## QA acceptance matrix

1. Pricing cards, FAQ, Terms and activation CTA agree with the backend catalog;
   annual figures calculate to the displayed 35% discount.
2. No remaining `Start Free Trial`, obsolete `Unlimited` plan naming, unverified
   outcome scores or full-WCAG claims on affected public routes.
3. Blog remains noindex/unlinked/absent from sitemap; prior URLs are not sent
   to the homepage.
4. Free-audit and accessibility language preserves evidence/coverage and
   automation boundaries.
5. Public routes pass lint, type checking, build, existing responsive sweep and
   relevant metadata/navigation checks.

## Non-goals and owner-confirmation boundary

This change does not publish blogs, make SEO ranking/traffic guarantees, add
testimonials, fabricate case studies, change paid entitlements, enable checkout
or alter legal billing/refund policy. A later change to prices, annual totals,
fair-use terms, tax, activation time or refunds requires owner confirmation and
a matching backend catalog update before public copy changes.

