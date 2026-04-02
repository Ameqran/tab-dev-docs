# Submission Intelligence — Implementation Plan

This document is the single reference for building the Submission Intelligence feature.
Tasks are atomic, isolated, and ordered so that each can be built, tested, and merged
independently without breaking existing Smart Library or Tabular Review functionality.

---

## Phases

| # | Name | Description |
|---|---|---|
| 1 | **Data Foundation** | All database migrations in dependency order. No application logic. Establishes every table, constraint, and index that the rest of the feature depends on. Nothing in this phase touches existing tables destructively. |
| 2 | **Field Definition Registry** | Repository, cache, and service layer on top of `field_definitions`. The hot-reload mechanism lives here. No API routes exposed yet — pure infrastructure consumed by later phases. |
| 3 | **Submission Core API** | CRUD for submissions and document attachment. No extraction logic. Gives the frontend enough to create a submission, upload documents, and read back field stubs. |
| 4 | **Extraction Worker** | The intelligence layer. Four sequential stages built as independent increments: RAG loop, conflict detection, long-context adjudication, and the final pass for calculated and manual stub rows. Reuses `QueryService` unchanged. |
| 5 | **Conflict Resolution API** | Underwriter-facing endpoints to list pending conflicts, inspect all candidates per conflict, and resolve by choosing a candidate or typing a free value. Depends on Phase 4 Stage 2 having written conflict rows. |
| 6 | **Operational Tooling** | Recovery job scripts mirroring the existing extraction/chunking scripts. Safe to build at any point after Phase 4 is merged. |

---

## Tasks

### Phase 1 — Data Foundation

| Task | Name | Notes |
|---|---|---|
| 1.1 | Migration: `field_definitions` | Create table. Add partial unique index on `(field_key) WHERE is_current = true` — enforces one active version per field key at the DB level. |
| 1.2 | Seed: field registry initial data | Insert all 26 fields from the registry config as `version = 1`, `is_current = true`. Verify seed is idempotent (safe to re-run). |
| 1.3 | Migration: `submissions` + `submission_documents` + `folders.folder_type` | Create both tables. Add additive `folder_type` column (`general\|submission`) to existing `folders` table — non-breaking. |
| 1.4 | Migration: `submission_fields` | Create table. Add unique constraint on `(submission_id, field_key)` — makes all worker upserts idempotent on retry. |
| 1.5 | Migration: `submission_extraction_jobs` | Create table. Add partial unique index on `(submission_id) WHERE status = 'running'` — enforces at most one active job per submission. |
| 1.6 | Migration: `field_conflicts` + `field_conflict_candidates` | Create both tables together. `field_conflict_candidates.conflict_id` is NOT NULL. `document_id` is nullable to support synthetic manual-override candidates. |
| 1.7 | Migration: `field_audit_log` | Create table. No constraints beyond FKs. Append-only — no updates ever. |

### Phase 2 — Field Definition Registry

| Task | Name | Notes |
|---|---|---|
| 2.1 | `FieldDefinitionRepository` | Knex repository. Methods: `findAllCurrent()`, `findByKey(fieldKey)`. Unit-test against test DB. |
| 2.2 | In-memory cache + hot-reload endpoint | Wrap repository with TTL cache (60s). Add `POST /api/admin/field-definitions/cache/invalidate` — authenticated, admin only. Editing a row in the DB then calling this endpoint is the full hot-reload flow — no restart needed. |
| 2.3 | `FieldDefinitionService` | Service layer over the repository. Exposes: `getActiveFields()`, `getMandatoryFields({ line_of_business, renewal_or_new })`, `getFieldsByBlock()`. Worker and API always call this — never the repository directly. |

### Phase 3 — Submission Core API

| Task | Name | Notes |
|---|---|---|
| 3.1 | `SubmissionRepository` | Knex repository. Methods: `create()`, `findById()`, `findByWorkspace()`, `updateStatus()`. |
| 3.2 | `SubmissionService.createSubmission()` | Single transaction: insert `submissions` row → create dedicated `folders` row with `folder_type = 'submission'` → write `folder_id` back onto the submission. Emit `submission:created` SSE on commit. |
| 3.3 | Submission controller + routes | Wire `POST /api/submissions`, `GET /api/submissions/:id`, `PATCH /api/submissions/:id`. Zod validation. Auth via existing `requireAuth`. No extraction endpoints yet. |
| 3.4 | `SubmissionDocumentRepository` + attach flow | Repository for `submission_documents`. Add `SubmissionService.attachDocument(submissionId, documentId, docType)`. Wire `POST /api/submissions/:id/documents`. Documents are already chunked and embedded before attachment — no new ingest logic needed. |
| 3.5 | Fields read + manual edit endpoints | `GET /api/submissions/:id/fields` — returns `submission_fields` grouped by block (empty at this stage — endpoint shape needed before worker writes). `PATCH /api/submissions/:id/fields/:fieldId` — writes `current_value`, flips `is_manually_edited = true`, appends `user_edit` row to `field_audit_log`. |

### Phase 4 — Extraction Worker

| Task | Name | Notes |
|---|---|---|
| 4.1 | `SubmissionExtractionEventBridge` | Mirror of existing `CellExecutionEventBridge`. Emits `submission:extraction_progress`, `submission:extraction_completed`, `submission:extraction_failed` via Redis Pub/Sub. Register in API server boot alongside existing bridges. |
| 4.2 | BullMQ queue registration | Register `<env>-submission-extraction-queue` in the worker server. Payload: `{ submissionExtractionJobId }`. Configure exponential backoff retries. No processor yet — queue registration only. |
| 4.3 | `SubmissionExtractionJobService` + trigger endpoint | Service creates `submission_extraction_jobs` row and enqueues the job. Wire `POST /api/submissions/:id/extractions`. Returns `202 Accepted` with `{ jobId }` immediately. |
| 4.4 | Worker Stage 1 — RAG extraction loop | For each active field definition from `FieldDefinitionService`: skip `manual` and `calculated` fields; identify preferred documents via `preferred_doc_types`; call existing `QueryService` with the field's extraction prompt; upsert result into `submission_fields` (`ai_value`, `ai_confidence`, source chunk IDs); emit `submission:extraction_progress` SSE after each field; write `ai_write` entry to `field_audit_log`. No changes to `QueryService`. |
| 4.5 | Worker Stage 2 — Conflict detection | After Stage 1: group all candidates by `field_key`. For any field where two or more candidates return distinct values and both exceed the confidence threshold (`>= 0.70`): insert one `field_conflicts` row (`resolution = pending`) and one `field_conflict_candidates` row per candidate document. Set `submission_fields.validation_status = conflict` on affected fields. |
| 4.6 | Worker Stage 3 — Long-context adjudication | For fields where `validation_status` is `missing`, `conflict`, or `low_confidence` and the field is mandatory: assemble evidence chunks and run adjudication LLM call. Set `was_chosen = true` on winning `field_conflict_candidates` row. Update `submission_fields.current_value`. Set `field_conflicts.resolution = resolved_by_ai`. Write `conflict_resolution` entry to `field_audit_log`. |
| 4.7 | Worker Stage 4 — Calculated fields + manual stubs | Compute calculated fields (e.g. `loss_ratio_5y`) from sibling `current_value` entries — never sent to LLM. Initialise stub `submission_fields` rows for all `manual` fields with `ai_value = null` and `validation_status = missing`. Finalise `submission_extraction_jobs` row with aggregate metrics (`fields_extracted`, `fields_not_found`, `conflicts_detected`, `avg_confidence`). Emit `submission:extraction_completed` SSE. |

### Phase 5 — Conflict Resolution API

| Task | Name | Notes |
|---|---|---|
| 5.1 | `GET /api/submissions/:id/conflicts` | Returns all `field_conflicts` with `resolution = pending`. Each conflict includes its full list of `field_conflict_candidates` with value, confidence, page number, and source document reference. |
| 5.2 | `POST /api/submissions/:id/conflicts/:conflictId/resolve` | Accepts `{ candidateId }` (underwriter picks an existing candidate) or `{ free_value }` (underwriter types a custom value). For `candidateId`: set `was_chosen = true` on that candidate, `false` on all others. For `free_value`: insert synthetic candidate with `document_id = null` and `was_chosen = true`. In both cases: set `field_conflicts.resolution = resolved_by_user`, update `submission_fields.current_value`, write `conflict_resolution` entry to `field_audit_log`. |

### Phase 6 — Operational Tooling

| Task | Name | Notes |
|---|---|---|
| 6.1 | `retry-failed-submission-extractions` job script | Finds `submission_extraction_jobs` rows with `status = failed`. Resets to `queued`. Re-enqueues into BullMQ. Same pattern as existing `reset-failed-extractions`. |
| 6.2 | `recompute-submission-fields` job script | For a given submission: clears all `submission_fields` rows where `is_manually_edited = false` (preserves underwriter edits). Bumps to latest `field_definition` versions. Re-enqueues extraction job. Used for prompt roll-forward after a field definition prompt is updated. |

---

## Dependency Graph

```
Phase 1 — Data Foundation
    └── Phase 2 — Field Definition Registry
            └── Phase 3 — Submission Core API
                    ├── Phase 4.1  SubmissionExtractionEventBridge
                    ├── Phase 4.2  Queue registration
                    └── Phase 4.3  Job service + trigger endpoint
                                └── Phase 4.4  Worker Stage 1: RAG loop
                                            └── Phase 4.5  Worker Stage 2: Conflict detection
                                                        ├── Phase 4.6  Worker Stage 3: Adjudication
                                                        └── Phase 5    Conflict Resolution API
                                            └── Phase 4.7  Worker Stage 4: Calculated + stubs

Phase 6 — Operational Tooling  (unblocked after Phase 4 merges)
```

---

## Constraints

- **No existing table is modified destructively.** Phase 1.3 adds a column to `folders` — additive only.
- **`QueryService` is not modified.** The worker calls it as-is. All submission-specific logic lives in the worker processor.
- **Manual edits survive re-extraction.** Task 6.2 explicitly skips rows where `is_manually_edited = true`.
- **Worker upserts are always idempotent.** The unique constraint on `(submission_id, field_key)` ensures a job retry updates existing rows rather than duplicating them.
- **Only one extraction job runs per submission at a time.** Enforced by the partial unique index on `submission_extraction_jobs (submission_id) WHERE status = 'running'`.
- **Conflict candidates below `0.70` confidence are not promoted to conflicts.** Prevents spurious conflicts from low-quality OCR extractions.