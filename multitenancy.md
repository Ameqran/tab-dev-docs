# Multitenancy Spike

## Purpose

This spike defines the recommended plan for adding multitenancy to Kaatch using a **schema-per-tenant PostgreSQL model**.

The current platform is authenticated and mostly user-scoped: a Clerk user maps to an internal user, and that user owns workspaces, documents, folders, reviews, submissions, SSE connections, and jobs. That is not enough for team/company tenants, tenant-level administration, collaboration, billing, data lifecycle controls, or stronger isolation around sensitive documents and LLM retrieval.

The target model is:

```text
public schema
  global identity, tenant metadata, memberships, global config

tenant_<id> schema
  workspaces, documents, reviews, submissions, chunks, jobs, audit trails
```

This gives stronger logical isolation than adding `tenant_id` to every row in one shared schema, while avoiding the operational weight of database-per-tenant.

## Current Architecture Summary

### Backend

The backend is a TypeScript Express API using:

- Clerk via `@clerk/express`.
- Express 5.
- Knex and PostgreSQL.
- pgvector for semantic retrieval.
- Google Cloud Storage for uploads and previews.
- Google Document AI for extraction.
- BullMQ and Redis for background jobs.
- Redis-backed Server-Sent Events.

Important files:

- `tabular-review-backend/src/api/app.ts`: Express setup, Clerk middleware, and `/api` auth enforcement.
- `tabular-review-backend/src/api/middlewares/clerk-auth.middleware.ts`: Clerk user resolution and internal user sync.
- `tabular-review-backend/src/types/express.d.ts`: current `req.user` shape.
- `tabular-review-backend/src/infrastructure/database/index.ts`: singleton Knex connection.
- `tabular-review-backend/src/application/workspaces/workspace.service.ts`: current workspace ownership checks.
- `tabular-review-backend/src/application/documents/document.service.ts`: current document ownership and GCS path generation.
- `tabular-review-backend/src/infrastructure/sse/sse-manager.ts`: current SSE authorization and Redis keying.
- `tabular-review-backend/src/infrastructure/queue/job-types.ts`: current worker payload types.

### Frontend

The frontend is a React/Vite application using:

- Clerk React.
- Axios interceptors to attach Clerk JWTs.
- React Query.
- Redux UI state for active workspace and active review.
- SSE client state for real-time updates.

Important files:

- `tabular-review-app/src/api/app.tsx`: authenticated API client.
- `tabular-review-app/src/providers/components/ClerkWrapper.tsx`: Clerk provider setup.
- `tabular-review-app/src/redux/slice/app.ts`: active workspace/review state.
- `tabular-review-app/src/services/sse/sseManager.ts`: workspace-aware SSE connections.
- `tabular-review-app/src/services/apiService.ts`: API service surface.

## Current Isolation Model

The current product model is:

```text
Clerk user -> internal user -> user-owned resources
```

Examples:

- `workspaces.user_id` is treated as owner.
- `documents.user_id` is treated as owner.
- `submissions.created_by` is treated as owner.
- Reviews are authorized by checking their workspace owner.
- GCS object paths use a user prefix: `${userId}/documents/...`.
- SSE keys are scoped by user ID and workspace ID.

This is user isolation, not tenant isolation.

## Recommended Isolation Strategy

### Chosen Strategy: Shared Database, Schema Per Tenant

Use one PostgreSQL database with:

- A stable `public` schema for global tables.
- One generated tenant schema per tenant.
- A controlled runtime wrapper that sets `search_path` inside a transaction.
- A tenant migration runner that applies tenant-schema migrations to all tenant schemas.

Example:

```text
public.tenants
public.users
public.tenant_memberships
public.tenant_schema_migrations
public.embedding_models
public.global_field_definitions

tenant_01hxyz.workspaces
tenant_01hxyz.documents
tenant_01hxyz.document_chunks
tenant_01hxyz.reviews
tenant_01hxyz.submissions
tenant_01hxyz.extraction_jobs
```

### Why This Is a Good Fit

Schema-per-tenant is a defensible choice for Kaatch because the product handles:

- Sensitive customer documents.
- Extracted underwriting data.
- Embeddings derived from private documents.
- LLM prompts and context windows that must never mix tenant data.
- Long-running asynchronous extraction/chunking jobs.

The schema boundary reduces the blast radius of missing filters. A query against `documents` inside `tenant_a` cannot accidentally read `tenant_b.documents` if the request is running with the correct `search_path`.

### Tradeoffs

Benefits:

- Stronger logical isolation than shared-schema rows.
- Cleaner tenant export, purge, and backup workflows.
- Simpler customer-data queries because most repositories do not need `tenant_id` filters.
- Easier future path to move a large tenant to dedicated infrastructure.
- Better mental model: `public` is platform data, tenant schemas are customer data.

Costs:

- Every request must resolve a tenant before touching customer data.
- Every tenant-scoped DB operation must run inside a transaction with a safe `search_path`.
- Knex migrations must be split into public migrations and tenant-schema migrations.
- Tenant provisioning must create a schema and run migrations.
- Workers must carry tenant context before loading customer rows.
- Cross-tenant analytics and admin reports need explicit iteration or warehouse-style copies.
- Connection pooling is risky if `search_path` is set globally instead of transaction-locally.

### Rejected Alternatives

Shared schema with `tenant_id`:

- Simpler migrations and queries across tenants.
- Weaker isolation because every query must remember tenant filters.
- Less aligned with sensitive document/embedding workloads.

Database per tenant:

- Strongest isolation.
- Operationally heavy for this stage.
- Requires connection routing, per-tenant migrations, per-tenant backups, and harder worker orchestration.

## Public Schema

The `public` schema should contain only platform-level data and metadata needed before a tenant schema is selected.

### `public.users`

Keep users global:

```sql
CREATE TABLE public.users (
  id uuid PRIMARY KEY,
  reference_id text NOT NULL UNIQUE,
  email text NOT NULL UNIQUE,
  name text,
  created_at timestamptz NOT NULL DEFAULT now(),
  updated_at timestamptz NOT NULL DEFAULT now()
);
```

`reference_id` remains the Clerk user ID.

### `public.tenants`

Add a first-class tenant registry:

```sql
CREATE TABLE public.tenants (
  id uuid PRIMARY KEY,
  name text NOT NULL,
  slug text NOT NULL UNIQUE,
  schema_name text NOT NULL UNIQUE,
  clerk_organization_id text UNIQUE,
  status text NOT NULL DEFAULT 'active'
    CHECK (status IN ('active', 'suspended', 'deleting', 'deleted')),
  plan text,
  region text,
  settings jsonb NOT NULL DEFAULT '{}',
  created_at timestamptz NOT NULL DEFAULT now(),
  updated_at timestamptz NOT NULL DEFAULT now(),
  CHECK (schema_name ~ '^tenant_[a-z0-9_]+$')
);
```

Rules:

- `schema_name` is generated by the backend, never accepted raw from the client.
- Use a strict format such as `tenant_<uuid_without_dashes>` or `tenant_<ulid_lowercase>`.
- Never let users rename schema names.
- Tenant display names/slugs can change; schema names should not.

### `public.tenant_memberships`

```sql
CREATE TABLE public.tenant_memberships (
  id uuid PRIMARY KEY,
  tenant_id uuid NOT NULL REFERENCES public.tenants(id) ON DELETE CASCADE,
  user_id uuid NOT NULL REFERENCES public.users(id) ON DELETE CASCADE,
  role text NOT NULL CHECK (role IN ('owner', 'admin', 'member', 'viewer')),
  status text NOT NULL DEFAULT 'active'
    CHECK (status IN ('active', 'invited', 'suspended')),
  created_at timestamptz NOT NULL DEFAULT now(),
  updated_at timestamptz NOT NULL DEFAULT now(),
  UNIQUE (tenant_id, user_id)
);
```

### Other Public Tables

Keep global:

- `embedding_models`, unless tenant-specific embedding providers become a product requirement.
- platform feature flags.
- tenant plan definitions.
- tenant schema migration tracking.
- global/default field definitions if tenants do not customize them.

Possible hybrid for field definitions:

```text
public.field_definitions
tenant schema field_definition_overrides
```

or:

```text
tenant schema field_definitions copied from public defaults at provisioning time
```

The second option is simpler if tenants need full customization.

## Tenant Schemas

Each tenant schema contains customer-owned data:

- `workspaces`
- `folders`
- `documents`
- `document_versions`
- `document_audit_trails`
- `document_chunks`
- `chunk_metadata`
- `reviews`
- `review_columns`
- `review_documents`
- `review_cell_values`
- `submissions`
- `submission_documents`
- `submission_fields`
- `field_audit_log`
- `field_conflicts`
- `field_conflict_candidates`
- `extraction_jobs`
- `extractions`
- `extraction_job_audit_trails`
- `extraction_audit_trails`
- `submission_extraction_jobs`
- `submission_extraction_job_audit_trails`

Inside a tenant schema, foreign keys can use normal unqualified table references if migrations run with `search_path` set to the tenant schema plus `public`.

Example:

```sql
SET search_path TO tenant_01hxyz, public;

CREATE TABLE documents (
  id uuid PRIMARY KEY,
  user_id uuid NOT NULL REFERENCES public.users(id),
  file_name text NOT NULL,
  storage_path text NOT NULL,
  ...
);
```

Keep user attribution columns:

- `user_id`
- `created_by`
- `uploaded_by`
- `added_by`

These should reference `public.users(id)` and represent attribution, not tenant ownership.

## Runtime Tenant Resolution

### Current State

`requireAuth` resolves only a user and attaches:

```ts
req.user = {
  id,
  referenceId,
  email,
  name,
};
```

### Target Request Context

Add an explicit auth context:

```ts
export interface AuthContext {
  user: {
    id: string;
    referenceId: string;
    email: string;
    name?: string;
  };
  tenant?: {
    id: string;
    schemaName: string;
    role: 'owner' | 'admin' | 'member' | 'viewer';
    clerkOrganizationId?: string;
  };
}
```

Tenant is optional only for tenant discovery and tenant creation routes.

### Tenant Selection

Supported options:

1. Clerk organization claim.
2. `X-Tenant-Id` request header.
3. Tenant route prefix such as `/api/tenants/:tenantId/...`.

Recommended path:

- Use `X-Tenant-Id` initially to reduce route churn.
- Map Clerk Organizations later if organization switcher/invites should live in Clerk.
- Always validate membership in `public.tenant_memberships`.
- Never trust tenant ID or schema name from the frontend.

### Middleware Order

```text
clerkMiddleware()
requireAuth()
resolveTenant()
apiRouter
```

`resolveTenant()` should:

- Skip tenant resolution for public/platform routes such as `GET /api/tenants`.
- Resolve tenant ID from header, route param, or Clerk org claim.
- Load `public.tenants` and active `public.tenant_memberships`.
- Reject missing tenant context on tenant-required routes.
- Attach `tenant.id`, `tenant.schemaName`, and `tenant.role`.

Error semantics:

- `401`: no authenticated user.
- `403`: authenticated user is not an active tenant member or lacks permission.
- `404`: resource not found inside selected tenant schema.
- `409`: tenant slug, membership, or provisioning conflict.

## Safe Schema Switching

This is the most important implementation detail.

Do not set `search_path` globally on a pooled connection. It can leak to a later request.

Use a transaction wrapper that sets `search_path` with `SET LOCAL`, which is scoped to the current transaction.

### Helper

Add a backend helper such as:

```ts
export async function withTenantTransaction<T>(
  ctx: AuthContext,
  callback: (trx: Knex.Transaction) => Promise<T>,
): Promise<T> {
  if (!ctx.tenant) {
    throw new Error('Tenant context is required');
  }

  assertValidSchemaName(ctx.tenant.schemaName);

  return getDb().transaction(async (trx) => {
    await trx.raw(
      `SET LOCAL search_path TO ${quoteIdentifier(ctx.tenant.schemaName)}, public`,
    );

    await trx.raw(`SELECT set_config('app.tenant_id', ?, true)`, [
      ctx.tenant.id,
    ]);
    await trx.raw(`SELECT set_config('app.user_id', ?, true)`, [ctx.user.id]);
    await trx.raw(`SELECT set_config('app.role', ?, true)`, [
      ctx.tenant.role,
    ]);

    return callback(trx);
  });
}
```

Notes:

- PostgreSQL identifiers cannot be passed like normal values in every context. Use a safe identifier quote helper or `pg-format` after validating schema names.
- Validate `schemaName` with `^tenant_[a-z0-9_]+$`.
- Never use user-provided strings as schema identifiers.
- All tenant data access should happen through this wrapper.

### Repository Rule

Repositories should not choose schemas themselves. Controllers/services should enter a tenant transaction, and repositories should receive `trx`.

Preferred:

```ts
await withTenantTransaction(ctx, async (trx) => {
  return documentRepository.findById(documentId, trx);
});
```

Avoid:

```ts
db.withSchema(schemaName)('documents')
```

Avoiding schema names in repository methods keeps tenant switching centralized and reduces injection risk.

## Authorization Model

### Roles

Start with:

- `owner`: full tenant control, tenant deletion, billing, membership management.
- `admin`: manage members and tenant data except billing/deletion.
- `member`: create and edit operational data.
- `viewer`: read-only access.

### Policy Layer

Add a policy service:

```ts
authorizationService.assertCan(ctx, 'document:create');
authorizationService.assertCan(ctx, 'workspace:update');
authorizationService.assertCan(ctx, 'tenant:manage_members');
```

Do not scatter role checks across controllers. Keep policy decisions centralized.

### Ownership Changes

Current checks compare resource user ID with current user ID.

Target behavior:

- Tenant membership grants access to tenant schema.
- Role policy decides whether the action is allowed.
- `user_id`, `created_by`, `uploaded_by`, and `added_by` become attribution.

Example:

- User A uploads a document.
- User B in the same tenant can view it if role allows.
- User C in another tenant cannot access it because their request runs against another tenant schema.

## Repository and Service Changes

### Controller Pattern

Current:

```ts
const document = await documentService.getDocumentById(id, req.user!.id);
```

Target:

```ts
const document = await withTenantTransaction(req.authContext!, (trx) =>
  documentService.getDocumentById(id, req.authContext!, trx),
);
```

The service checks policy and passes `trx` to repositories.

### Repository Signatures

Current:

```ts
findById(id)
findByUserId(userId)
save(entity)
```

Target:

```ts
findById(id, trx)
findAll(options, trx)
findByWorkspaceId(workspaceId, options, trx)
save(entity, trx)
delete(id, trx)
```

No tenant ID is needed in tenant-schema repositories because the schema is selected by `search_path`. Public repositories, such as tenant and membership repositories, keep using fully qualified `public` tables.

### Query Safety

Rules:

- Tenant tables should be referenced without schema qualification only inside `withTenantTransaction`.
- Public tables should be explicitly qualified as `public.users`, `public.tenants`, etc.
- Cross-schema queries should be rare and reviewed.
- Do not concatenate schema names outside the central transaction/migration/provisioning layer.

## Migration Architecture

The current Knex migrations assume one `public` schema. Schema-per-tenant requires splitting migrations into two tracks.

### Public Migrations

Public migrations create and update:

- `public.users`
- `public.tenants`
- `public.tenant_memberships`
- `public.tenant_invitations`
- `public.tenant_schema_migrations`
- global config tables

These run once per environment.

### Tenant Migrations

Tenant migrations create and update customer tables. They must be runnable against any tenant schema.

Example folder structure:

```text
tabular-review-backend/migrations/public
tabular-review-backend/migrations/tenant
```

or:

```text
tabular-review-backend/migrations
tabular-review-backend/tenant-migrations
```

### Tenant Migration Runner

Create a script/service that:

1. Loads all active tenants from `public.tenants`.
2. For each tenant:
   - Validates `schema_name`.
   - Acquires an advisory lock.
   - Creates schema if missing.
   - Sets `search_path` to tenant schema and `public`.
   - Runs pending tenant migrations.
   - Records migration state in `public.tenant_schema_migrations` or in each tenant schema's migration table.

Recommended migration tracking:

```sql
CREATE TABLE public.tenant_schema_migrations (
  tenant_id uuid NOT NULL REFERENCES public.tenants(id),
  migration_name text NOT NULL,
  applied_at timestamptz NOT NULL DEFAULT now(),
  PRIMARY KEY (tenant_id, migration_name)
);
```

### Tenant Provisioning

Tenant creation should be transactional where possible:

1. Insert `public.tenants` with generated `schema_name`.
2. Insert owner membership.
3. Create schema.
4. Run tenant migrations for that schema.
5. Optionally seed default tenant data.

If schema creation or migrations fail:

- Mark tenant `status = 'provisioning_failed'` or roll back where possible.
- Do not expose failed tenants as active.
- Log enough metadata to retry provisioning.

## Data Migration Plan

### Phase 1: Prepare Public Schema

- Add `public.tenants`.
- Add `public.tenant_memberships`.
- Add `schema_name`.
- Add tenant provisioning and tenant migration scripts.
- Keep existing app behavior unchanged.

### Phase 2: Create One Tenant Per Existing User

For current production data, generate one tenant per existing user:

```text
tenant.name = user.name or user.email
tenant.slug = generated unique slug
tenant.schema_name = generated tenant_<id>
membership.role = owner
```

### Phase 3: Provision Tenant Schemas

For each generated tenant:

- Create tenant schema.
- Run tenant migrations.
- Copy that user's existing customer data from current public tables into the tenant schema.

Copy groups:

- `workspaces` where `user_id = user.id`.
- `folders` where `user_id = user.id`.
- `documents` where `user_id = user.id`.
- Document versions/audit trails through matching documents.
- Reviews through matching workspaces.
- Review columns/documents/cells through matching reviews.
- Submissions where `created_by = user.id`.
- Submission documents/fields/conflicts/audit through matching submissions.
- Extraction and chunking rows through matching documents/jobs.

### Phase 4: Dual Read or Cutover

Preferred for simplicity:

- Do a maintenance-window cutover if data volume allows.
- Freeze writes.
- Copy data.
- Verify counts.
- Deploy tenant-schema code.
- Unfreeze writes.

Alternative:

- Dual-write to old public tables and tenant schemas temporarily.
- More complex and likely not worth it unless downtime is unacceptable.

### Phase 5: Retire Old Public Customer Tables

After verification:

- Stop application reads from old public customer tables.
- Keep old tables read-only for rollback period.
- Archive or drop after retention period.

## API Design

### Tenant Discovery

Add:

```text
GET /api/tenants
POST /api/tenants
GET /api/tenants/:tenantId
PATCH /api/tenants/:tenantId
GET /api/tenants/:tenantId/members
POST /api/tenants/:tenantId/invitations
PATCH /api/tenants/:tenantId/members/:userId
DELETE /api/tenants/:tenantId/members/:userId
```

These use `public` schema only.

### Tenant-Scoped APIs

Initial recommendation:

```text
X-Tenant-Id: <tenant uuid>
GET /api/workspaces
GET /api/documents/:id
GET /api/submissions
```

This minimizes route churn.

Long-term public API clarity may favor:

```text
GET /api/tenants/:tenantId/workspaces
GET /api/tenants/:tenantId/documents/:id
```

Regardless of API shape, backend must validate membership and derive `schema_name` from `public.tenants`.

## Frontend Changes

### Tenant State

Add `activeTenant` to Redux state next to `activeWorkspace` and `activeReview`.

When `activeTenant` changes:

- Clear active workspace.
- Clear active review.
- Clear pending documents.
- Disconnect/reconnect SSE.
- Invalidate tenant-scoped query keys or rely on query keys containing tenant ID.

### Tenant Switcher

Add a tenant switcher above the workspace selector. Workspaces should remain tenant-internal project/review groupings, not tenant selectors.

### API Client

Update `useApiClient()` to attach tenant context for tenant-scoped routes:

```ts
requestConfig.headers.Authorization = `Bearer ${token}`;
requestConfig.headers['X-Tenant-Id'] = selectedTenantId;
```

Tenant discovery routes must work before a tenant is selected.

### React Query Keys

Current keys are user-level:

```ts
['workspaces', 'list']
['reviews', 'workspace', workspaceId]
```

Target keys:

```ts
['tenants', 'list']
['workspaces', tenantId, 'list']
['reviews', tenantId, 'workspace', workspaceId]
['documents', tenantId, ...]
['submissions', tenantId, ...]
```

This prevents cached data leakage when switching tenants.

## Object Storage

### Current State

Objects are stored under:

```text
{userId}/documents/{objectName}
```

### Target State

Use tenant-prefixed paths:

```text
tenants/{tenantId}/documents/{documentId}/original/{objectName}
tenants/{tenantId}/documents/{documentId}/versions/{versionId}/{objectName}
tenants/{tenantId}/document-ai-output/{jobId}/
```

Rules:

- Generate paths server-side.
- Never trust raw storage paths from the frontend.
- Keep signed URL TTL short.
- Store object metadata: `tenant_id`, `document_id`, `uploaded_by`.
- Prefer explicit frontend origins in GCS CORS instead of wildcard CORS before production hardening.

## Background Jobs and Workers

Current job payloads only include resource IDs:

```ts
interface ExtractionJobPayload {
  extractionJobId: string;
}
```

Target payloads must include tenant context:

```ts
interface ExtractionJobPayload {
  tenantId: string;
  schemaName: string;
  extractionJobId: string;
  requestedByUserId?: string;
}

interface ChunkingJobPayload {
  tenantId: string;
  schemaName: string;
  extractionId: string;
  documentId: string;
}

interface SubmissionExtractionJobPayload {
  tenantId: string;
  schemaName: string;
  submissionExtractionJobId: string;
  requestedByUserId?: string;
}
```

Workers must:

- Validate `schemaName` against `public.tenants` by `tenantId`.
- Run tenant data access inside `withTenantTransaction`.
- Refuse jobs where tenant status is suspended/deleting/deleted.
- Include tenant ID in logs, metrics, and retry metadata.

Do not trust the queued `schemaName` blindly. It is useful for observability and avoiding an extra lookup in controlled cases, but the worker should still verify against `public.tenants`.

### Queue Fairness

Use shared queues initially, with tenant-aware concurrency and rate limits.

Future options:

- Dedicated queues for enterprise tenants.
- Per-tenant job groups.
- Tenant priority weights.

Idempotency keys should include tenant:

```text
tenant:{tenantId}:document:{documentId}:chunking
tenant:{tenantId}:submission:{submissionId}:extraction
```

## SSE and Redis

### Current State

SSE uses user/workspace connection indexes and workspace access checks.

### Target State

SSE connection metadata should include tenant:

```ts
{
  connectionId,
  tenantId,
  schemaName,
  workspaceId,
  userId,
}
```

Redis keys should be tenant-prefixed:

```text
sse:tenant:{tenantId}:workspace:{workspaceId}:connections
sse:tenant:{tenantId}:user:{userId}:connections
sse:tenant:{tenantId}:workspace:{workspaceId}:events
sse:tenant:{tenantId}:user:{userId}:events
```

Authorization:

- Verify active tenant membership.
- If `workspaceId` is supplied, check the workspace exists in the selected tenant schema.
- Cache membership checks with short TTL.
- Invalidate membership cache when roles/status change.

Event replay streams must be tenant-specific.

## LLM, Embeddings, and Retrieval

This is the highest-risk isolation area.

With schema-per-tenant:

- `document_chunks` and `chunk_metadata` live inside the tenant schema.
- Hybrid search runs against only the selected tenant schema.
- Prompt context can only come from that tenant schema if `search_path` is correct.

Still required:

- Validate document membership in the selected review/submission before retrieval.
- Keep all retrieval inside `withTenantTransaction`.
- Include tenant ID in LLM logs/metrics but do not log prompt content or extracted sensitive data.
- Do not run any cross-tenant semantic search path in the application.

`ChunkRepository.findHybrid()` should remain schema-agnostic but transaction-bound:

```ts
findHybrid(params, trx)
```

It should never use the global `db` connection without a tenant transaction for tenant data.

## Rate Limits and Quotas

Add tenant-level limits:

- API requests per tenant and per user.
- Concurrent uploads per tenant.
- Document processing jobs per tenant.
- LLM requests/tokens per tenant.
- Embedding requests/tokens per tenant.
- Storage quota per tenant.
- Max users, workspaces, submissions, documents.

Redis keys:

```text
rate:tenant:{tenantId}:api:{window}
rate:tenant:{tenantId}:llm:{window}
quota:tenant:{tenantId}:storage_bytes
quota:tenant:{tenantId}:documents
```

Plan limits should live in `public`.

## Audit Logging

Use both:

1. Tenant-local domain audit tables for resource history.
2. A central `public.audit_events` table for platform/security auditing.

Central audit event fields:

- `tenant_id`
- `actor_user_id`
- `actor_role`
- `action`
- `resource_type`
- `resource_id`
- `request_id`
- `ip_address`
- `user_agent`
- `metadata`
- `created_at`

Do not store document text, prompts, signed URLs, or tokens in audit metadata.

## Observability

Every log/metric for tenant work should include:

- `tenantId`
- `schemaName`
- `userId`
- `requestId`
- `workspaceId` when relevant
- `documentId` when relevant
- `jobId` when relevant

Metrics:

- API latency by tenant and route.
- Worker latency by tenant and job type.
- Queue depth by tenant.
- Document AI failures by tenant.
- LLM token usage by tenant.
- SSE connections by tenant.
- Storage usage by tenant.
- Tenant migration duration/failures.

## Security Controls

Required:

- Backend-only schema name generation.
- Strict schema name validation.
- Central `withTenantTransaction` using `SET LOCAL search_path`.
- No global `search_path` mutation on pooled connections.
- Tenant membership validation in middleware.
- Tenant context required for tenant-scoped APIs.
- Public repositories explicitly qualified to `public`.
- Tenant repositories only used inside tenant transactions.
- Tenant context in worker payloads and worker verification.
- Tenant-prefixed GCS paths.
- Tenant-prefixed Redis keys.
- Tenant-aware frontend query keys.
- Central authorization policy layer.
- Cross-tenant access tests for every API family.

Optional defense in depth:

- Store `tenant_id` in critical tenant tables even inside tenant schemas for audit/debugging.
- PostgreSQL RLS using `current_setting('app.tenant_id')` if redundant tenant IDs are added.
- Dedicated queues or databases for enterprise tenants.
- Tenant-specific encryption keys for enterprise tenants.

## Performance and Scaling

### Database

Each tenant schema has its own table/index set. This has useful locality but increases object count.

Watch:

- Number of schemas.
- Number of tables/indexes per schema.
- Migration time across all tenants.
- Autovacuum behavior across many small tables.
- pgvector index size per tenant.

For very many small tenants, shared-schema may become operationally simpler. For fewer tenants with sensitive/high-value data, schema-per-tenant is strong.

### pgvector

Each tenant schema can have its own vector index. Benefits:

- Smaller per-tenant indexes.
- Better isolation.
- Less chance of accidental cross-tenant retrieval.

Costs:

- More indexes to maintain.
- Tenant migrations involving vector indexes may be slower.

### Workers

Prevent one tenant from starving others:

- Per-tenant concurrency limits.
- Tenant-aware job dedupe.
- Tenant-aware backpressure.
- Dead-letter queues with tenant metadata.

## Testing Strategy

### Unit Tests

Add tests for:

- `resolveTenant`.
- schema name validation.
- `withTenantTransaction`.
- authorization policies.
- tenant provisioning service.
- tenant migration runner.

### Integration Tests

Create at least two tenant schemas per test run:

- `tenant_a` can create/read/update/delete its own data.
- `tenant_b` cannot read `tenant_a` data by ID.
- Switching tenant context changes visible data.
- Missing tenant context fails before repository access.
- Worker rejects mismatched tenant/schema payload.
- SSE does not replay or broadcast across tenants.

### E2E Tests

Scenarios:

- Two users in one tenant share a workspace/document according to roles.
- One user in two tenants switches tenants without cached data leakage.
- Viewer can read but cannot write.
- Admin can manage members but cannot owner-only actions.
- Tenant A document IDs cannot be used in Tenant B review/submission endpoints.

### Migration Tests

Seed existing public user-owned data, then run migration:

- One tenant is created per existing user.
- Tenant schemas are created.
- Data counts match expected owner-scoped records.
- Parent/child records are copied together.
- New application reads from tenant schemas only.

## Rollout Plan

### Milestone 1: Public Tenant Foundation

- Add `public.tenants`.
- Add `public.tenant_memberships`.
- Add `AuthContext`.
- Add tenant discovery endpoints.
- Add tenant switcher state in frontend.
- Keep existing user-scoped data paths working.

### Milestone 2: Tenant Schema Infrastructure

- Split public and tenant migrations.
- Build tenant migration runner.
- Build tenant provisioning service.
- Add `withTenantTransaction`.
- Add schema validation and identifier quoting helper.

### Milestone 3: First Tenant-Scoped Vertical Slice

Convert workspaces and reviews first:

- Create tenant schema versions of workspace/review tables.
- Route workspace/review APIs through tenant context.
- Update frontend to send `X-Tenant-Id`.
- Add cross-tenant tests.

This proves the pattern before moving sensitive document/chunk data.

### Milestone 4: Documents, Storage, and SSE

- Move documents/folders to tenant schemas.
- Change GCS paths to tenant prefixes.
- Add tenant to SSE metadata and Redis keys.
- Update document and folder APIs.
- Add cross-tenant tests for signed URL access and SSE.

### Milestone 5: Submissions and Extraction

- Move submissions and related tables.
- Move extraction jobs/extractions/chunks/chunk metadata.
- Add tenant context to job payloads.
- Update workers to use tenant transactions.
- Add tenant-aware quotas for expensive processing.

### Milestone 6: Migration and Cutover

- Generate one tenant per existing user.
- Provision schemas.
- Copy data.
- Verify counts and relationships.
- Deploy tenant-schema application.
- Retain old public customer tables read-only for rollback.

### Milestone 7: Collaboration and Administration

- Membership management UI.
- Invitations.
- Role-specific UI.
- Tenant settings.
- Billing/plan enforcement.
- Tenant export/delete workflows.

## Codebase-Specific Notes

### Workspaces

Current workspaces are user-owned. In the tenant model:

- `workspaces` moves into tenant schemas.
- `user_id` becomes `created_by` or owner attribution.
- `WorkspaceService.userHasAccess()` becomes tenant membership + workspace existence check.

### Documents

Current documents are user-owned and stored under user paths.

Target:

- `documents` moves into tenant schemas.
- `user_id` becomes uploader attribution.
- GCS paths use tenant and document IDs.
- `DocumentRepository.findById()` must use transaction-bound tenant schema.
- `document_chunks` and `chunk_metadata` live in the same tenant schema.

### Reviews

Reviews already belong to workspaces. Both live in tenant schemas.

Review columns, review documents, and cell values also move into tenant schemas.

### Submissions

Submissions are currently owned by `created_by`.

Target:

- `submissions` moves into tenant schemas.
- `created_by` becomes attribution.
- Submission documents, fields, conflicts, and audit logs live in tenant schemas.
- Decide whether submissions are tenant-wide or workspace-scoped. Frontend types currently suggest `workspaceId`, while inspected backend services/migrations do not make submissions workspace-owned.

### Field Definitions

Decide early:

- Global defaults in `public`.
- Tenant-local copies in each schema.
- Hybrid global defaults plus tenant overrides.

For insurance workflows, tenant-local customization is likely useful. A practical path is seeding tenant-local field definitions from public defaults when provisioning.

### Jobs

Job payloads must include `tenantId` and `schemaName`; workers must verify against `public.tenants`.

### Query and Chunking

All chunking/query services must run inside `withTenantTransaction`. This is non-negotiable because chunks and embeddings are the highest-risk data leakage path.

## Open Questions

- Should Clerk Organizations be the source of truth for tenant membership, or should the app own membership and use Clerk only for identity?
- Are submissions tenant-wide or workspace-scoped?
- Do tenants need custom field definitions, prompts, extraction settings, or model choices?
- How much downtime is acceptable for the initial migration?
- What are the expected tenant counts and largest-tenant data volumes?
- Are there future data residency or enterprise isolation requirements that would require database-per-tenant?
- What billing dimensions matter: users, documents, storage, pages processed, LLM tokens, submissions, or workspaces?

## Recommended First Implementation Slice

Build the smallest complete schema-per-tenant slice:

1. Add `public.tenants` and `public.tenant_memberships`.
2. Add tenant resolver middleware and `AuthContext`.
3. Add `withTenantTransaction`.
4. Add tenant provisioning and tenant migration runner.
5. Add frontend tenant selection and `X-Tenant-Id`.
6. Move workspaces and reviews into tenant schemas.
7. Add cross-tenant tests for workspace/review APIs.

Once that is stable, move documents, chunks, submissions, jobs, SSE, and storage.

This sequence is optimal because it proves the schema-switching and migration machinery on lower-risk product tables before touching the most sensitive paths: documents, embeddings, extraction jobs, and LLM retrieval context.
