# Reliable Two-Stage Audit Pipeline Implementation Plan

## Decisions and consistency resolutions

- Keep the existing primary audit statuses: `pending`, `analyzing`, `partial`, `completed`, and `failed`.
- Define `completed_at` as the first usable deterministic report publication time. Track optional work separately with `initial_completed_at`, `enrichment_started_at`, and `enrichment_completed_at`.
- Reuse the same failed audit row on user retry. Re-reserve quota under the existing organization lock, clear all prior lifecycle/task state atomically, and do not create duplicate history rows.
- Never reset automatic dispatch attempts to zero. Pending recovery is bounded by both attempts and total age, then fails safely with `queue_unavailable`.
- Treat `audit_dispatches.dispatched_at` only as enqueue acknowledgement. Actual execution proof is the Stage-A claim/heartbeat plus a dedicated fast-worker heartbeat.
- Add `waiting` to enrichment task states so the AI task can be durably created with the initial report but cannot run before provider merging is terminal.
- Expose only aggregate enrichment state/counts and customer-safe text. Do not expose queue names, provider task names, credentials, raw errors, or exception bodies.
- Preserve the existing AI provider chain and fallback behavior: Groq, Gemini, Mistral, Cloudflare Workers AI, OpenRouter, and Claude. The repository has Groq rather than an xAI/Grok adapter; no existing provider is removed or replaced.
- Keep AI important to the final report: it enriches executive summary, recommendations, and outreach from persisted real evidence after deterministic/provider work. It cannot alter checks, score, or coverage.
- Add no new third-party package and make no production/infrastructure change during development.

## Milestone 1 — Schema, models, and safe API lifecycle contract

Add one forward-only migration that:

- adds nullable `initial_completed_at`, `enrichment_started_at`, and `enrichment_completed_at` plus unsigned `report_revision` defaulting to zero on `audits`;
- creates `audit_enrichment_tasks` with audit foreign key, `kind`, `phase`, state, normalized JSON result, safe reason code, attempts/dispatch attempts, availability/dispatch/claim/heartbeat/finish timestamps, unique `(audit_id, kind)`, and recovery indexes.

Add `AuditEnrichmentTask` with constants, casts, relationship, and terminal/claimable scopes. Extend `Audit` casts/fillable and add its tasks relationship. Extend factories only where tests need deterministic lifecycle data.

Update `AuditResource` to return:

- lifecycle timestamps;
- integer `report_revision`;
- safe aggregate `enrichment: { state, completed, total, stage_text }` with backward-compatible `null`/completed behavior for legacy audits.

Files:

- `backend/database/migrations/2026_10_21_000001_add_two_stage_audit_pipeline.php`
- `backend/app/Models/Audit.php`
- `backend/app/Models/AuditEnrichmentTask.php`
- `backend/app/Http/Resources/AuditResource.php`
- relevant factories and tests

Verification:

- migration up/down on SQLite test database;
- unique task constraint and casts;
- legacy and active-enrichment resource shapes;
- tenant-scoped audit access remains unchanged.

## Milestone 2 — One deterministic report assembler

Extract report derivation from `DeterministicPageAnalyzer` into a reusable `AuditReportAssembler`. It accepts checks and the stable report sections, then derives through existing services:

- score and coverage via `AuditScoreCalculator`;
- category summaries and metrics;
- warning/failed findings;
- analyzer summaries;
- opportunities via `LeadOpportunityEngine`;
- report summary.

`DeterministicPageAnalyzer` continues to build core checks, crawl evidence, first-party business/contact intelligence, pending provider checks, and AI narrative placeholders, then delegates final derivation to the assembler.

Add an enrichment merger that:

- keys checks by stable check ID;
- replaces only checks owned by the completed provider task;
- merges Hunter contacts only after existing provenance filtering;
- merges provider state summaries internally;
- invokes the same assembler and transactional persistence path;
- increments `report_revision` only for a material report update.

Split `AuditService::complete` into guarded `publishInitial` and `publishEnrichment` operations while retaining one internal finding-persistence implementation. Fire `WebsiteAuditCompleted` only for the initial publication so notifications/activity/lead workflows are not duplicated.

Files:

- `backend/app/Services/Audit/DeterministicPageAnalyzer.php`
- `backend/app/Services/Audit/AuditReportAssembler.php`
- `backend/app/Services/Audit/AuditReportEnrichmentService.php`
- `backend/app/Services/Audit/AuditService.php`
- `backend/app/Services/Audit/AuditScoreCalculator.php` only if an existing public seam is required
- focused unit tests

Verification:

- Stage-A pending provider checks remain `not_tested`;
- unavailable/missing evidence never passes;
- provider merge changes only its owned check;
- score/coverage/findings/opportunities all match the existing calculator rules;
- initial event fires once and enrichment does not duplicate it.

## Milestone 3 — Bounded core collection and fast initial job

Introduce a small monotonic `AuditDeadline` value/service with a default 25-second core budget. Pass it through `WebsiteTool::analyze`, crawl, and fetch:

- clamp connect/request timeouts to remaining budget;
- revalidate every URL and redirect with `SafeWebsiteUrl`;
- retain DNS pinning, redirect, payload, and content-type limits;
- require the homepage;
- skip optional extra pages, robots, or sitemap when insufficient time remains;
- record skipped work as untested/coverage loss rather than pass/fail invention.

Create `RunInitialAuditJob` on `audit-fast`:

- atomically claim only `pending` audits;
- set `starting`, `collecting_core`, and `finalizing_initial` with heartbeats;
- call `WebsiteTool` with external providers disabled;
- classify required-core failures safely;
- atomically publish the initial report and durable enrichment tasks;
- set `completed_at` and `initial_completed_at` and transition to `partial` or `completed`;
- trigger enrichment dispatch only after the transaction commits.

Update `AuditDispatchService` to enqueue `RunInitialAuditJob`. Retain `AnalyzeWebsiteJob` temporarily as a guarded compatibility handoff that dispatches the new job only for a still-pending audit; it must never execute the old monolithic pipeline.

Remove the obsolete private `AuditController::process` path and unused old-pipeline imports.

Files:

- `backend/app/Support/AuditDeadline.php`
- `backend/app/AI/Tools/WebsiteTool.php`
- `backend/app/Jobs/Audit/RunInitialAuditJob.php`
- `backend/app/Jobs/AI/AnalyzeWebsiteJob.php`
- `backend/app/AI/Pipelines/WebsiteAuditPipeline.php` (reduce to initial orchestration or replace with a named initial pipeline)
- `backend/app/Services/Audit/AuditDispatchService.php`
- `backend/app/Http/Controllers/Api/V1/AuditController.php`
- `backend/config/audit.php`
- tests

Verification:

- Stage A invokes neither `AuditProviderManager::collect` nor `AIService`;
- duplicate jobs no-op after one claim;
- core deadline stops optional work;
- all URL/redirect safety regressions still pass;
- the initial report remains a normal versioned report with real checks/intelligence.

## Milestone 4 — Durable provider fan-out, merge coordination, and AI enrichment

Refactor `AuditProviderManager` without duplicating provider logic:

- expose configured provider keys and one-provider collection;
- retain cache versioning, daily budgets, telemetry, Hunter provenance filtering, and normalized unavailable checks;
- keep the existing bulk `collect` method for public/legacy callers until callers are migrated.

Add `AuditEnrichmentService` and `AuditEnrichmentDispatchService` to:

- create one idempotent provider task per configured source when external enrichment is enabled;
- create Hunter only when Stage-A first-party evidence has no usable email;
- create AI as `waiting`, or `pending` when no provider phase exists;
- atomically claim/dispatch due tasks with bounded attempts/backoff;
- derive customer-safe aggregate state.

Create `RunAuditProviderEnrichmentJob` on `audit-enrichment`:

- claim one pending task under a row lock;
- call exactly one existing provider adapter;
- persist a JSON-safe normalized result and safe terminal reason;
- update heartbeat/telemetry;
- invoke the coordinator after terminal persistence.

Create `FinalizeAuditProviderEnrichmentJob` or an equivalent idempotent coordinator:

- wait until all provider tasks are terminal or the provider deadline is reached;
- terminalize overdue tasks as unavailable;
- merge all evidence under an audit lock;
- activate the waiting AI task;
- finish the audit as `complete` or `enrichment_partial` when AI is disabled.

Create `RunAuditAiEnrichmentJob` on `audit-enrichment`:

- read only persisted, tenant-owned evidence;
- run the unchanged `AIService`/provider fallback chain;
- pass the result through `AuditNarrativeValidator`;
- update only executive summary, recommendations, and outreach;
- increment report revision and mark enrichment terminal;
- retain the complete deterministic/provider report if AI is invalid or unavailable.

Bound audit AI separately without weakening other AI features: allow provider request timeout through the existing `AIProvider::generate` options, have `FallbackProvider` honor a total audit-enrichment deadline, and leave the provider order/configuration intact.

Files:

- `backend/app/Services/Audit/AuditProviderManager.php`
- `backend/app/Services/Audit/AuditEnrichmentService.php`
- `backend/app/Services/Audit/AuditEnrichmentDispatchService.php`
- `backend/app/Jobs/Audit/RunAuditProviderEnrichmentJob.php`
- `backend/app/Jobs/Audit/FinalizeAuditProviderEnrichmentJob.php`
- `backend/app/Jobs/Audit/RunAuditAiEnrichmentJob.php`
- `backend/app/AI/Providers/FallbackProvider.php`
- existing AI provider adapters only for per-call timeout option support
- `backend/app/Services/AI/AIService.php`
- `backend/app/Services/Audit/AuditNarrativeValidator.php` only if its public seam needs extension
- tests

Verification:

- providers can complete independently and concurrently;
- duplicate/late task deliveries are harmless;
- PageSpeed timeout does not block core report or other providers;
- AI provider fallback tests for Groq/Gemini/etc. continue to pass;
- AI receives observed checks/opportunities and cannot mutate score/coverage;
- provider/AI failure preserves the initial report and terminalizes enrichment.

## Milestone 5 — Bounded recovery, heartbeat, and health command

Fix `RecoverStaleAudits` so pending audits no longer loop forever:

- use configurable pending claim timeout, maximum dispatch attempts, and maximum pending age;
- preserve attempts across automatic redispatch;
- redispatch idempotently while below limits;
- fail terminally with `queue_unavailable` when limits are exhausted;
- continue marking stale core execution `worker_stalled` or `analysis_timed_out`.

Add enrichment recovery:

- dispatch due task rows every minute;
- retry stale running tasks below the ceiling;
- terminalize exhausted tasks as unavailable/failed;
- always invoke final coordination so usable reports do not remain “enriching” indefinitely.

Add `AuditQueueHeartbeatJob` on `audit-fast` every minute and `audits:queue-health --json`. The command reports fast-worker heartbeat age, queue depth/oldest age, pending and analyzing audit age, enrichment pending/running/stale counts, and detectable failed-job count with healthy/degraded/unhealthy exit codes.

Schedule initial dispatch, enrichment dispatch, heartbeat dispatch, and recovery every minute without overlap.

Files:

- `backend/app/Console/Commands/RecoverStaleAudits.php`
- `backend/app/Console/Commands/DispatchPendingAudits.php`
- `backend/app/Console/Commands/DispatchAuditEnrichments.php`
- `backend/app/Console/Commands/AuditQueueHealth.php`
- `backend/app/Jobs/Audit/AuditQueueHeartbeatJob.php`
- `backend/routes/console.php`
- `backend/config/audit.php`
- `backend/config/queue.php` only if the connection setting needs a named environment seam
- tests

Verification:

- a no-worker initial audit becomes meaningfully failed rather than remaining preparing;
- stale tasks finish enrichment as partial;
- health output and exit code match fresh/stale/missing heartbeat fixtures;
- scheduler registration is present and non-overlapping.

## Milestone 6 — Retry, compatibility, tenant isolation, and API regression

Update retry to clear/recreate lifecycle and task state atomically while preserving the existing audit public ID, history row, organization ownership, idempotency behavior, and quota semantics. Existing completed/partial/failed reports remain unchanged by migration.

Ensure:

- existing pending audits are picked up by new reconciliation;
- legacy `ai` deliveries cannot overwrite new results;
- enrichment rows are never loaded or mutated across tenants;
- provider result payloads contain no credentials/raw exception traces;
- initial failure still does not count against quota, while retry reserves quota;
- audit history, lead creation, public share, activity, notification, and report endpoints retain their contracts.

Files:

- `backend/app/Http/Controllers/Api/V1/AuditController.php`
- `backend/app/Services/Audit/AuditSubmissionService.php`
- `backend/app/Models/Audit.php`
- policies/resources/list repositories only where eager counts are required
- existing lifecycle/security/quota/admin tests plus new regressions

## Milestone 7 — Report-first frontend and automatic enrichment updates

Extend the authenticated audit contract in `audits.service.ts` with the lifecycle fields, revision, and safe aggregate enrichment object. Add pure progress helpers for legacy compatibility, visible-report truth, active enrichment, and stage copy.

Update `useAuditDetail` to poll every three seconds while primary analysis or enrichment is active and explicitly refetch on focus/reconnect. Keep cached report data during transient refetch failure.

Update the detail page while preserving its design:

- keep current skeleton, branded core-progress card, and mapped terminal failure card;
- map `starting`, `collecting_core`, and `finalizing_initial` truthfully;
- render a persisted initial report independently from enrichment state;
- show a compact accessible “Initial report ready — adding performance and AI insights” banner with aggregate progress;
- show a neutral optional-insights-unavailable state without hiding the report;
- show a low-emphasis “Update delayed” notice if a background refetch fails;
- keep lead, outreach, service CTA, share, fresh-audit, and Print/PDF actions available from the initial report;
- print a current-snapshot note while enrichment is active.

Expand safe error mapping for queue, worker, timeout, target, DNS, response-type, and size failures. Update retry mutation caching so the detail query immediately reflects the returned preparing state.

Files:

- `frontend/src/services/audits.service.ts`
- `frontend/src/lib/audit-progress.ts` and tests
- `frontend/src/lib/audit-failure-reasons.ts` and tests
- `frontend/src/hooks/use-audit-detail.ts`
- `frontend/src/hooks/use-collections.ts`
- `frontend/src/app/(dashboard)/audits/[id]/page.tsx`
- focused Playwright tests under `frontend/tests/e2e`

## Milestone 8 — Automated and real-worker verification

Add backend regression coverage for:

- schema/resource lifecycle;
- Stage-A provider/AI exclusion and 25-second budget behavior;
- initial atomic publication and task creation;
- provider fan-out, merge, cache, rate-limit, and unavailable states;
- evidence-grounded AI merge and invariant preservation;
- bounded initial/enrichment recovery;
- idempotency, quota, authorization, duplicate delivery, admin retry, history, lead, share, and export/report data.

Add a real database-queue smoke procedure/test harness that does not use `Queue::fake`:

1. configure a disposable database queue;
2. dispatch a heartbeat and an audit fast job;
3. run `php artisan queue:work --once --queue=audit-fast` (then enrichment workers);
4. assert persisted claim/heartbeat/status/report transitions;
5. retain timestamps/log evidence.

Add deterministic frontend/browser state-transition coverage, then run opt-in live E2E audits for:

- `https://www.axelsports.store/`;
- `https://ai.toolscase.com/`;
- unreachable host/timeout;
- private or unsafe redirect;
- provider timeout/429/5xx;
- AI unavailable;
- duplicate submission;
- stopped/restarted worker.

Verify initial report timing, enrichment revision without manual refresh, audit history, lead creation, public sharing, and Print/PDF. Live-target checks are not placed in default CI.

Fresh verification gates:

```text
backend: php artisan test
backend: vendor/bin/pint --test
frontend: pnpm test
frontend: pnpm type-check
frontend: pnpm lint
frontend: pnpm build
frontend: targeted pnpm exec playwright test ...
```

## Milestone 9 — Production operations documentation

Correct and extend `backend/DEPLOYMENT_RUNBOOK.md` and `.env.example` with:

- migration order and config-cache steps;
- `audit-fast` workers (recommended two processes, worker timeout 45 seconds);
- `audit-enrichment` workers (recommended three processes, worker timeout 130 seconds);
- temporary legacy `ai` worker until drained;
- unchanged `emails,default` worker coverage;
- `DB_QUEUE_RETRY_AFTER` greater than the longest worker timeout (240 seconds remains acceptable);
- scheduler requirement and exact commands;
- all audit deadlines/retry/heartbeat environment variables;
- explicit external-provider and AI opt-ins/credentials;
- `queue:restart`, Supervisor reload/status, health command, queue-depth queries, log paths, alert thresholds, failed-job handling, controlled reconciliation, rollback, and legacy queue removal criteria.

Also update `docs/CODEX_PROJECT_STATE.md` with the final architecture, tests, measurements, and any honest remaining operational dependencies.

No production command is run. Production rollout remains an explicit user-approved follow-up.

## Execution order and commit boundaries

Use small coherent commits after each verified boundary:

1. schema/model/API lifecycle;
2. shared report assembly and deadline-bound core job;
3. provider/AI enrichment and merge;
4. recovery/health/compatibility;
5. frontend report-first UX;
6. tests, runbook, and final project-state evidence.

At every boundary, inspect both repository diffs, preserve unrelated work, run the focused tests first, and only then run the full gates. Do not claim the 30-second target or worker execution without fresh measured evidence.
