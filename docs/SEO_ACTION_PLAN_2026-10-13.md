# Prospetra organic search action plan

## Diagnosis

- Search Console evidence supplied: about 460 impressions and 3 clicks over 50
  days; branded position about 11.2; accessibility articles at roughly 75–95;
  the accessibility badge resource has about 7.7% CTR at position 4.8.
- Observed in code: private routes used nofollow, both blog routes returned
  404, and edge redirect evidence showed mixed-host chains.
- First failing layers are canonicalization and content availability. Ranking
  work follows after those are fixed.
- No query export, backlink export, or verified Product Hunt/GitHub/X profiles
  were supplied. Keyword clusters below are hypotheses, not volume claims.

## 14-day branded entity sprint

No implementation can guarantee position 1 in 14 days.

Days 1–2: deploy the one-hop redirect, canonical sitemap, restored blog, and
Organization/WebSite schema. Verify Search Console ownership, inspect the
homepage and important commercial pages, and align every owned title/logo/name
to exactly Prospetra.

Days 3–5: complete real company profiles with the apex URL, same logo,
description, and contact data. Add only verified profiles to sameAs. Correct
legacy Prospectra spellings on owned profiles and citations.

Days 5–10: earn legitimate branded citations from partners, customers,
founders, relevant product directories, and communities. Prefer natural
anchors such as Prospetra or Prospetra website audit; avoid bulk links.

Days 10–14: compare branded impressions, CTR, device/country position, Google's
selected canonical, site name, favicon, and apex URL. Fix inconsistency rather
than repeatedly requesting indexing.

## Article overhaul

### /blog/website-accessibility-audit

Title: Website Accessibility Audit: Complete Checklist

Description: Learn how to audit accessibility, test WCAG issues, document
evidence, and prioritize fixes with a practical website checklist.

Candidate clusters: website accessibility audit checklist; how to audit a
website for accessibility; automated vs manual accessibility testing; WCAG 2.2
audit steps; accessibility remediation priority list.

### /blog/ada-website-compliance

Title: ADA Website Compliance: Practical 2026 Guide

Description: Understand ADA website accessibility, WCAG testing, common
barriers, and practical remediation steps. Not a substitute for legal advice.

Candidate clusters: ADA website compliance checklist; how to make a website
ADA compliant; ADA compliance for small business websites; WCAG and ADA
website requirements; test a website for ADA accessibility.

### /blog/website-speed-test

Title: Website Speed Test: Measure and Fix Slow Pages

Description: Test website speed, understand Core Web Vitals, find slow-page
bottlenecks, and prioritize fixes that improve real user experience.

Candidate clusters: how to test website loading speed; Core Web Vitals speed
test explained; why is my website loading slowly; mobile website speed test
checklist; improve LCP INP and CLS.

Compare actual GSC queries before editing. Put the answer near the top, use
original evidence, show tool limitations, expose author/update dates, and
update existing articles instead of publishing overlapping variants.

## Internal funnel and CRO

Use contextual paths:

    Accessibility or ADA guide -> free audit -> evidence -> registration
    Speed guide                -> free audit -> website fix
    All guides                 -> pricing when capacity is relevant
    Accessibility badge       -> free audit -> result -> registration

Suggested anchors: Check your site's observable accessibility evidence; Get
help implementing these fixes; Compare audit capacity for client work.

On the badge page, keep the responsible-use explanation, then present Run your
free accessibility check. Ask for registration after value is shown. Measure
badge landing to audit start, audit completion, registration, and first
authenticated audit.

## Measurement

Annotate deployment/content dates. Review canonical exclusions, submitted vs
indexed URLs, branded impressions/CTR/position, the three article cohorts, the
badge funnel, and audit submission-to-start/completion/stale rates. Prefer a
28-day comparison when traffic permits and do not infer causality from a short
ranking movement alone.
