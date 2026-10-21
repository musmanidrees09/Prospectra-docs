# Reliable Two-Stage Audit Pipeline Design

**Status:** Approved conversational design; implementation specification awaiting review  
**Date:** 2026-10-21  
**Scope:** Authenticated audit submission, execution, enrichment, recovery, progress UI, and production operations

## Purpose

Replace the single long-running audit job with a durable two-stage pipeline that produces a genuine deterministic report within 30 seconds under normal supported conditions, then enriches that report asynchronously with optional external-provider and AI evidence.

The initial report is a usable product result, not a placeholder. Enrichment may improve coverage, findings, and narrative, but it must never delay or invalidate the initial report.

## Confirmed Incident Boundary

The current request path is:

1. `AuditController::store` validates and delegates to `AuditSubmissionService`.
2. The service transaction creates an audit with `status=pending`, `processing_stage=preparing`, and one durable `audit_dispatches` row.
3. `AuditDispatchService` dispatches `AnalyzeWebsiteJob` to the `ai` database queue and marks the outbox row dispatched.
4. The first transaction inside `AnalyzeWebsiteJob::handle` changes the audit to `status=analyzing` and `processing_stage=starting` before any website or provider request begins.

Therefore, an audit that remains `pending/preparing` did not begin job execution. Target DNS, crawling, PageSpeed, and AI cannot be the cause of that state. The failure boundary is between queue insertion and worker consumption. The most likely production causes are no live worker, a worker not consuming `ai`, stale worker configuration, or an operational queue failure.

The current `audit_dispatches.dispatched_at` field proves only that Laravel accepted the dispatch call; it does not prove that a worker reserved or ran the job. The exact production instance cause must be confirmed from the production database, Supervisor/systemd state, and worker logs before deployment. No production access or mutation is part of this implementation without separate approval.

## Measured Baseline

Read-only local probes against the two supplied targets produced the following baseline:

| Target | Core analysis only | Core plus current providers |
| --- | ---: | ---: |
| `https://www.axelsports.store/` | 14.565 s | 48.276 s |
| `https://ai.toolscase.com/` | 7.008 s | Not required for diagnosis |

For the provider-enabled axelsports run, PageSpeed alone took 22.040 s and the remaining providers ran serially. This establishes two facts:

- the existing deterministic work can meet the initial-report target under normal supported conditions;
- optional providers cannot remain on the critical path if the product promises an initial result within 30 seconds.

## Goals

- Return the audit submission response promptly with a stable public identifier and idempotent quota reservation.
- Begin core analysis within five seconds at p95 when the production worker pool is healthy.
- Persist and expose a truthful initial report within 30 seconds at p95 for normally reachable supported sites.
- Show that report immediately, while enrichment continues independently.
- Update the visible report automatically without a full-page refresh.
- Bound every network call, job, retry sequence, and recovery loop.
- Convert unrecoverable initial-pipeline failures into safe, meaningful terminal errors.
- Preserve the initial report when any enrichment provider or AI service fails.
- Preserve deterministic scoring, separate Audit Coverage, security controls, quota rules, idempotency, and organization isolation.
- Recover or safely terminalize audits submitted before and during rollout.
- Keep audit history, lead generation, sharing, and print/PDF compatible.

## Non-Goals

- Enrichment is not required to finish within 30 seconds.
- AI does not determine pass/fail states or the Website Score.
- This work does not deploy to production, change DNS, or modify external hosting configuration.
- This work does not promise that all third-party providers will be available.
- This work does not turn queue internals or provider names into customer-facing implementation details.

## Architecture

### End-to-end flow

```text
POST /audits
  -> validate + tenant authorization + quota reservation
  -> create audit + initial dispatch outbox atomically
  -> return 202/pending
  -> audit-fast worker claims audit
  -> safe bounded core crawl + deterministic analysis
  -> persist initial report + findings + lead data atomically
  -> status partial/completed; report becomes visible
  -> create durable enrichment tasks atomically
  -> audit-enrichment workers fan out provider tasks
  -> deterministic merger replaces only supported pending checks
  -> optional AI narrative task runs from persisted evidence
  -> final enrichment state is completed or partial
```

The frontend continues polling the audit resource. Polling is already established in the project and is sufficient for automatic updates without adding a WebSocket dependency. Polling continues while either initial analysis or enrichment is active and stops only when the pipeline is terminal.

### Stage A: fast core audit

`RunInitialAuditJob` replaces the broad responsibility of `AnalyzeWebsiteJob` and runs only on `audit-fast`.

It performs:

- a `SafeWebsiteUrl`-validated homepage fetch;
- safe redirect validation and DNS pinning on every request;
- bounded same-origin crawl within the remaining time budget;
- robots and sitemap discovery when budget remains;
- deterministic HTML, headers, technical SEO, accessibility, security, content, and first-party contact analysis;
- deterministic score, coverage, findings, recommendations, opportunities, and initial lead data persistence.

It does not call PageSpeed, CrUX, Open PageRank, IPInfo, RDAP, Observatory, Hunter, Cloudflare Radar, or AI.

The job receives a monotonic deadline with a default core budget of 25 seconds. Fetch timeouts are capped by remaining budget. Optional crawl pages are skipped when the remaining budget is insufficient; corresponding checks stay `not_tested`, reducing coverage rather than becoming synthetic passes. The required homepage failure produces a classified initial-audit failure.

The normal operational target is:

- submission API: under two seconds;
- queue wait: under five seconds at p95;
- core execution and persistence: under 25 seconds at p95;
- accepted submission to visible initial report: under 30 seconds at p95.

These are measured service objectives, not permission to bypass safety checks.

### Stage B: provider enrichment

After the initial report transaction commits, durable provider tasks are dispatched to `audit-enrichment`. Each provider is its own idempotent job so independent services can execute concurrently when multiple workers are available and one slow service cannot block all others.

Expected provider task kinds are derived from enabled backend configuration:

- PageSpeed;
- Chrome UX Report;
- Open PageRank;
- IPInfo;
- RDAP;
- Mozilla Observatory;
- Hunter only when first-party contact discovery found no usable email and Hunter is enabled.

Cloudflare Radar remains absent unless configuration explicitly enables it. Google Custom Search remains unsupported and is never required.

Each task:

1. atomically claims its durable task row;
2. revalidates the target through existing backend-only provider boundaries;
3. uses bounded connect, request, payload, and retry limits;
4. records normalized evidence or an explicit unavailable reason;
5. writes provider telemetry;
6. invokes the coordinator after its terminal write.

An unavailable provider replaces no deterministic fact with a pass. It leaves its checks `not_tested` with a safe availability reason. Provider evidence can only change checks owned by that provider.

### Stage B: deterministic merge

The last terminal provider task, or scheduled reconciliation after the provider deadline, invokes an idempotent merger under an audit row lock.

The merger:

- loads the persisted initial report and normalized task results;
- replaces the matching pending provider-backed checks;
- recomputes score and coverage through the existing `AuditScoreCalculator` rules;
- regenerates category summaries, findings, recommendations, and opportunities through shared report-assembly services;
- updates persisted findings transactionally;
- never changes an evidence-backed core check based on AI output;
- records that provider enrichment is complete or partial.

Report assembly logic must be extracted and reused. Stage B must not duplicate scoring formulas or create a second source of truth.

### Stage B: optional AI narrative

Once provider merging is terminal, a separate `RunAuditAiEnrichmentJob` receives only persisted, tenant-authorized audit evidence. It may improve summaries, prioritization language, proposal context, or outreach copy. It may not invent facts, change deterministic check states, or directly set the Website Score or Audit Coverage.

AI fallback providers share one total deadline. A failed or unavailable AI task is terminal for that task but not for the audit report. The UI retains deterministic recommendations and identifies narrative enrichment as unavailable without exposing provider errors.

## Durable Data Model

### Audits

Add nullable timestamps:

- `initial_completed_at`: first usable report persistence;
- `enrichment_started_at`: first enrichment task claim;
- `enrichment_completed_at`: all enrichment work terminal.

Keep the existing primary status values:

- `pending`;
- `analyzing`;
- `partial`;
- `completed`;
- `failed`.

Do not add a `timed_out` primary status because `PlanLimitService` currently excludes only failed audits from quota. Initial timeout and worker exhaustion use `status=failed` with a classified error code. Enrichment failure never changes a usable audit to failed.

`completed_at` is set when the initial report becomes usable so history, lead workflows, and existing integrations continue to treat the audit as complete. `enrichment_completed_at` tracks the separate enhancement lifecycle.

### Enrichment tasks

Add `audit_enrichment_tasks` as the durable task/outbox/result table:

- `id`;
- `audit_id` foreign key with cascade delete;
- `kind` (provider key or `ai_narrative`);
- `phase` (`provider` or `ai`);
- `state` (`pending`, `running`, `succeeded`, `unavailable`, `failed`, or `skipped`);
- `result` nullable encrypted/JSON-safe normalized payload;
- `reason_code` nullable safe machine code;
- `attempts`, `dispatch_attempts`;
- `available_at`, `last_dispatched_at`;
- `started_at`, `heartbeat_at`, `finished_at`;
- timestamps;
- unique index on `(audit_id, kind)`;
- recovery indexes on `(state, available_at)` and `(state, heartbeat_at)`.

Task rows are created in the same transaction that publishes the initial report. The unique key makes duplicate initial-job delivery and scheduled recovery safe. Results contain normalized provider data, never secrets or raw exception traces.

The existing `audit_provider_runs` table remains append-only operational telemetry. It is not used as the task coordinator because it currently represents attempts rather than one durable result per audit/provider.

### API representation

`AuditResource` adds:

- the three lifecycle timestamps;
- an `enrichment` object with `state`, `completed`, `total`, and safe stage text;
- a stable `report_revision` derived from the audit update/version state.

The API does not expose internal queue names, provider credentials, raw failures, or cross-tenant task records.

## State Machine

| Primary status | Processing stage | Report visible | Meaning |
| --- | --- | --- | --- |
| `pending` | `preparing` | No | Accepted and durably awaiting a fast worker |
| `analyzing` | `starting` | No | Fast worker claimed the audit |
| `analyzing` | `collecting_core` | No | Bounded safe crawl is active |
| `analyzing` | `finalizing_initial` | No | Deterministic report transaction is active |
| `partial` or `completed` | `enriching_providers` | Yes | Initial report is ready; providers are active |
| `partial` or `completed` | `enriching_ai` | Yes | Provider merge is terminal; optional AI is active |
| `partial` or `completed` | `complete` | Yes | All enabled enrichment is terminal |
| `partial` or `completed` | `enrichment_partial` | Yes | Report is usable; at least one enrichment source was unavailable |
| `failed` | `failed` | No | Initial pipeline exhausted recovery or could not produce a truthful report |

Primary `partial` versus `completed` remains determined by tested evidence and coverage thresholds, not by whether AI ran.

All writes are state-guarded. Duplicate or late jobs return without overwriting a later terminal state. Row locks protect initial publication, provider merge, retries, and task coordination.

## Queue Topology

Use dedicated queues so optional work cannot starve initial audits:

- `audit-fast`: initial jobs and fast-worker heartbeat probes;
- `audit-enrichment`: provider and AI tasks;
- `ai`: legacy queue consumed temporarily during rollout;
- `emails,default`: existing non-audit work.

Recommended production worker groups:

| Group | Queue | Processes | Worker timeout | Purpose |
| --- | --- | ---: | ---: | --- |
| `prospetra-audit-fast` | `audit-fast` | 2 | 45 s | Initial reports and heartbeat |
| `prospetra-audit-enrichment` | `audit-enrichment` | 3 | 130 s | Provider fan-out and bounded AI |
| `prospetra-legacy-ai` | `ai` | 1 temporarily | 240 s | Drain/no-op legacy jobs during rollout |
| existing general worker | `emails,default` | Existing | Existing | Non-audit work |

The database queue `retry_after` must be greater than the largest active worker timeout with a safety margin. The existing 240-second setting is sufficient for the proposed 130-second enrichment timeout but must be verified in production configuration and cached config.

Internal HTTP retries are reduced and bounded so one job cannot exceed its worker timeout. Queue retries use explicit backoff and terminal failure handlers. Provider tasks may retry transient failures; deterministic invalid-target or unsupported-response errors do not retry.

## Dispatch and Recovery

### Initial dispatch

The existing audit submission/outbox transaction remains authoritative for quota and idempotency. `AuditDispatchService` dispatches `RunInitialAuditJob` to `audit-fast` and tracks dispatch attempts.

Scheduled reconciliation runs every minute:

- undispatched rows are dispatched with bounded exponential backoff;
- pending audits without a worker claim are safely redispatched because the job claim is idempotent;
- after the configured attempt/age ceiling, the audit becomes `failed` with `queue_unavailable`;
- failed initial audits remain excluded from quota according to current entitlement rules;
- user/admin retry creates a fresh reservation path through existing rules and dispatches the new pipeline.

Pending audits are never reset and redispatched forever.

### Initial worker recovery

The fast job updates `heartbeat_at` at meaningful phase boundaries. Scheduled recovery marks a stale analyzing audit failed with `analysis_timed_out` or `worker_stalled` after its bounded recovery ceiling. Its `failed()` handler performs the same guarded terminal transition if Laravel exhausts the job.

Customer messages are safe and actionable, for example: “We could not start this audit” or “The site analysis did not finish in time. Try again.” Raw exceptions remain in structured logs.

### Enrichment recovery

Enrichment dispatch is also outbox-like: task rows exist before dispatch, and the scheduler dispatches eligible undispatched tasks. Stale running tasks are retried up to their task ceiling. When exhausted, they become `unavailable` or `failed`, the coordinator proceeds, and the initial report remains visible.

The coordinator has an overall provider deadline. At the deadline, unfinished provider tasks become unavailable and the report is merged from all evidence actually obtained. AI has a separate bounded deadline.

### Worker health

Add a lightweight `AuditQueueHeartbeatJob` scheduled every minute on `audit-fast`. A successful reservation updates a cache-backed heartbeat with worker identity and time. Add an `audits:queue-health --json` command that reports:

- fast-worker heartbeat age;
- queued job count and oldest age per audit queue;
- pending audit count and oldest pending age;
- analyzing audit count and oldest heartbeat age;
- enrichment pending/running/stale counts;
- failed-job count scoped to audit queues where detectable;
- healthy/degraded/unhealthy exit status.

This supplements, rather than replaces, Supervisor/systemd monitoring. Production alerts should fire on stale heartbeat, oldest fast job over 30 seconds, pending audit over 60 seconds, stale analyzing heartbeat, or elevated initial failure rate.

## Error Classification

Initial terminal error codes include:

- `queue_unavailable`;
- `worker_stalled`;
- `analysis_timed_out`;
- `unsafe_target`;
- `dns_resolution_failed`;
- `site_unreachable`;
- `unsupported_response`;
- `response_too_large`;
- `analysis_failed`.

Enrichment uses task-local reason codes such as `provider_timeout`, `provider_rate_limited`, `provider_unavailable`, `provider_unsupported`, and `ai_unavailable`. These codes do not change the usable audit to failed.

HTTP responses and UI messages expose only mapped safe text. Structured logs include audit ID, organization ID, stage, job/task ID, attempt, duration, and exception class without credentials or sensitive response bodies.

## Security and Correctness Invariants

- Every fetched target and redirect continues through `SafeWebsiteUrl`.
- Only public HTTP(S) destinations on standard ports are allowed.
- DNS pinning, redirect limits, response-size limits, content-type limits, and request timeouts remain enforced.
- External credentials and calls remain backend-only.
- Audit, task, finding, lead, and retry access remains organization-scoped.
- `PlanLimitService` remains the single entitlement authority; admin remains unlimited.
- Submission idempotency remains scoped to organization, requester, and key.
- Failed initial audits do not consume quota; retry still reserves quota through the authoritative service.
- Missing or unavailable evidence never produces `passed` or a synthetic 100.
- Website Score and Audit Coverage remain distinct.
- AI cannot set check states, score, or coverage.
- Cloudflare Radar stays disabled unless explicitly enabled.
- Hunter remains fallback/on-demand after first-party discovery.
- Google Custom Search is not a required dependency.

## Frontend Experience

The visual system, layout, typography, and report components remain unchanged. The behavior changes are localized:

- `pending/preparing`: retain the current branded loading card, add truthful “Waiting for an audit worker” timing guidance only after a normal grace period, and avoid invented percentages;
- `analyzing`: show stage-specific core-analysis text and an accessible live status region;
- initial report ready: render the full report immediately even while enrichment is active;
- enrichment active: show a compact non-blocking banner such as “Initial report ready — adding performance and AI insights” with completed/total task progress;
- enrichment partial: keep the report and show a dismissible neutral note that some optional insights were unavailable;
- initial failure: replace the spinner with the mapped error, retry action, and link back to audit history;
- stale polling/network error: show “Update delayed” without changing the persisted audit state.

The query hook polls every three seconds while the primary status is pending/analyzing **or** enrichment is pending/running. It updates the report in place when `report_revision` changes, refetches on focus/reconnect, and stops at a terminal state.

Print/PDF is enabled as soon as the initial report is visible. If enrichment is active, the print view labels the output as the current report snapshot. Later exports include merged evidence automatically.

## Existing Audit Compatibility

- Existing completed, partial, and failed reports are not rewritten.
- Existing pending audits are eligible for the new fast job after deployment reconciliation.
- A legacy `AnalyzeWebsiteJob` delivery becomes a guarded handoff/no-op based on current status so it cannot overwrite a new result.
- Existing analyzing audits older than the configured heartbeat ceiling are terminalized safely and remain retryable.
- Existing `ai` jobs are drained by a temporary legacy worker; the queue is removed from Supervisor only after it is empty and no supported code dispatches to it.
- Audit history/API status values stay backward compatible.
- Lead generation continues from the initial deterministic result and may be enriched idempotently later.

## Implementation Boundaries

Expected backend changes include:

- split `AnalyzeWebsiteJob` into initial, provider-task, AI-task, and heartbeat jobs;
- add the enrichment task migration/model/coordinator;
- extract a reusable deterministic report assembler/merger;
- add core and provider deadline handling;
- update dispatch/recovery commands and scheduler registration;
- add queue-health command and structured audit metrics;
- extend `AuditResource` without exposing internal failures;
- update deployment and recovery runbooks.

Expected frontend changes include:

- extend audit types with enrichment lifecycle data;
- continue polling while enrichment is active;
- render the report independent of enrichment state;
- add accessible initial-progress, enrichment, delayed-update, and failure treatments;
- preserve existing report, lead, share, and print/PDF components.

No unrelated role, billing, SEO, blog, or visual redesign is included.

## Automated Verification

### Backend unit and feature tests

- submission transaction creates one audit and one initial outbox row;
- repeated idempotency key returns the same audit and quota reservation;
- real database-queue worker consumes `audit-fast` and transitions pending to analyzing;
- Stage A never invokes external providers or AI;
- deadline exhaustion stops optional crawling and leaves checks `not_tested`;
- initial persistence publishes score, coverage, report, findings, lead, and enrichment tasks atomically;
- provider tasks are unique and safe under duplicate delivery;
- independent provider success/unavailable/timeout results merge correctly;
- score and coverage are recomputed only from tested deterministic evidence;
- AI output cannot modify deterministic check states or score;
- AI/provider failure preserves the initial report;
- stale pending audits redispatch only to the configured ceiling, then fail safely;
- stale analyzing audits and exhausted jobs become classified failures;
- stale enrichment becomes unavailable and the coordinator still completes;
- tenant A cannot read, mutate, retry, or export tenant B’s audit/tasks;
- failed initial audits remain excluded from quota while retries reserve correctly;
- existing audits remain readable and legacy jobs cannot overwrite terminal results;
- API resources expose safe progress and never raw provider errors.

### Frontend tests

- preparing and core-analysis states render correct accessible copy;
- a partial/completed initial report renders while enrichment is running;
- polling continues through enrichment and stops at terminal completion;
- report revisions replace visible data without page refresh;
- optional enrichment failure leaves the report and export controls usable;
- initial failure shows mapped error and retry affordance;
- audit history and lead links remain functional;
- print layout includes the available report and snapshot notice when needed.

### End-to-end tests

Run local application services with a real database queue and managed workers, then verify:

1. submit `https://www.axelsports.store/`;
2. observe `pending -> analyzing -> partial/completed` and record each timestamp;
3. confirm initial report within 30 seconds under normal supported conditions;
4. confirm provider/AI progress updates without refresh;
5. confirm the final report revision, history row, lead, sharing, and PDF/print;
6. repeat for `https://ai.toolscase.com/`;
7. stop the fast worker and verify bounded `queue_unavailable` behavior;
8. simulate website timeout, redirect loop/private redirect, provider timeout/429/5xx, AI failure, duplicate submission, worker crash, and stale enrichment;
9. restart workers and verify safe recovery with no duplicate audit or quota charge.

Live-target tests are opt-in and excluded from normal CI because they depend on third-party availability. Default CI uses deterministic HTTP/provider fakes. Test evidence records environment, queue connection, worker commands, timings, terminal states, and relevant log excerpts.

## Production Configuration and Rollout

Implementation updates `docs/DEPLOYMENT_RUNBOOK.md` with exact Supervisor configuration. The rollout order is:

1. back up the production database through the existing hosting procedure;
2. verify queue tables, database queue connection, scheduler, and current failed jobs;
3. deploy code and run forward-only migrations;
4. clear/rebuild Laravel configuration cache;
5. install/start `audit-fast`, `audit-enrichment`, legacy `ai`, and general worker groups;
6. run `php artisan queue:restart` and verify every process is running;
7. run `php artisan audits:queue-health --json` and require a fresh fast heartbeat;
8. run reconciliation for existing pending/stale audits;
9. submit one controlled smoke audit and verify initial and enriched timestamps;
10. monitor queue age, error rate, and worker memory before draining/removing the legacy queue.

Supervisor processes use `autorestart=true`, bounded `stopwaitsecs`, explicit logs, a non-root application user, and environment/config matching PHP-FPM. The scheduler continues to run every minute. Log rotation and failed-job retention are documented; `queue:retry all` is never used blindly.

Rollback keeps migrations forward-compatible: old readers ignore new audit fields, and existing status values remain unchanged. If enrichment workers are stopped, initial reports still function. If fast workers are rolled back, the legacy worker remains available until rollback is complete.

## Observability and Acceptance Criteria

Record and alert on:

- submission-to-worker-start latency;
- core-analysis duration;
- submission-to-initial-report latency;
- provider and AI task duration/state by kind;
- queue depth and oldest job age;
- pending and stale-analyzing audit counts;
- initial completion/failure rate;
- enrichment completion/partial rate;
- report score and coverage revision changes;
- duplicate delivery and recovery counts.

The implementation is accepted only when fresh evidence shows:

- a real worker, not only `Queue::fake`, consumes the configured fast queue;
- both supplied sites produce genuine initial reports within the target under normal supported conditions;
- at least one enrichment revision appears without a browser refresh;
- initial failure and enrichment failure scenarios behave differently as designed;
- no unavailable check becomes passed and coverage remains truthful;
- idempotency, quota, tenant isolation, and safe-URL tests pass;
- audit history, lead generation, share, and print/PDF still work;
- backend tests and Pint pass;
- frontend tests, type-check, lint, and production build pass;
- deployment documentation contains the final worker, scheduler, health, recovery, and monitoring commands.

## Design Risks and Mitigations

- **Core crawl approaches the deadline:** stop optional page discovery using the shared monotonic deadline and preserve truthful coverage.
- **Provider fan-out increases outbound load:** retain caches, cap enrichment concurrency, apply provider-specific rate limits, and use unique tasks.
- **Concurrent results race:** persist task results separately and merge under an audit row lock.
- **AI changes report truth:** restrict AI to narrative fields and test invariant preservation.
- **Worker outage repeats the incident:** dedicated queues, heartbeat probe, bounded recovery, health command, Supervisor alerts, and meaningful terminal errors.
- **Rollout leaves legacy jobs behind:** temporarily consume `ai`, guard old jobs, reconcile existing states, and remove the legacy worker only after verification.
- **Live-target variability causes flaky CI:** keep live E2E opt-in and use deterministic fakes in the regression suite.

