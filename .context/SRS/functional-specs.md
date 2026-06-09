# Functional Specs — upex-bunkai-tms

> Generated: 2026-06-08 | Phase: Project Discovery Phase 2 — Architecture
> Sources: app/(auth)/*, app/(app)/*, app/api/v1/*, supabase/migrations/0001–0012, lib/api/*

---

## FR-001: Authentication — Magic Link (Browser)

**Module**: Auth
**Route**: `POST /api/v1/auth/magic-link`, `/auth/callback`
**Preconditions**:
- User provides a valid email address
- Supabase Auth is configured with a working email provider

**Business Rules**:
- BR-001-1: Email must be valid RFC 5321 format, max 254 chars.
- BR-001-2: `next` redirect parameter must be root-relative (starts with `/`, not `//`). Validated as open-redirect guard.
- BR-001-3: If Supabase returns HTTP 429, the endpoint must propagate `rate_limited` (429) — not mask it as a 200.
- BR-001-4: Magic-link issuance must write a best-effort audit row to `magic_link_tokens` + `magic_link_token_secrets` (SHA-256 of synthetic token + IP hash). Failure here must NOT fail the user-visible request.
- BR-001-5: Default post-auth redirect is `/projects` if no `next` param is provided.

**State Machine**:
```
anonymous → [POST magic-link] → email sent (ok: true)
email sent → [OTP click] → /auth/callback → session cookie set → redirect to next
```

**Validations**:
- Zod: `email` (string, email format, max 254), `next` (string, root-relative path, optional)

**Discovery Gaps**:
- `auth/callback` route file not found in explored directories — OTP exchange logic unverified.

Source: app/api/v1/auth/magic-link/route.ts, middleware.ts

---

## FR-002: Authentication — Headless Signup (CLI / Agent)

**Module**: Auth / API
**Route**: `POST /api/v1/auth/signup`
**Preconditions**: None (public endpoint)

**Business Rules**:
- BR-002-1: Email must be unique in Supabase Auth. If it already exists, return `conflict` (409) — never leak whether the email is registered separately.
- BR-002-2: User is created with `email_confirm: true` — no confirmation email sent. This is intentional for headless QA bot provisioning.
- BR-002-3: After user creation, the handler must immediately sign the user in and mint a PAT in the same response. The caller receives `{ user, session, pat }` in one round trip.
- BR-002-4: PAT is shown exactly once in the response. The `warning` field must be present: `"Store the PAT token now — it cannot be retrieved later."`.
- BR-002-5: `pat_scopes` defaults to all allowed scopes if not provided.
- BR-002-6: `pat_expires_in_days` max is 365. No expiry if omitted.

**Validations**:
- Zod: `email` (string, email, max 254), `password` (string, min 6, max 128), `pat_name` (optional, max 80), `pat_scopes` (optional enum array), `pat_expires_in_days` (optional int, 1–365)

Source: app/api/v1/auth/signup/route.ts, lib/api/pat.ts

---

## FR-003: Authentication — Headless Sign-In (CLI / Agent)

**Module**: Auth / API
**Route**: `POST /api/v1/auth/signin`
**Preconditions**: User account must exist in Supabase Auth

**Business Rules**:
- BR-003-1: On wrong email or wrong password, always return `unauthorized` (401). Never differentiate between "email not found" and "wrong password" — uniform 401 prevents email enumeration.
- BR-003-2: Response includes both the Supabase session (access_token, refresh_token) and a freshly minted PAT.
- BR-003-3: Same PAT options as FR-002 (scopes, expiry, name).

**Validations**: Same Zod schema as FR-002.

Source: app/api/v1/auth/signin/route.ts

---

## FR-004: Workspace Creation

**Module**: Workspace Management
**Route**: `POST /api/v1/workspaces`, `/onboarding` (UI)
**Preconditions**: User must be authenticated (cookie session)

**Business Rules**:
- BR-004-1: Slug must match `^[a-z0-9][a-z0-9-]{1,38}[a-z0-9]$` — lowercase letters, digits, hyphens; 3–40 chars; no leading/trailing hyphen.
- BR-004-2: Slug must not be a reserved system path. Reserved values: `admin`, `api`, `app`, `auth`, `docs`, `invites`, `login`, `logout`, `onboarding`, `projects`, `public`, `qa`, `settings`, `static`, `workspaces`, `_next`.
- BR-004-3: Slug must be globally unique in the `workspaces` table. Collision → `conflict` (409).
- BR-004-4: Workspace creation is atomic — uses `bunkai_bootstrap_workspace` RPC which inserts `workspaces` + `workspace_members` (role=`owner`, status=`active`) in a single transaction.
- BR-004-5: The creating user is automatically the `owner`. The `owner_user_id` field is set at creation and is immutable.
- BR-004-6: New workspaces are created with plan=`community` by default.
- BR-004-7: If the user already has an active workspace membership, the `/onboarding` page short-circuits to `/projects`.

**Validations**:
- Zod: `name` (string, trimmed, min 1, max 80), `slug` (3–40 chars, regex + reserved-slug refinements)

Source: app/api/v1/workspaces/route.ts, app/(app)/onboarding/page.tsx, migrations/0006_bootstrap_workspace.sql

---

## FR-005: Workspace Retrieval and Update

**Module**: Workspace Management
**Route**: `GET /api/v1/workspaces`, `GET /api/v1/workspaces/[id]`, `PATCH /api/v1/workspaces/[id]`
**Preconditions**: Authenticated (cookie or bearer)

**Business Rules**:
- BR-005-1: `GET /api/v1/workspaces` returns only workspaces where the caller is an active member. RLS enforces this for cookie callers; admin+join enforces it for bearer callers.
- BR-005-2: `GET /api/v1/workspaces/[id]` returns 404 if the workspace does not exist or the caller is not a member.
- BR-005-3: `PATCH /api/v1/workspaces/[id]` only updates `name`. No other fields are patchable through this endpoint.
- BR-005-4: Only the workspace `owner` can PATCH — RLS `workspaces_update_owner` policy enforces this. Non-owner receives `forbidden` (403).
- BR-005-5: At least one field must be provided in the PATCH body — empty patch returns `bad_request` (400).

**Validations**:
- PATCH Zod: `name` (optional, string, trimmed, min 1, max 80)

Source: app/api/v1/workspaces/route.ts, app/api/v1/workspaces/[id]/route.ts, migrations/0005_rls_helpers.sql

---

## FR-006: Personal Access Token (PAT) Management

**Module**: Access Tokens / Security
**Routes**: `POST /api/v1/tokens`, `GET /api/v1/tokens`, `DELETE /api/v1/tokens/[id]`
**Preconditions**: Authenticated (cookie for creation; cookie or bearer for list/revoke)

**Business Rules**:
- BR-006-1: PAT issuance requires a cookie session — a PAT cannot create another PAT (prevents privilege escalation via compromised token).
- BR-006-2: Token format: `bk_pat_<12-char-prefix>.<secret>`. The family prefix `bk_pat_` enables GitHub/GitGuardian secret-scanning detection.
- BR-006-3: Secret is 256-bit random (32 bytes, base64url-encoded). Only SHA-256 of the secret is stored in `access_token_secrets`. Raw secret is shown exactly once in the POST response.
- BR-006-4: Token lookup is O(1): `token_prefix` (first 12 chars) is indexed; constant-time SHA-256 comparison prevents timing attacks.
- BR-006-5: `scopes` array must have at least 1 element. Allowed values: `atc:read`, `atc:write`, `run:execute`, `workspace:admin`.
- BR-006-6: `workspace_id` is optional (null = global/cross-workspace token).
- BR-006-7: `expires_in_days` max is 365. No expiry if omitted (null = never expires).
- BR-006-8: Revocation is soft-delete: `DELETE` stamps `revoked_at`, does not physically delete the row.
- BR-006-9: `GET /api/v1/tokens` returns all caller's tokens (no hash, no prefix collision data). RLS `access_tokens` ownership policy enforces caller can only see own tokens.
- BR-006-10: `last_used_at` is updated by the bearer middleware on each successful authentication.

**State Machine** (PAT lifecycle):
```
created (revoked_at = null, expires_at = null|future)
  → active: used in requests → last_used_at updated
  → revoked: revoked_at stamped (soft-delete, immutable)
  → expired: expires_at < now() (app-layer check in bearer middleware)
```

Source: app/api/v1/tokens/route.ts, lib/api/pat.ts, lib/api/middleware/bearer.ts, migrations/0008_access_tokens.sql

---

## FR-007: Workspace Invite Flow

**Module**: Team Management
**Routes**: `POST /api/v1/workspaces/[id]/invites` (issue), `GET /api/v1/workspaces/[id]/invites` (list), `DELETE /api/v1/workspaces/[id]/invites/[inviteId]` (revoke), `POST /api/v1/invites/accept` (accept)
**Preconditions**: Issuer must be workspace admin or owner. Acceptor must be signed in.

**Business Rules**:
- BR-007-1: Only `admin` and `owner` roles can issue invites. RLS `workspace_members_insert_admin` enforces this — non-admins receive `forbidden` (403).
- BR-007-2: The `owner` role cannot be granted via invite. `workspace_invites.role` is constrained to `viewer` | `member` | `admin`.
- BR-007-3: Default invite role is `member`.
- BR-007-4: Invite token is raw random bytes (SHA-256 stored in `workspace_invite_secrets`). The raw token + `accept_url` are shown exactly once in the POST response.
- BR-007-5: Invites expire after 7 days by default (`expires_at = now() + 7 days`).
- BR-007-6: Invite acceptance validates: (a) token not revoked, (b) not already accepted, (c) not expired, (d) caller's email matches invite email (case-insensitive). Mismatch → `forbidden` (403).
- BR-007-7: Acceptance upserts `workspace_members` — if a row already exists (e.g., status=`invited`), it is promoted to `active`.
- BR-007-8: MVP does not send transactional email — the accept_url is logged to stdout. The caller must copy it from the POST response.
- BR-007-9: `GET /api/v1/workspaces/[id]/invites` returns derived `status` field: `pending` | `accepted` | `revoked` | `expired`.

**State Machine** (invite lifecycle):
```
created (expires_at = now+7d)
  → pending: default state for unexpired, unaccepted, unrevoked invites
  → accepted: accepted_at stamped, workspace_members upserted
  → revoked: revoked_at stamped (admin action)
  → expired: expires_at < now() (app-layer check, no DB column)
```

Source: app/api/v1/workspaces/[id]/invites/route.ts, app/api/v1/invites/accept/route.ts, migrations/0010_workspace_invites.sql

---

## FR-008: Acceptance Criteria Management (CRUD under a User Story)

**Module**: Story Authoring / AC Management
**Jira Story**: BK-15 — "TMS-AC | Manage criteria under a user story"
**Routes**: No standalone REST endpoint found in current `app/api/v1/` routes. CRUD is performed via Supabase PostgREST directly from server components.
**Preconditions**: User must be an active workspace member with at least `member` role. The parent User Story must exist.

**Business Rules**:
- BR-008-1: An Acceptance Criterion belongs to exactly one User Story (`user_story_id` FK, non-nullable).
- BR-008-2: ACs within a story are ordered by `position` (int). The combination `(user_story_id, position)` is unique.
- BR-008-3: An AC cannot be deleted if it has active bindings in `atc_acceptance_criteria` — ON DELETE CASCADE would propagate the deletion to the binding, which would violate the anchoring moat if the ATC has no other bound ACs.
- BR-008-4: The `title` field is required (non-nullable). The `description` field is optional.
- BR-008-5: The ATC editor (`/projects/[projectSlug]/atcs/[atcId]`) loads all ACs for all user stories in the project to populate the AC picker — ordered by `position` ascending.
- BR-008-6: The "anchoring moat" constraint: every ATC must be bound to at least 1 AC via `atc_acceptance_criteria`. The `bunkai_save_atc` RPC enforces this — if `ac_ids` array is empty, the save should fail (exact DB-level enforcement not confirmed in migration 0007; application-layer guard expected).
- BR-008-7: RLS policies on `acceptance_criteria` follow the standard workspace-scoped pattern:
  - SELECT: `bunkai_is_workspace_member` (via us → modules → projects chain)
  - INSERT/UPDATE/DELETE: `bunkai_can_write_workspace` (same chain)

**State Machine**: No explicit state machine — ACs are simple ordered text records. Lifecycle is create → optional update → delete (delete cascades to AC bindings, potentially orphaning ATCs if they have no other ACs).

**Validations** (inferred from schema):
- `title`: text, required, no length constraint in migration (application-layer validation expected)
- `description`: text, nullable
- `position`: int, required — must be unique within the parent story

**Discovery Gaps**:
- No standalone `POST /api/v1/stories/{id}/criteria` or `POST /api/v1/criteria` endpoint found. AC creation/update/deletion appears to go through Supabase PostgREST (direct table access) rather than a dedicated API route. A dedicated REST endpoint may be planned as part of BK-15.
- The exact minimum AC count enforcement for the anchoring moat is not confirmed in `bunkai_save_atc` migration (0007). The RPC signature accepts `ac_ids` but whether it validates array length is not visible from the migration file alone.
- No story editor UI page was found — user stories appear to be managed via PostgREST or a not-yet-built UI. The AC picker in the ATC editor provides read access, not AC management.

Source: migrations/0003_authoring.sql, migrations/0004_atcs.sql, migrations/0005_rls_helpers.sql, migrations/0007_save_atc.sql, app/(app)/projects/[projectSlug]/atcs/[atcId]/page.tsx

---

## FR-009: ATC Authoring and Versioning

**Module**: Test Case Authoring
**Routes**: `/projects/[projectSlug]/atcs/[atcId]` (UI), `bunkai_save_atc` RPC
**Preconditions**: Active session. User has `member` or above role. Parent project and user story must exist.

**Business Rules**:
- BR-009-1: An ATC must be linked to a User Story (`user_story_id` FK, ON DELETE RESTRICT — cannot delete a story with bound ATCs).
- BR-009-2: An ATC must be bound to at least one Acceptance Criterion. Binding stored in `atc_acceptance_criteria` M:N table.
- BR-009-3: `layer` must be one of: `UI` | `API` | `Unit`.
- BR-009-4: `status` starts as `unrun` on creation. Valid statuses: `pass` | `fail` | `blocked` | `skipped` | `running` | `unrun`. Any status can return to `unrun` (re-run).
- BR-009-5: Each save atomically increments `atcs.version` — acts as an optimistic-lock handle.
- BR-009-6: `bunkai_save_atc` is a full-replace RPC: it deletes and reinserts all steps, assertions, and AC bindings in the same transaction. There is no partial update path.
- BR-009-7: After each save, two triggers fire: `bunkai_set_updated_at()` updates `atcs.updated_at`, and `bunkai_atcs_refresh_tsv()` regenerates the GIN-indexed `tsv` column from title + tags for full-text search.
- BR-009-8: `atc_steps` are ordered by `position`. Each step has optional `input_data` and `expected` fields. `(atc_id, position)` is unique.
- BR-009-9: `atc_assertions` are ordered by `position`. `(atc_id, position)` is unique.
- BR-009-10: `slug` is unique per project. Generation strategy not confirmed in migrations.
- BR-009-11: `tags` is a `text[]` array — free-form labels included in `tsv` for full-text search.

**State Machine** (ATC status):
```
unrun (default)
  ├── [start run]  → running
  ├── [skip]       → skipped
  └── [block]      → blocked

running
  ├── [pass]   → pass
  └── [fail]   → fail

pass | fail | blocked | skipped
  └── [re-run] → unrun
```

Source: migrations/0004_atcs.sql, migrations/0007_save_atc.sql, app/(app)/projects/[projectSlug]/atcs/[atcId]/page.tsx

---

## FR-010: Project and Module Tree Navigation

**Module**: Project / Module Management
**Routes**: `/projects/[projectSlug]` (UI), `/projects` (index with auto-redirect)
**Preconditions**: Active session. User is an active workspace member.

**Business Rules**:
- BR-010-1: A project belongs to exactly one workspace. Projects are scoped by `workspace_id`.
- BR-010-2: `project.slug` is unique within a workspace — the combination `(workspace_id, slug)` has a UNIQUE constraint.
- BR-010-3: Modules form a self-referential tree with max depth 6 (path segments). The `modules.path` column is a materialized slash-separated path (e.g., `auth/login`).
- BR-010-4: `(project_id, path)` is unique — no two modules in the same project can share a path.
- BR-010-5: Module siblings are ordered by `position` (int).
- BR-010-6: Root modules have `parent_module_id = null`.
- BR-010-7: The `/projects/[projectSlug]` page fetches all modules, user stories, ATCs, and ACs for the project in parallel (Promise.all), then passes them to `buildModuleTree()` for client-side tree construction.
- BR-010-8: `/projects` (index) auto-redirects to the first project (by `created_at` ascending). If no projects exist, shows an empty-state placeholder. If no active workspace membership exists, redirects to `/onboarding`.
- BR-010-9: Project creation UI is not yet implemented ("ships in Phase E"). No `POST /api/v1/workspaces/{id}/projects` endpoint exists in current routes.

**Discovery Gaps**:
- Module CRUD endpoints not found in `app/api/v1/` — likely PostgREST only.
- Project creation endpoint not found.

Source: app/(app)/projects/page.tsx, app/(app)/projects/[projectSlug]/page.tsx, migrations/0002_projects_modules.sql

---

## FR-011: Identity Introspection (`/me`)

**Module**: Identity / API
**Route**: `GET /api/v1/me`
**Preconditions**: Authenticated (cookie or bearer)

**Business Rules**:
- BR-011-1: Response includes: `user.id`, `user.email`, `workspaces[]`, `active_workspace_id`, `auth.source`, `auth.scopes`.
- BR-011-2: For cookie callers, `active_workspace_id` is read from the `bk_active_ws` cookie. If the cookie points to a workspace the user can no longer see, falls back to the oldest visible workspace.
- BR-011-3: For bearer callers, `active_workspace_id` is the `workspace_id` bound to the token at issuance (may be null for global tokens — falls back to oldest visible workspace).
- BR-011-4: For bearer callers, email is fetched best-effort via admin auth API. Failure → email = null, request still succeeds.
- BR-011-5: Cookie callers have `auth.scopes = []` — the cookie session is unscoped; UI gates apply instead of scope checks.

Source: app/api/v1/me/route.ts, lib/api/auth.ts

---

## FR-012: Activity Log (Audit Trail)

**Module**: Audit / Observability
**Routes**: No read endpoint found in `app/api/v1/`. DB-level only via `qa_inspector_ro`.
**Preconditions**: Append-only. Written by `service_role` via logging middleware.

**Business Rules**:
- BR-012-1: The activity log is append-only — no UPDATE or DELETE policies exist on `activity_log`.
- BR-012-2: Each row records: workspace context, actor user, entity type + id, action string, jsonb payload, timestamp.
- BR-012-3: `entity_type` and `action` values have no CHECK constraint — valid values are not documented in the current migrations.
- BR-012-4: `qa_inspector_ro` and `qa_inspector_rw` roles can read activity log rows for their workspace.

**Discovery Gaps**:
- No API endpoint for reading the activity log was found.
- `entity_type` and `action` valid values not enumerated in schema.

Source: migrations/0009_cross_cutting.sql, migrations/0011_split_token_secrets.sql

---

## FR-013: Feature Flags

**Module**: Platform / Feature Management
**Routes**: No API endpoint found. DB-level only.

**Business Rules**:
- BR-013-1: Feature flags have `scope`: `global` (all users) or `workspace` (per-workspace override).
- BR-013-2: When `scope = 'workspace'`, the `workspace_id` field must be set.
- BR-013-3: Flags carry an optional `payload` (jsonb) for structured configuration.
- BR-013-4: No feature gate logic was found in current migrations or routes — the flag system is a Phase 2 mechanism. No flags are actively read in the current codebase.

Source: migrations/0009_cross_cutting.sql

---

## FR-014: User View State Persistence

**Module**: UI / UX
**Routes**: No API endpoint found. Likely client-side via Supabase PostgREST.

**Business Rules**:
- BR-014-1: View state is per `(user_id, project_id, view_kind)`. Each combination stores a serialized `state` (jsonb).
- BR-014-2: `view_kind` values have no CHECK constraint — valid values not documented.

Source: migrations/0009_cross_cutting.sql

---

## FR-015: Testability Guide (`/qa`)

**Module**: Platform / Developer Experience
**Route**: `/qa` (public, no auth required)

**Business Rules**:
- BR-015-1: The page is publicly accessible — no auth gate. Credentials are never inlined; they are stored in the Jira Epic `BK-29`.
- BR-015-2: The page renders `QaShell` with the `qaConfig` from `app/qa/qa-config.ts`.
- BR-015-3: Content includes: Playwright fixture snippets, MCP config blocks, PAT bootstrap flows, API base URL, OpenAPI spec reference, DBHub config.
- BR-015-4: The page content is generated with `language=es` flag (Spanish-language guide).

**Discovery Gaps**:
- The actual rendered sections of `QaShell` were not explored — the full content of `qa-config.ts` was not re-read in Phase 2 (covered in Phase 1).

Source: app/qa/page.tsx, app/qa/qa-config.ts (Phase 1)

---

## FR-016: Error Envelope and Request Tracing

**Module**: Cross-cutting / API Infrastructure
**Scope**: All `app/api/v1/*` routes that use `withApiHandler`

**Business Rules**:
- BR-016-1: Every API response (success and error) carries an `x-request-id` header.
- BR-016-2: Every error response uses the envelope: `{ error: { code, message, details?, request_id? } }`.
- BR-016-3: Error codes map to HTTP status via a fixed lookup table (see `lib/api/error-envelope.ts`): `bad_request`→400, `validation_failed`→422, `unauthorized`→401, `forbidden`→403, `not_found`→404, `conflict`→409, `rate_limited`→429, `internal_error`→500, `upstream_error`→502.
- BR-016-4: Zod validation failures automatically become `validation_failed` (422) with `details` containing the issues array.
- BR-016-5: Every request is logged (method, path, status, duration_ms, error_code if applicable) to stdout in structured JSON format (Vercel-indexable).

Source: lib/api/handler.ts, lib/api/error-envelope.ts
