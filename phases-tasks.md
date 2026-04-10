# Submission Intelligence - Phases and Tasks (Reconstructed)

This file reconstructs the lost `phases-tasks` document from the v0 plan and
updates it with what is actually merged in `tabular-review-backend` `main`.

Snapshot date: `2026-04-02`
Source baseline: `submission-implementation-plan-v0.md`

---

## Legend

- `done`: merged in `main`
- `partial`: started but not fully merged
- `not_started`: not merged in `main`
- `unmerged_branch`: implemented on branch(es) but not merged in `main`

---

## Phases

| # | Name | v0 Goal | Notes |
|---|---|---|---|
| 1 | Data Foundation | Create all submission/field/conflict/audit schema and seeds | merged in `main` |
| 2 | Field Definition Registry | Repository + cache + service for active field definitions | merged in `main` |
| 3 | Submission Core API | Submission CRUD + document attach + fields read/edit | merged in `main` |
| 4 | Extraction Worker | Event bridge, queue, trigger, Stage 1..4 pipeline | 4.1-4.4 found on branches, not in `main` |
| 5 | Conflict Resolution API | List/resolve conflicts with candidate management | not yet merged in `main` |
| 6 | Operational Tooling | Retry/recompute submission extraction jobs | not yet merged in `main` |

---

## Tasks

### Phase 1 - Data Foundation

| Task | Name | Description | Evidence in `main` |
|---|---|---|---|
| 1.1 | Migration: `field_definitions` (#317) | Create table. Add partial unique index on `(field_key) WHERE is_current = true` to enforce one active version per field key. | merged via PR #315 (`d15d39e`) |
| 1.2 | Seed: initial field registry data (#318) | Insert all registry fields as `version = 1`, `is_current = true`. Seed must be idempotent. | merged via PR #315 (`64f3ccc`) |
| 1.3 | Migration: `submissions` + `submission_documents` + `folders.folder_type` (#319) | Create both tables and add additive `folders.folder_type` (`general|submission`) with non-breaking behavior. | merged via PR #315 (`cc0c61c`) + follow-up folder type hardening (`aba8300`, `20260315103000_ensure_folders_folder_type.ts`) |
| 1.4 | Migration: `submission_fields` (#320) | Create table and add unique `(submission_id, field_key)` for idempotent worker upserts on retry. | merged via PR #315 (`3126f73`) |
| 1.5 | Migration: `submission_extraction_jobs` (#321) | Create table with partial unique index on `(submission_id) WHERE status = 'running'` to allow only one active job per submission. | merged via PR #315 (`303c854`) |
| 1.6 | Migration: `field_conflicts` + `field_conflict_candidates` (#322) | Create both tables together; `field_conflict_candidates.conflict_id` is required and `document_id` is nullable for synthetic manual-override candidates. | merged via PR #315 (`795e90f`) |
| 1.7 | Migration: `field_audit_log` (#323) | Create table with only FK constraints; append-only (no updates). | merged via PR #315 (`6ee548c`) |

### Phase 2 - Field Definition Registry

| Task | Name | Description | Evidence in `main` |
|---|---|---|---|
| 2.1 | `FieldDefinitionRepository` (#327) | Knex repository with `findAllCurrent()` and `findByKey(fieldKey)`, covered with DB tests. | merged via PR #331 (`48e726f`) |
| 2.2 | In-memory cache + invalidate endpoint (#328, #332) | Wrap repository with TTL cache (60s) and add authenticated admin endpoint `POST /api/admin/field-definitions/cache/invalidate` for hot reload. | merged via PR #331 (`bd5e3e2`) |
| 2.3 | `FieldDefinitionService` (#329) | Service layer exposing `getActiveFields()`, `getMandatoryFields({ })`, and `getFieldsByBlock()`; API/worker use service not repository directly. | merged via PR #331 (`ad5673c`) |

### Phase 3 - Submission Core API

| Task | Name | Description | Evidence in `main` |
|---|---|---|---|
| 3.1 | `SubmissionRepository` (#339) | Knex implementation in `infrastructure/submissions` with integration tests for mapping and not-found behavior. | merged via PR #372 (`0035f94`, `878b43a`) |
| 3.2 | `SubmissionService.createSubmission()` (#340) | Single transaction: insert submission -> create `folders` row with `folder_type='submission'` -> write `folder_id` back to submission. | merged via PR #372 (`4381907`, `1b80e61`) |
| 3.3 | Submission controller + routes (#342, #343, #344) | Add controller methods/routes/Zod schemas for create, fetch by id, and patch update on submissions. | merged via PR #373/#374 (`02bc497`, `fbf8dd9`, `48b94b9`) |
| 3.4 | `SubmissionDocumentRepository` + attach flow (#345, #346, #347) | Add submission-doc domain/contract/repo and attach use case validating submission/document existence, inserting `submission_documents`, and exposing attach endpoint (no ingest logic). | merged via PR #375 (`01261d6`, `cff3782`, `5098b73`) |
| 3.5 | Fields read + manual edit endpoints (#348, #349) | Add fields read grouped by block and manual edit endpoint for `currentValue` updates, including API docs and e2e coverage. | merged via PR #376/#377 (`5d967f0`, `ec41c38`, `b95ec6c`) |

### Phase 4 - Extraction Worker

| Task | Name | Description | Evidence |
|---|---|---|---|
| 4.1 | `SubmissionExtractionEventBridge` (#381) | Mirror `CellExecutionEventBridge`; emit `submission:extraction_progress`, `submission:extraction_completed`, `submission:extraction_failed` via Redis Pub/Sub and register at API boot. | branch `381-submission-phase4-01-submission-extraction-event-bridge` (`6216a21`) |
| 4.2 | BullMQ queue registration (#382) | Register `<env>-submission-extraction-queue` in worker with payload `{ submissionExtractionJobId }` and exponential backoff; queue wiring only (no processor). | branch `382-submission-phase4-02-submission-extraction-queue-registration` (`633d81f`) |
| 4.3 | Job service + trigger endpoint (#383) | Service creates `submission_extraction_jobs` row and enqueues job; wire `POST /api/submissions/:id/extractions` returning `202` + `{ jobId }`, with OpenAPI update. | branch `383-submission-phase4-03-submission-extraction-job-trigger` (`74ec792`) |
| 4.4 | Worker Stage 1 - RAG loop (#384) | For each active field: skip manual/calculated; default to aggressive extraction across all submission docs; when soft-cost mode is enabled, run source-ordered (`primary_source` then `fallback_secondary_source`) first and fallback to aggressive on missing/low/null confidence; if both return values, keep the higher-confidence candidate (tie keeps source-ordered); threshold is configurable and also drives `low_confidence` validation; persist `submission_fields` outputs (`ai_value`, `ai_confidence`, source chunk IDs), emit progress SSE per field, append `ai_write` in `field_audit_log`; no `QueryService` changes. | branch `384-submission-phase4-04-submission-stage1-rag-loop` (`e6ee62e`) |
| 4.5 | Worker Stage 2 - Conflict detection (#385) | After Stage 1, group by `field_key`; when distinct high-confidence candidates (`>=0.70`) exist, persist one conflict aggregate plus candidate rows and mark affected `submission_fields.validation_status = conflict`. | no merge commit in `main` |
| 4.6 | Worker Stage 3 - Adjudication (#386) | For mandatory fields in `missing/conflict/low_confidence`, assemble evidence and run adjudication LLM call, mark winning candidate, update `current_value`, set conflict resolution state, and append `conflict_resolution` audit entries via service/repository layer. | no merge commit in `main` |
| 4.7 | Worker Stage 4 - Calculated + manual stubs (#387) | Compute calculated fields from sibling `current_value` (never via LLM), initialize manual stubs, finalize extraction job metrics through services/repositories, and emit `submission:extraction_completed` SSE. | no merge commit in `main` |

### Phase 5 - Conflict Resolution API

| Task | Name | Description | Evidence |
|---|---|---|---|
| 5.1 | `GET /api/submissions/:id/conflicts` (#379) | Underwriter-facing endpoint to list pending conflicts and inspect all candidates per conflict. | no route/controller merged in `main` |
| 5.2 | `POST /api/submissions/:id/conflicts/:conflictId/resolve` (#379) | Underwriter-facing endpoint to resolve conflicts by selecting a candidate or submitting a free value. | no route/controller merged in `main` |

### Phase 6 - Operational Tooling

| Task | Name | Description | Evidence |
|---|---|---|---|
| 6.1 | `retry-failed-submission-extractions` (#380) | Recovery script mirroring existing extraction/chunking tooling to retry failed submission extractions after Phase 4 merge. | no submission-specific job script merged in `main` |
| 6.2 | `recompute-submission-fields` (#380) | Recovery script for recomputing submission fields, mirroring existing extraction/chunking operational patterns after Phase 4 merge. | no submission-specific recompute script merged in `main` |

---

## Post-v0 Submission Updates Already Merged

These landed after the original v0 and are now part of current submission behavior.

| Area | Update | Evidence in `main` |
|---|---|---|
| Submission list API | Added `GET /api/submissions` with pagination | PR #392 (`df28376`) |
| Submission documents list API | Added `GET /api/submissions/:id/documents` with pagination | PR #399 (`6437c00`) |
| Submission document update API | Added `PATCH /api/submissions/:id/documents/:documentId` | PR #400 (`4c4273b`) |
| Doc type behavior | `docType` optional in attach; defaults to `unclassified` | PR #424 (`974b814`, `4f69d1a`) |
| Attach policy | Removed status-based restriction for attaching docs | PR #425 (`8a54d35`) |
| Submission create contract | `name` is required; migration adds persisted name column | PR #427 (`073796c`) + migration `20260402100000_add_name_to_submissions.ts` |
| Submission docs response payload | include document details alongside submission-doc rows | PR #428 (`a3da331`) |
| OpenAPI coverage | docs for GET submission documents + name requirement examples | PR #429/#430 (`5abafbc`, `37ae5db`) |

---

## Open Post-v0 Submission API Follow-ups

These are newer submission-adjacent API contract tasks that sit on top of the
merged v0 foundation.

| Area | Update | Notes |
|---|---|---|
| Submission field citations contract | Consolidate Citation Schema for Submissions and Reviews (API Level) (#435) | API-only translation layer task: remove flat `aiSource*` response fields from submission field DTOs and expose unified `citations[]`, mapped from existing `submission_fields` source columns without changing DB schema or repositories. Also confirm submission SSE payloads do not leak a divergent citation contract. |

---

## Unmerged Phase 4 Branch Snapshot

Local/remote branches detected but not merged into `main`:

- `381-submission-phase4-01-submission-extraction-event-bridge`
- `382-submission-phase4-02-submission-extraction-queue-registration`
- `383-submission-phase4-03-submission-extraction-job-trigger`
- `384-submission-phase4-04-submission-stage1-rag-loop`
- `388-submission-phase-4-foundation-trigger-api`
- `feature/phase-4-demo`

Additional unmerged demo/foundation signals:

- `feature/phase-4-demo`: `9493dee` (phase 4 demo extraction worker 4.1-4.4)
- `388-submission-phase-4-foundation-trigger-api`: carries 4.1-4.3 equivalent commits (different SHAs due branch history)

---

## Dependency Graph (v0 structure, status-aware)

```text
Phase 1 - done
  -> Phase 2 - done
    -> Phase 3 - done
      -> Phase 4.1 - unmerged_branch
      -> Phase 4.2 - unmerged_branch
      -> Phase 4.3 - unmerged_branch
        -> Phase 4.4 - unmerged_branch
          -> Phase 4.5 - not_started
            -> Phase 4.6 - not_started
            -> Phase 5   - not_started
          -> Phase 4.7 - not_started

Phase 6 - not_started (planned after Phase 4 merges)
```

---

## Constraints (carried forward from v0)

- No destructive modification to existing tables.
- `QueryService` remains reusable by submission extraction worker.
- Manual edits must survive reruns/re-extractions.
- Worker writes should stay idempotent via uniqueness constraints.
- Only one extraction job runs per submission at a time.
- Conflict promotion threshold should avoid low-confidence noise.
