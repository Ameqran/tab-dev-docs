# Encryption Scan and Implementation Plan

Date: 2026-04-27

## Scope

This scan covered:

- `tabular-review-backend`: API, workers, repositories, migrations, GCS wrapper, configuration, SSE, extraction, chunking, reviews, submissions, and tenant work.
- `tabular-review-app`: frontend API/SSE consumers and local state surface at a high level.
- `tab-dev-docs`: multitenancy and product/security notes.

The backend is the primary encryption control point because it owns persistence to Postgres, GCS object access, Redis queue/SSE state, and third-party AI calls.

## Current Security Baseline

- Authentication is handled by Clerk bearer tokens in backend middleware.
- User ownership and workspace access checks exist around workspaces, documents, reviews, submissions, and SSE.
- Document binary access is mediated by GCS V4 signed URLs with a 15-minute expiry.
- Secrets are supplied by environment-backed config keys for Clerk, GCS, Postgres, Redis, OpenAI, Gemini, and Vertex AI.
- Product docs already state encryption in transit and at rest as a requirement, but the codebase did not have application-layer encryption for database payloads before this work.

## Sensitive Data Inventory

| Area                        | Storage                                                                                     | Sensitive examples                                                          | Encryption fit                                                                                                                                                                                                            |
| --------------------------- | ------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Original uploaded documents | GCS objects referenced by `documents.storage_path` and `document_versions.storage_path`     | Insurance documents, policies, broker submissions, HR docs                  | Use cloud storage encryption by default, then add customer-managed keys or per-tenant KMS keys. App-layer file encryption is possible but affects preview, Document AI, and range reads.                                  |
| Document AI output          | GCS extraction output plus `extractions.output_path`                                        | Full OCR text and layout                                                    | Prefer GCS CMEK first. App-layer encryption before chunking is difficult because workers need plaintext for parsing.                                                                                                      |
| Document chunks             | `document_chunks.content`, `content_tsv`; `chunk_metadata.embedding` and extracted entities | Searchable OCR snippets and semantic embeddings                             | Do not blindly encrypt content while FTS/vector search depends on plaintext. Consider per-tenant isolated schemas plus DB/storage encryption; use redaction or separate protected fields for especially sensitive chunks. |
| Submission fields           | `submission_fields.ai_value`, `current_value`, source docs/chunks                           | Extracted policyholder, financial, underwriting, legal, and compliance data | Best first app-layer encryption target. Values are read/write payloads and not currently indexed by content.                                                                                                              |
| Field audit and conflicts   | `field_audit_log`, `field_conflicts`, conflict candidates                                   | Before/after values and conflicting extracted values                        | Encrypt after submission fields because these mirror the same sensitive values and need similar envelopes.                                                                                                                |
| Review cells                | `review_cell_values.value`, citations, enriched citations                                   | LLM answers over documents and evidence snippets                            | Good second target. Content is not central to retrieval, but API consumers expect plaintext after repository hydration.                                                                                                   |
| Redis queues and SSE        | BullMQ jobs, Redis streams/hashes                                                           | IDs, status payloads, event data, possibly extracted values                 | Keep payloads minimal. Avoid placing values in events; if unavoidable, encrypt event payloads or store references only.                                                                                                   |
| Config and credentials      | env/config                                                                                  | API keys, DB passwords, GCS credentials                                     | Keep in secret manager. Do not store secrets in repo config beyond empty defaults.                                                                                                                                        |
| Frontend/browser            | React state, local memory, network responses                                                | User-visible field values and signed URLs                                   | Transport encryption is HTTPS. Avoid localStorage for sensitive values, avoid logging, and minimize signed URL lifetime.                                                                                                  |

## Recommended Encryption Layers

1. Infrastructure encryption:
   Use managed encryption for Postgres disks, Redis, and GCS. For production, prefer customer-managed keys where available.

2. Application-layer encryption:
   Encrypt high-value JSON values before writing them to Postgres when they are not used for database filtering, FTS, or vector search.

3. Key isolation:
   Start with one environment key. Evolve to per-tenant data encryption keys once tenant schemas and membership are fully enforced.

4. Key rotation:
   Store a `keyId` in encrypted envelopes. Add migration jobs that decrypt with old keys and rewrite with a new key.

5. Search-aware design:
   Do not encrypt columns used by FTS/vector queries without replacing the retrieval design. For searchable content, use tenant isolation, cloud/DB encryption, redaction, or derived blind indexes only for exact-match cases.

## Priority Map

| Priority | Target                                                             | Reason                                                                    | Rollout risk                     |
| -------- | ------------------------------------------------------------------ | ------------------------------------------------------------------------- | -------------------------------- |
| P0       | `submission_fields.ai_value`, `submission_fields.current_value`    | High sensitivity, not content-indexed, narrow repository boundary         | Low                              |
| P1       | `field_audit_log.old_value/new_value` and `field_conflicts` values | Mirrors extracted field data                                              | Medium                           |
| P1       | `review_cell_values.value`, citations                              | LLM outputs can contain document excerpts                                 | Medium                           |
| P2       | GCS CMEK for uploads/extractions                                   | Strong storage boundary with minimal app changes                          | Medium, infrastructure-dependent |
| P2       | Redis event payload minimization/encryption                        | Prevents sensitive data lingering in streams                              | Medium                           |
| P3       | Document chunks                                                    | Highest sensitivity, but breaks FTS/vector retrieval if naively encrypted | High                             |

## Chosen Use Case Started

Implemented the P0 use case: optional application-layer encryption for `submission_fields.ai_value` and `submission_fields.current_value`.

Implementation shape:

- AES-256-GCM encryption service in `tabular-review-backend/src/infrastructure/security`.
- Encrypted JSON envelope stored directly in existing JSONB columns:
  - `__encrypted: true`
  - `version: 1`
  - `alg: aes-256-gcm`
  - `keyId`
  - `iv`
  - `tag`
  - `ciphertext`
- Repository-level encryption before insert/update.
- Repository-level decryption during domain hydration.
- Plaintext backward compatibility: existing unencrypted rows still read normally.
- Default-off config to avoid breaking local development and tests.

Environment controls:

- `FIELD_VALUE_ENCRYPTION_ENABLED=true`
- `FIELD_VALUE_ENCRYPTION_KEY=base64:<32-byte-base64-key>`
- `FIELD_VALUE_ENCRYPTION_KEY_ID=<key-id>`

Generate a local key with Node:

```bash
node -e "console.log('base64:' + require('crypto').randomBytes(32).toString('base64'))"
```

## Follow-Up Work

- Add a rotation keyring instead of a single configured key.
- Add a backfill job that rewrites existing plaintext submission field values as encrypted envelopes.
- Extend the same envelope service to field audit logs and conflicts.
- Add redaction rules to logs and SSE payload tests to prevent decrypted values from leaking outside API responses.
- Decide whether GCS CMEK should be global, per environment, or per tenant after tenant key ownership is finalized.
