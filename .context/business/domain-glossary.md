# Domain Glossary — upex-bunkai-tms

> Generated: 2026-06-08 | Phase: Project Discovery Phase 1 — Constitution
> Sources: supabase/migrations/0001–0012, lib/types.ts, lib/api/pat.ts, DESIGN.md

---

## Core Entities

### Workspace
The multi-tenant root — every piece of data belongs to exactly one workspace. Analogous to a GitHub organization.

| Field | Type | Notes |
|-------|------|-------|
| `id` | uuid PK | |
| `slug` | text UNIQUE | URL-safe identifier; 3–40 chars, `[a-z0-9-]`, no leading/trailing hyphen |
| `name` | text | Display name |
| `owner_user_id` | uuid FK → auth.users | Immutable after creation |
| `plan` | text | `community` \| `cloud` \| `enterprise` |
| `created_at` | timestamptz | |

Source: migrations/0001_tenancy.sql

---

### WorkspaceMember
RBAC join table. A user only accesses a workspace's data when an `active` membership row exists.

| Field | Type | Notes |
|-------|------|-------|
| `workspace_id` | uuid FK | |
| `user_id` | uuid FK → auth.users | |
| `role` | text | `viewer` \| `member` \| `admin` \| `owner` |
| `status` | text | `active` \| `invited` \| `suspended` |
| `joined_at` | timestamptz | |

PK: `(workspace_id, user_id)`

Source: migrations/0001_tenancy.sql

---

### Project
The "app under test". Scoped to a single workspace. One workspace can have many projects.

| Field | Type | Notes |
|-------|------|-------|
| `id` | uuid PK | |
| `workspace_id` | uuid FK | |
| `slug` | text | URL-safe, UNIQUE per workspace |
| `name` | text | |
| `description` | text nullable | |
| `created_at` | timestamptz | |

Unique constraint: `(workspace_id, slug)`

Source: migrations/0002_projects_modules.sql

---

### Module
A node in a self-referential hierarchical tree (max depth 6) under a Project. Organizes User Stories into a navigable file-tree structure.

| Field | Type | Notes |
|-------|------|-------|
| `id` | uuid PK | |
| `project_id` | uuid FK | |
| `parent_module_id` | uuid FK → modules(id) nullable | `null` = root node |
| `path` | text | Materialized slash-separated path (e.g. `auth/login`) |
| `name` | text | |
| `position` | int | Sibling ordering |
| `created_at` | timestamptz | |

Unique: `(project_id, path)`. Depth constraint: 1–6 path segments.

Source: migrations/0002_projects_modules.sql

---

### UserStory
The unit of business intent. Anchored to a Module. Optionally linked to an external tracker (e.g. Jira) via `external_id` / `external_url`.

| Field | Type | Notes |
|-------|------|-------|
| `id` | uuid PK | |
| `module_id` | uuid FK | |
| `title` | text | |
| `description` | text nullable | |
| `external_id` | text nullable | E.g. Jira issue key `BK-15` |
| `external_url` | text nullable | E.g. `https://…atlassian.net/browse/BK-15` |
| `created_at` | timestamptz | |

Source: migrations/0003_authoring.sql

---

### AcceptanceCriterion (AC)
A testable condition attached to a User Story. Ordered by `position`.

| Field | Type | Notes |
|-------|------|-------|
| `id` | uuid PK | |
| `user_story_id` | uuid FK | |
| `title` | text | |
| `description` | text nullable | |
| `position` | int | Ordering within story |
| `created_at` | timestamptz | |

Unique: `(user_story_id, position)`

Source: migrations/0003_authoring.sql

---

### ATC (Acceptance Test Case)
The core test entity. An ATC is a versioned, layered test case anchored to a Project + Module + UserStory and bound to ≥ 1 Acceptance Criterion. Full-text searchable via `tsv` (GIN index over title + tags).

| Field | Type | Notes |
|-------|------|-------|
| `id` | uuid PK | |
| `project_id` | uuid FK | |
| `module_id` | uuid FK | |
| `user_story_id` | uuid FK ON DELETE RESTRICT | Cannot delete story with bound ATCs |
| `slug` | text UNIQUE per project | URL-safe identifier |
| `title` | text | |
| `layer` | text | `UI` \| `API` \| `Unit` |
| `version` | int | Incremented on every save (optimistic-lock handle) |
| `status` | text | `pass` \| `fail` \| `blocked` \| `skipped` \| `running` \| `unrun` |
| `tags` | text[] | Free-form labels; fed into `tsv` |
| `tsv` | tsvector | Full-text search — populated by trigger |
| `created_at` | timestamptz | |
| `updated_at` | timestamptz | Updated by `bunkai_set_updated_at()` trigger |

Source: migrations/0004_atcs.sql

---

### AtcStep
An ordered execution step within an ATC. Steps have optional `input_data` and `expected` outcome fields.

| Field | Type | Notes |
|-------|------|-------|
| `id` | uuid PK | |
| `atc_id` | uuid FK | |
| `position` | int | Execution order |
| `content` | text | Step description |
| `input_data` | text nullable | Input values for this step |
| `expected` | text nullable | Expected outcome |

Unique: `(atc_id, position)`

Source: migrations/0004_atcs.sql

---

### AtcAssertion
A discrete assertion attached to an ATC. Complements steps with explicit pass/fail conditions.

| Field | Type | Notes |
|-------|------|-------|
| `id` | uuid PK | |
| `atc_id` | uuid FK | |
| `position` | int | |
| `content` | text | Assertion text |

Unique: `(atc_id, position)`

Source: migrations/0004_atcs.sql

---

### AtcAcceptanceCriteria (M:N junction)
Binding table enforcing the "anchoring moat" — every ATC must reference at least one AC. This is the structural guarantee that no ATC floats free of business requirements.

| Field | Type |
|-------|------|
| `atc_id` | uuid FK → atcs(id) ON DELETE CASCADE |
| `acceptance_criterion_id` | uuid FK → acceptance_criteria(id) ON DELETE CASCADE |

PK: `(atc_id, acceptance_criterion_id)`

Source: migrations/0004_atcs.sql

---

### AccessToken (PAT — Personal Access Token)
Issued via `POST /api/v1/tokens` for headless CLI / CI / AI-agent use. Format: `bk_pat_<prefix>.<secret>`. Secret hash is stored in the sibling `access_token_secrets` table (isolated from QA roles).

| Field | Type | Notes |
|-------|------|-------|
| `id` | uuid PK | |
| `user_id` | uuid FK | |
| `workspace_id` | uuid FK nullable | `null` = global/cross-workspace token |
| `name` | text nullable | Human label |
| `token_prefix` | text | First 12 chars of secret — indexed for O(1) lookup |
| `scopes` | text[] | Non-empty; subset of `{atc:read, atc:write, run:execute, workspace:admin}` |
| `expires_at` | timestamptz nullable | `null` = never expires |
| `revoked_at` | timestamptz nullable | Soft-delete; no DELETE policy |
| `last_used_at` | timestamptz nullable | |
| `created_at` | timestamptz | |

Source: migrations/0008_access_tokens.sql, lib/api/pat.ts

---

### AccessTokenSecret (sibling)
Secret material for PATs. Isolated from `qa_inspector_ro/rw` roles — `service_role` only.

| Field | Type |
|-------|------|
| `token_id` | uuid PK FK → access_tokens(id) |
| `hash` | text (SHA-256 of raw secret) |

Source: migrations/0011_split_token_secrets.sql

---

### WorkspaceInvite
Time-limited invitation issued by admin/owner. Token stored as hash in sibling table. Accepted via `POST /api/v1/invites/accept`.

| Field | Type | Notes |
|-------|------|-------|
| `id` | uuid PK | |
| `workspace_id` | uuid FK | |
| `email` | text | Invitee email |
| `role` | text | `viewer` \| `member` \| `admin` (not `owner`) |
| `invited_by_user_id` | uuid FK nullable | |
| `created_at` | timestamptz | |
| `expires_at` | timestamptz | Default: now + 7 days |
| `accepted_at` | timestamptz nullable | |
| `accepted_by_user_id` | uuid FK nullable | |
| `revoked_at` | timestamptz nullable | Soft-revoke |

Source: migrations/0010_workspace_invites.sql

---

### IdempotencyKey
POST replay protection. 24h TTL. Keyed on `(user_id, endpoint, key)`.

| Field | Type | Notes |
|-------|------|-------|
| `status` | text | `pending` \| `succeeded` \| `failed` |
| `response_snapshot` | jsonb nullable | Cached response for replays |

Source: migrations/0009_cross_cutting.sql

---

### ActivityLog
Append-only audit trail per workspace. Written by service_role via logging middleware. QA/analytics roles can read their workspace's log.

| Field | Type | Notes |
|-------|------|-------|
| `entity_type` | text | E.g. `atc`, `workspace`, `user_story` |
| `entity_id` | uuid nullable | |
| `action` | text | E.g. `created`, `updated`, `deleted` |
| `payload` | jsonb | |

Source: migrations/0009_cross_cutting.sql

---

### FeatureFlag
Phase-2 gradual rollout gate. Scope: `global` (all users) or `workspace` (per-workspace override).

| Field | Type | Notes |
|-------|------|-------|
| `key` | text | Flag identifier |
| `scope` | text | `global` \| `workspace` |
| `workspace_id` | uuid nullable | Required when scope=workspace |
| `enabled` | boolean | |
| `payload` | jsonb | Structured config payload |

Source: migrations/0009_cross_cutting.sql

---

### UserViewState
Per-user UI state persistence. Stores serialized view configuration per `(user_id, project_id, view_kind)`.

Source: migrations/0009_cross_cutting.sql

---

### MagicLinkToken (audit)
Bunkai-specific audit of magic-link issuances for replay detection. Actual auth is managed by Supabase Auth; this table provides rate-limiting signals. Secret fields in sibling `magic_link_token_secrets`.

Source: migrations/0009_cross_cutting.sql, migrations/0011_split_token_secrets.sql

---

## Relationships

```
Workspace (tenant root)
  |-- plan: community | cloud | enterprise
  |
  +--< WorkspaceMember (RBAC join)
  |     role: viewer | member | admin | owner
  |     status: active | invited | suspended
  |
  +--< Project (app under test)
        |
        +--< Module (tree, max depth 6, self-referential)
        |     |
        |     +--< UserStory
        |           |
        |           +--< AcceptanceCriterion
        |                 ^
        |                 | M:N (atc_acceptance_criteria)
        +--< ATC ----------+
              |
              +--< AtcStep (ordered, with input_data + expected)
              +--< AtcAssertion (ordered)
```

Cross-cutting (not in hierarchy):
```
Workspace
  +--< AccessToken (PAT — workspace-scoped or global)
        +--  AccessTokenSecret (sibling, service_role only)
  +--< WorkspaceInvite
        +--  WorkspaceInviteSecret (sibling, service_role only)
  +--< ActivityLog
  +--< FeatureFlag (scope=workspace)
  +--< IdempotencyKey

User
  +--< UserViewState (per user+project+view_kind)
  +--< MagicLinkToken → MagicLinkTokenSecret (sibling)
```

---

## State Machines

### ATC Status (`atcs.status`)
```
         +----------+
         |  unrun   |  (default on creation)
         +----------+
              |
     +--------+--------+
     |        |        |
  [start]  [skip]   [block]
     |        |        |
  running  skipped  blocked
     |
  [pass/fail]
     |        |
   pass     fail
```

States: `unrun` → `running` → `pass` | `fail` | `blocked` | `skipped`
Any state can return to `unrun` (re-run).

Source: migrations/0004_atcs.sql (CHECK constraint)

---

### WorkspaceMember Status (`workspace_members.status`)
```
invited --[accept]--> active --[suspend]--> suspended
                         ^                      |
                         +-----[reinstate]------+
```

Source: migrations/0001_tenancy.sql (CHECK constraint)

---

### IdempotencyKey Status (`idempotency_keys.status`)
```
pending --[response received]--> succeeded | failed
```

Source: migrations/0009_cross_cutting.sql (CHECK constraint)

---

### WorkspaceInvite lifecycle (fields, not CHECK)
```
created (expires_at = now+7d)
  |--[accept]--> accepted_at set, accepted_by_user_id set
  |--[revoke]--> revoked_at set
  |--[expire]--> expires_at < now (app-layer check)
```

Source: migrations/0010_workspace_invites.sql

---

## Enumerations (CHECK constraints)

| Table | Column | Allowed Values |
|-------|--------|---------------|
| `workspaces` | `plan` | `community`, `cloud`, `enterprise` |
| `workspace_members` | `role` | `viewer`, `member`, `admin`, `owner` |
| `workspace_members` | `status` | `active`, `invited`, `suspended` |
| `workspace_invites` | `role` | `viewer`, `member`, `admin` (owner NOT invitable) |
| `atcs` | `layer` | `UI`, `API`, `Unit` |
| `atcs` | `status` | `pass`, `fail`, `blocked`, `skipped`, `running`, `unrun` |
| `access_tokens` | `scopes` (each element) | `atc:read`, `atc:write`, `run:execute`, `workspace:admin` |
| `idempotency_keys` | `status` | `pending`, `succeeded`, `failed` |
| `feature_flags` | `scope` | `global`, `workspace` |

Source: migrations/0001–0009

---

## RLS Helper Functions

These `SECURITY DEFINER` functions bypass RLS internally for safe membership checks:

| Function | Returns | Description |
|----------|---------|-------------|
| `bunkai_is_workspace_member(ws_id)` | boolean | Caller has active membership |
| `bunkai_can_write_workspace(ws_id)` | boolean | Caller is active + role in (`member`,`admin`,`owner`) |
| `bunkai_is_workspace_admin(ws_id)` | boolean | Caller is active + role in (`admin`,`owner`) |
| `bunkai_is_workspace_owner(ws_id)` | boolean | Caller is active + role = `owner` |

Source: migrations/0005_rls_helpers.sql

---

## Database Functions (RPCs)

| Function | Auth | Description |
|----------|------|-------------|
| `bunkai_bootstrap_workspace(slug, name)` | `authenticated` | Atomic workspace + owner membership creation (resolves chicken-and-egg) |
| `bunkai_save_atc(atc_id, title, layer, tags, user_story_id, steps, assertions, ac_ids)` | `authenticated` (SECURITY INVOKER) | Atomic ATC full-replace: header + steps + assertions + AC bindings in one transaction |
| `bunkai_set_updated_at()` | trigger | Sets `updated_at = now()` before UPDATE on `atcs` |
| `bunkai_atcs_refresh_tsv()` | trigger | Refreshes `atcs.tsv` on INSERT/UPDATE of `title` or `tags` |

Source: migrations/0006_bootstrap_workspace.sql, 0007_save_atc.sql, 0004_atcs.sql

---

## UI Label vs Code Identifier

| UI Label | Code Identifier | Entity |
|----------|-----------------|--------|
| Workspace | `workspace` | `workspaces` table |
| Project | `project` | `projects` table |
| Module | `module` | `modules` table |
| User Story | `user_story` | `user_stories` table |
| Acceptance Criterion | `acceptance_criterion` | `acceptance_criteria` table |
| ATC / Test Case | `atc` | `atcs` table |
| Step | `atc_step` | `atc_steps` table |
| Assertion | `atc_assertion` | `atc_assertions` table |
| PAT / Access Token | `access_token` | `access_tokens` + `access_token_secrets` |
| Invite | `workspace_invite` | `workspace_invites` + `workspace_invite_secrets` |
| Layer: UI | `'UI'` | `atcs.layer` enum |
| Layer: API | `'API'` | `atcs.layer` enum |
| Layer: Unit | `'Unit'` | `atcs.layer` enum |
| Status: Pass | `'pass'` | `atcs.status` |
| Status: Fail | `'fail'` | `atcs.status` |
| Status: Blocked | `'blocked'` | `atcs.status` |
| Status: Skipped | `'skipped'` | `atcs.status` |
| Status: Running | `'running'` | `atcs.status` |
| Status: Not Run | `'unrun'` | `atcs.status` |
| Role: Viewer | `'viewer'` | `workspace_members.role` |
| Role: Member | `'member'` | `workspace_members.role` |
| Role: Admin | `'admin'` | `workspace_members.role` |
| Role: Owner | `'owner'` | `workspace_members.role` |

Source: lib/types.ts, DESIGN.md §3.5–§3.6, migrations/0001–0004

---

## Discovery Gaps

- [ ] `view_kind` enumeration values for `user_view_state` — the column is `text` with no CHECK; valid values not documented in migrations.
- [ ] `activity_log.entity_type` enumeration — no CHECK constraint; valid entity type strings not documented.
- [ ] `activity_log.action` enumeration — no CHECK constraint; valid action strings not documented.
- [ ] Feature flag `key` naming convention — no examples in code; patterns not yet established.
- [ ] ATC run/execution entity — `atcs.status` persists the last known result, but no `run`, `run_result`, or `test_execution` table exists in migrations 0001–0012 (planned for Sprint 2 per qa-config.ts).
- [ ] `qa_inspector_ro` / `qa_inspector_rw` role creation DDL — referenced in migrations/0011 but `CREATE ROLE` not in any migration file.
