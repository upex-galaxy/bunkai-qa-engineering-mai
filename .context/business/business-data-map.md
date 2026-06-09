# Business Data Map — upex-bunkai-tms

> Generated: 2026-06-08 | Mode: CREATE | Sources: supabase/migrations/0001–0012, app/api/v1/**, lib/api/**, app/(app)/**, existing context files (business-model.md, domain-glossary.md, functional-specs.md, architecture.md, backend.md)

```
+-----------------------------------------------------------------------+
|                         BUNKAI TMS (分解)                             |
|   Multi-tenant Test Management System                                 |
|   Next.js 15 · Supabase Postgres 17 · Bun · TypeScript               |
|   "Developer-first ATC authoring with structural AC anchoring"        |
+-----------------------------------------------------------------------+
```

---

## 1. Executive Summary

Bunkai ("to decompose/analyze") is a developer-first, multi-tenant Test Management System designed for QA/SDET teams who manage large, structured test suites. The core problem it solves is test-case orphaning: in spreadsheets and generic tools, test cases accumulate without clear links to the business requirements they verify. Bunkai makes this structurally impossible — every Acceptance Test Case (ATC) must be bound to at least one Acceptance Criterion, and ACs are themselves anchored to User Stories in a module tree.

The system supports three first-class consumers: **QA Engineers** working in the browser with a Monaco-powered editor; **SDETs and AI Agents** operating headlessly via a REST API authenticated with Personal Access Tokens (PATs); and **Engineering Managers** administering workspaces and teams. Multi-tenancy is enforced at every layer — Postgres RLS ensures no workspace sees another's data; plan tiers (`community / cloud / enterprise`) gate feature access.

The product is **API-first by design**: an OpenAPI spec is generated and served at `/api/docs`, PATs carry fine-grained scopes, and the `/qa` testability guide page ships with Playwright fixtures and MCP config blocks so AI agents can onboard themselves.

```
+---------------+        +-------------------+        +------------------+
| QA Engineer   |        | SDET / AI Agent   |        | Eng Manager      |
| Browser/Cookie|        | Bearer PAT        |        | Browser/Cookie   |
+-------+-------+        +--------+----------+        +-------+----------+
        |                         |                           |
        v                         v                           v
+---------------------------------------------------------------+
|                      Bunkai TMS API                           |
|           Next.js 15 App Router  /api/v1/*                    |
|   withApiHandler → requireAuth → RLS-backed Supabase client   |
+--------------------------------+------------------------------+
                                 |
              +------------------+------------------+
              |                                     |
    +---------+---------+                 +---------+---------+
    |  Supabase Auth    |                 |  Supabase Postgres |
    |  Magic Link OTP   |                 |  17 + RLS + RPCs   |
    +-------------------+                 +-------------------+
```

---

## 2. Entity Map

### ASCII Relationship Diagram

```
auth.users (Supabase-managed)
  |
  +--< Workspace (tenant root, plan: community|cloud|enterprise)
        |
        +--< WorkspaceMember (RBAC: viewer|member|admin|owner / active|invited|suspended)
        |
        +--< WorkspaceInvite -------> WorkspaceInviteSecret (sibling, service_role only)
        |
        +--< AccessToken -----------> AccessTokenSecret (sibling, service_role only)
        |
        +--< ActivityLog (append-only audit trail)
        |
        +--< FeatureFlag (global or workspace-scoped)
        |
        +--< Project (the "app under test")
              |
              +--< Module (self-referential tree, max depth 6)
                    |
                    +--< UserStory (unit of business intent, optional Jira link)
                          |
                          +--< AcceptanceCriterion (ordered testable condition)
                                       ^
                                       | M:N (atc_acceptance_criteria)
                    +--< ATC ----------+ ---- must bind >= 1 AC (anchoring moat)
                          |
                          +--< AtcStep (ordered, input_data + expected)
                          +--< AtcAssertion (ordered, explicit pass/fail condition)

User (auth.users)
  +--< UserViewState (per user+project+view_kind, UI preference persistence)
  +--< MagicLinkToken -> MagicLinkTokenSecret (audit trail, service_role only)
  +--< IdempotencyKey (POST replay protection, 24h TTL)
```

### Entity Business Role Table

| Entity | Business Role | Why it exists |
|--------|--------------|---------------|
| `workspaces` | Tenant root | Every piece of data belongs to a workspace. Isolates teams from one another via RLS. The slug becomes the team URL. |
| `workspace_members` | RBAC join | Maps users to workspaces with a role (viewer/member/admin/owner) and status. A user sees NO workspace data without an active row here. |
| `workspace_invites` | Team onboarding | Admin/owner issues time-limited (7-day) email invites. Token shown once; hash stored in sibling table. MVP has no email — link returned in API response. |
| `workspace_invite_secrets` | Secret isolation | Stores invite token hash separately so QA/analytics DB roles cannot read it. service_role only. |
| `projects` | App under test | A workspace can test multiple products/repos. Projects scope all test artifacts (modules, ATCs). |
| `modules` | Navigation tree | Self-referential hierarchy (max depth 6) that organizes User Stories into a file-tree UI. Path is materialized (`auth/login`) for fast lookups. |
| `user_stories` | Business intent unit | Represents a feature slice or user need. Optional `external_id/external_url` fields link to Jira without requiring live sync. |
| `acceptance_criteria` | Testable conditions | Ordered, titled conditions under a story. The bridge between business requirements and test cases. |
| `atcs` | The core test entity | Acceptance Test Case — versioned, searchable, layer-typed (UI/API/Unit), bound to ≥1 AC. Status tracks last execution result. Full-text search via GIN-indexed `tsv`. |
| `atc_steps` | Execution procedure | Ordered steps with optional `input_data` and `expected` outcome. The "how to run this test". |
| `atc_assertions` | Pass/fail conditions | Discrete assertions the test must verify. Complements steps for explicit outcome documentation. |
| `atc_acceptance_criteria` | Anchoring moat | M:N junction ensuring every ATC proves at least one business requirement. Structural prevention of orphan tests. |
| `access_tokens` | Headless auth | Personal Access Tokens for CLI/CI/agent use. Scoped (`atc:read`, `atc:write`, `run:execute`, `workspace:admin`). Soft-deleted on revocation. |
| `access_token_secrets` | Secret isolation | PAT hash isolated from QA roles. service_role only. |
| `activity_log` | Audit trail | Append-only workspace audit log. Written by service_role via middleware. No client write policies. |
| `feature_flags` | Gradual rollout | Phase-2 feature gating. Global (all users) or workspace-scoped overrides. Currently no flags actively read in code. |
| `user_view_state` | UI persistence | Per-user, per-project view configuration. Owner-only read/write. Survives sessions. |
| `idempotency_keys` | POST replay protection | Prevents duplicate mutations on POST endpoints. Keyed on `(user_id, endpoint, key)`, 24h TTL. Available but not yet wired to any route. |
| `magic_link_tokens` | OTP audit | Best-effort audit of magic-link issuances for rate-limiting signals. Hash material in sibling table. |
| `magic_link_token_secrets` | Secret isolation | IP hash + token hash isolated from QA roles. service_role only. |

### Key Relationship Narratives

**The Anchoring Moat**: `atc_acceptance_criteria` is the structural heart of Bunkai. An ATC without an AC binding has no proof of WHY it exists. The `bunkai_save_atc` RPC atomically replaces all AC bindings in the same transaction — partial saves are impossible. Deleting a User Story with bound ATCs is blocked (`ON DELETE RESTRICT` on `atcs.user_story_id`), protecting the anchor chain.

**Tenant Isolation Chain**: Every table resolves its tenant by walking up: `atcs → projects → workspaces`, `user_stories → modules → projects → workspaces`, `acceptance_criteria → user_stories → modules → projects → workspaces`. RLS policies inline this join so the DB enforces isolation without application-layer filtering.

**Secret Sibling Pattern**: Three entities (`access_tokens`, `workspace_invites`, `magic_link_tokens`) have companion "secret" tables that hold only the hash material. This pattern was introduced in migration 0011 after realizing Postgres cannot hide a single column once a table-level SELECT grant exists. The DB-level REVOKE on sibling tables means QA roles with BYPASSRLS still cannot read secrets.

---

## 3. Business Flows

### Flow 1 — Workspace Creation + Member Invitation

```
+----------+     POST /api/v1/workspaces      +-------------+
|  User    |--------------------------------->|  API Route  |
| (cookie  |  { name, slug }                  |             |
|  session)|                                  |  1. Zod     |
+----------+                                  |     validate|
                                              |  2. Check   |
                                              |     reserved|
                                              |     slugs   |
                                              |  3. Call    |
                                              |     RPC     |
                                              +------+------+
                                                     |
                              bunkai_bootstrap_workspace(slug, name)
                              (SECURITY DEFINER — bypasses RLS)
                                                     |
                                            +--------v--------+
                                            | Postgres RPC    |
                                            |                 |
                                            | INSERT workspaces|
                                            | INSERT workspace |
                                            |   _members      |
                                            | (role=owner,    |
                                            |  status=active) |
                                            | RETURNS uuid    |
                                            +--------+--------+
                                                     |
                                        +-----------v-----------+
                                        | Response: 201         |
                                        | { workspace: {...} }  |
                                        +-----------------------+

--- INVITE FLOW (subsequent, admin/owner only) ---

+----------+  POST /api/v1/workspaces/{id}/invites  +-------------+
|  Admin   |---------------------------------------->|  API Route  |
+----------+  { email, role }                        |             |
                                                     |  1. Zod     |
                                                     |     validate|
                                                     |  2. Generate|
                                                     |     raw token|
                                                     |  3. Hash    |
                                                     |     token   |
                                                     +------+------+
                                                            |
                         +----------------------------------+
                         | RLS-gated INSERT (admin check)
                         |
                +--------v---------+    +---------------------------+
                | workspace_invites|    | workspace_invite_secrets  |
                | (metadata only)  |    | (token_hash via admin     |
                +------------------+    |  client)                  |
                                        +---------------------------+
                                                     |
                                        +-----------v-----------+
                                        | Response: 201         |
                                        | { invite, token,      |
                                        |   accept_url,         |
                                        |   warning }           |
                                        +-----------------------+

--- ACCEPTANCE FLOW ---

+----------+  POST /api/v1/invites/accept  +-------------+
| Invitee  |----------------------------->|  API Route  |
| (cookie) |  { token }                   |             |
+----------+                              |  1. Hash    |
                                          |     token   |
                                          |  2. Lookup  |
                                          |     secret  |
                                          |     sibling |
                                          |  3. Validate|
                                          |     expiry/ |
                                          |     revoked/|
                                          |     email   |
                                          +------+------+
                                                 |
                         +-----------------------+
                         |
                +--------v---------+
                | workspace_members|
                | UPSERT (active)  |
                +--------+---------+
                         |
                +--------v---------+
                | workspace_invites|
                | accepted_at stamp|
                +------------------+
```

**Business Rules**:
- BR-004-1/2/3: Slug must be `^[a-z0-9][a-z0-9-]{1,38}[a-z0-9]$`, not in reserved list, globally unique
- BR-004-4: Creation uses `bunkai_bootstrap_workspace` RPC — atomic, single transaction
- BR-004-5: Creating user is always `owner`, `owner_user_id` immutable
- BR-004-6: New workspaces default to `plan = 'community'`
- BR-007-1: Only `admin`/`owner` can issue invites (RLS enforced)
- BR-007-2: `owner` role cannot be granted via invite
- BR-007-6: Acceptance validates: not revoked, not accepted, not expired, email match (case-insensitive)
- BR-007-7: Acceptance upserts `workspace_members` — promotes `invited` → `active`

**Code paths**: `app/api/v1/workspaces/route.ts`, `app/api/v1/workspaces/[id]/invites/route.ts`, `app/api/v1/invites/accept/route.ts`, `supabase/migrations/0006_bootstrap_workspace.sql`, `supabase/migrations/0010_workspace_invites.sql`

---

### Flow 2 — User Story Creation with Acceptance Criteria

```
+----------+   PostgREST (direct table access)    +--------------+
|  Member  |------------------------------------->| Supabase     |
| (browser)|   INSERT INTO user_stories           | PostgREST    |
|          |   { module_id, title, description,   +---------+----+
|          |     external_id, external_url }               |
+----------+                                               |
                                         RLS check (SELECT 1 FROM modules
                                          JOIN projects JOIN workspace_members)
                                                           |
                                              +-----------v-----------+
                                              | user_stories row      |
                                              | created               |
                                              +-----------+-----------+
                                                          |
                        +------------------------------+--+
                        | INSERT INTO acceptance_criteria  |
                        | { user_story_id, title,          |
                        |   description, position }        |
                        +----------------------------------+
                                                          |
                                              +-----------v-----------+
                                              | AC rows created       |
                                              | (position unique per  |
                                              |  story)               |
                                              +-----------------------+
```

**Business Rules**:
- BR-008-1: AC belongs to exactly one User Story (non-nullable FK)
- BR-008-2: ACs ordered by `position`; `(user_story_id, position)` is unique
- BR-008-5: AC editor in ATC view loads all ACs for all stories in the project
- BR-009-1: ATCs reference User Story via `user_story_id` FK with `ON DELETE RESTRICT` — deleting a story with ATCs is blocked

**Discovery note**: No dedicated `POST /api/v1/stories` or `POST /api/v1/criteria` REST endpoint was found. Creation goes through Supabase PostgREST directly.

**Code paths**: `supabase/migrations/0003_authoring.sql`, `app/(app)/projects/[projectSlug]/atcs/[atcId]/page.tsx`

---

### Flow 3 — ATC Creation + Versioning + AC Linking

```
+----------+   ATC Editor (Monaco)    +-------------------+
|  Member  |------------------------->| Next.js Server    |
| (browser)|   Save action            | Component / SA    |
+----------+                          +--------+----------+
                                               |
                             bunkai_save_atc RPC (single transaction)
                             (SECURITY INVOKER — RLS still applies)
                                               |
                        +----------------------+----------------------+
                        |                                             |
               +--------v--------+                        +----------v--------+
               |  UPDATE atcs    |                        |  DELETE + INSERT  |
               |  title, layer,  |    Triggers fire:      |  atc_steps        |
               |  tags,          |  bunkai_set_updated_at |  atc_assertions   |
               |  user_story_id, |  bunkai_atcs_refresh_  |  atc_acceptance_  |
               |  version + 1    |  tsv (GIN index)       |  criteria         |
               +-----------------+                        +-------------------+
                        |
               +--------v--------+
               | Response: void  |
               | (client reads   |
               |  updated row)   |
               +-----------------+
```

**Business Rules**:
- BR-009-2: Must bind to ≥1 AC. `bunkai_save_atc` accepts `p_ac_ids uuid[]` — empty array allowed by DB (application-layer enforcement expected)
- BR-009-5: Every save increments `atcs.version` — optimistic-lock handle
- BR-009-6: Full-replace RPC — no partial update path. Steps/assertions/bindings fully replaced each save
- BR-009-7: Two triggers fire post-save: `bunkai_set_updated_at()` + `bunkai_atcs_refresh_tsv()` (GIN index refresh)
- BR-009-8/9: Steps ordered by `position`; assertions ordered by `position`
- BR-009-10: `slug` is UNIQUE per project

**Code paths**: `supabase/migrations/0007_save_atc.sql`, `supabase/migrations/0004_atcs.sql`, `app/(app)/projects/[projectSlug]/atcs/[atcId]/page.tsx`

---

### Flow 4 — Project Tree Loading (Browse ATCs)

```
+----------+  GET /projects/{slug}  +-------------------+
|  QA Eng  |----------------------->| Next.js Server    |
| (browser)|                        | Component (RSC)   |
+----------+                        +--------+----------+
                                             |
                      +----------------------+
                      | Supabase server client (cookie-auth)
                      |
             +--------v--------+
             | SELECT projects  |
             | WHERE slug = ?   |
             | (RLS: member)    |
             +--------+---------+
                      |
             +--------v---------+
             | SELECT modules   |
             | WHERE project_id |
             | ORDER position   |
             +--------+---------+
                      |
             +--------v---------+   Parallel (Promise.all)
             | SELECT           +---+
             |  user_stories    |   |
             |  IN(module_ids)  |   | SELECT atcs
             +------------------+   | WHERE project_id
                                    +--------+--------+
                                             |
                      +--------------------+ |
                      | SELECT             | |
                      |  acceptance_       | |
                      |  criteria          | |
                      |  IN(story_ids)     | |
                      +--------------------+ |
                                             |
                             +--------------v----------+
                             | buildModuleTree()        |
                             | (client-side tree build) |
                             | module → stories → atcs  |
                             +--------------------------+
                                             |
                             +--------------v----------+
                             | Render: Sidebar tree     |
                             | + AtcTable rows          |
                             +-------------------------+
```

**Business Rules**:
- BR-010-3: Module tree max depth 6; path is materialized (`auth/login`)
- BR-010-4: `(project_id, path)` unique — no two modules share a path
- BR-010-7: All modules/stories/ATCs/ACs loaded in parallel for the project view

**Code paths**: `app/(app)/projects/[projectSlug]/page.tsx`, `lib/tree.ts`

---

### Flow 5 — PAT Generation + Authentication

```
--- ISSUANCE (session-authenticated only) ---

+----------+  POST /api/v1/tokens       +-------------------+
|  User    |--------------------------->| API Route         |
| (cookie) |  { scopes, name,           |                   |
|          |    workspace_id?,          | 1. Verify session |
|          |    expires_in_days? }      | 2. Zod validate   |
+----------+                            | 3. generateSecret |
                                        |    (32 bytes,     |
                                        |     base64url)    |
                                        | 4. tokenPrefix =  |
                                        |    secret[0:12]   |
                                        | 5. sha256Hex      |
                                        +--------+----------+
                                                 |
                              +------------------+
                              | admin client (service_role)
                              |
                     +--------v--------+    +---------------------------+
                     | INSERT          |    | INSERT                    |
                     | access_tokens   |    | access_token_secrets      |
                     | (no hash here)  |    | { token_id, hash }        |
                     +--------+--------+    | (QA roles cannot read)    |
                              |             +---------------------------+
                     +--------v--------+
                     | Response: 201   |
                     | {               |
                     |   token:        |
                     |   "bk_pat_<12>  |
                     |    .<remainder>"|
                     |   warning: "..."|}
                     +-----------------+

--- AUTHENTICATION (every bearer request) ---

+----------+  GET/POST /api/v1/*        +-------------------+
|  Client  |--------------------------->| lib/api/auth.ts   |
+----------+  Authorization:            |                   |
             Bearer bk_pat_<p>.<r>      | requireAuth()     |
                                        | detects Bearer    |
                                        +--------+----------+
                                                 |
                                        lib/api/middleware/bearer.ts
                                        requireBearerToken()
                                                 |
                               +------ strip "bk_pat_" prefix
                               |       split at first "."
                               |       prefix = first 12 chars
                               |       fullSecret = prefix + remainder
                               |
                      +--------v--------+
                      | SELECT          |
                      | access_tokens   |
                      | WHERE           |
                      | token_prefix=?  |
                      | (O(1) index)    |
                      +--------+--------+
                               |
                      +--------v--------+
                      | SELECT          |
                      | access_token_   |
                      | secrets         |
                      | WHERE token_id  |
                      | IN candidates   |
                      +--------+--------+
                               |
                      +--------v--------+
                      | sha256(full     |
                      | Secret) ==      |
                      | stored hash?    |
                      | revoked_at NULL?|
                      | expires_at ok?  |
                      +--------+--------+
                               |
                      +--------v--------+
                      | fire-and-forget |
                      | UPDATE          |
                      | last_used_at    |
                      +-----------------+
                               |
                      +--------v--------+
                      | Returns:        |
                      | { userId,       |
                      |   workspaceId,  |
                      |   scopes,       |
                      |   tokenId }     |
                      +-----------------+
```

**Business Rules**:
- BR-006-1: PAT issuance requires cookie session — PAT cannot create another PAT
- BR-006-2: Token format `bk_pat_<12-char-prefix>.<secret>` — family prefix enables GitHub/GitGuardian detection
- BR-006-3: 256-bit random secret; only SHA-256 stored; raw shown once
- BR-006-4: O(1) lookup via indexed `token_prefix`; constant-time SHA-256 compare
- BR-006-8: Revocation is soft-delete (`revoked_at` stamp, no physical DELETE)
- Bearer auth: all failures collapse to uniform `unauthorized` (401) — never reveals which check failed

**Code paths**: `lib/api/pat.ts`, `lib/api/middleware/bearer.ts`, `lib/api/auth.ts`, `app/api/v1/tokens/route.ts`, `supabase/migrations/0008_access_tokens.sql`, `supabase/migrations/0011_split_token_secrets.sql`

---

### Flow 6 — Magic Link Authentication (Browser)

```
+----------+  POST /api/v1/auth/magic-link  +-------------------+
|  User    |------------------------------>| API Route         |
| (anon)   |  { email, next? }             |                   |
+----------+                               | 1. Zod validate   |
                                           | 2. open-redirect  |
                                           |    guard on next  |
                                           +--------+----------+
                                                    |
                                 supabase.auth.signInWithOtp()
                                 emailRedirectTo: /auth/callback
                                                    |
                              +---------------------+
                              | Supabase Auth (email provider)
                              | sends OTP email
                              +---------------------+
                                                    |
                              Best-effort (void, swallowed on error):
                              recordIssuance() → INSERT magic_link_tokens
                                               + INSERT magic_link_token_secrets
                                                    |
                                         +---------v---------+
                                         | Response: 200     |
                                         | { ok: true }      |
                                         +-------------------+

--- OTP CLICK (user email client) ---

+----------+  GET /auth/callback?code=&next=  +-------------------+
|  User    |--------------------------------->| Next.js Route     |
| (email   |                                  | app/auth/callback |
|  client) |                                  |                   |
+----------+                                  | supabase.auth.    |
                                              | exchangeCode      |
                                              | ForSession(code)  |
                                              +--------+----------+
                                                       |
                                     Sets cookie: sb-<ref>-auth-token
                                     Redirects to: /projects (or next)
```

**Business Rules**:
- BR-001-2: `next` param must start with `/` and not `//` — open-redirect guard
- BR-001-3: Supabase 429 → propagated as `rate_limited` (never masked as 200)
- BR-001-4: Magic link audit is best-effort — failure NEVER blocks the auth flow

**Code paths**: `app/api/v1/auth/magic-link/route.ts`, `app/auth/callback/route.ts`, `middleware.ts`

---

### Flow 7 — Headless Signup (CLI / AI Agent Bootstrap)

```
+----------+  POST /api/v1/auth/signup  +-------------------+
|  CLI /   |-------------------------->| API Route         |
|  Agent   |  { email, password,       |                   |
|  (anon)  |    pat_name?,             | 1. Zod validate   |
|          |    pat_scopes?,           | 2. admin.auth.    |
|          |    pat_expires_in_days? } |    createUser     |
+----------+                           |    (email_confirm:|
                                       |     true)         |
                                       +--------+----------+
                                                |
                                       +--------v-----------+
                                       | supabase.auth.     |
                                       | signInWithPassword |
                                       | (gets session)     |
                                       +--------+-----------+
                                                |
                                       +--------v-----------+
                                       | mintPat()           |
                                       | INSERT access_tokens|
                                       | INSERT access_token_|
                                       |   secrets           |
                                       +--------+-----------+
                                                |
                                       +--------v-----------+
                                       | Response: 201      |
                                       | { user, session,   |
                                       |   pat: { token,    |
                                       |          scopes }, |
                                       |   warning }        |
                                       +--------------------+
```

**Business Rules**:
- BR-002-2: `email_confirm: true` — no confirmation email. Intentional for headless provisioning.
- BR-002-3: User created + signed in + PAT minted in a single round trip
- BR-002-4: PAT shown once, `warning` field always present
- BR-002-1: If email already exists → `conflict` (409). Caller should use `/auth/signin` instead.

**Code paths**: `app/api/v1/auth/signup/route.ts`, `lib/api/pat.ts`

---

### Flow 8 — Identity Introspection (`/me`)

```
+----------+  GET /api/v1/me  +-------------------+
|  Any     |----------------->| API Route         |
|  Client  |  Cookie OR Bearer|                   |
+----------+                  | requireAuth()     |
                               | → source:        |
                               |   cookie|bearer  |
                               +----+----+--------+
                                    |
               +--------------------+--------------------+
               | Cookie path                Bearer path   |
               |                                          |
       +-------v-------+                 +---------------v-----+
       | createClient() |                | admin client         |
       | RLS-backed     |                | workspace_members    |
       | SELECT         |                | JOIN workspaces      |
       | workspaces     |                | WHERE user_id = ?    |
       +-------+--------+                | AND status = active  |
               |                         +---------------+------+
               |                                         |
       +-------v-----------------------------------------v-------+
       | Resolve active_workspace_id:                            |
       |  Cookie: read bk_active_ws cookie → validate visible   |
       |  Bearer: use token.workspace_id → fallback oldest ws   |
       +---------------------------------------------------------+
               |
       +-------v---------------------------+
       | Response: 200                     |
       | { user: { id, email },            |
       |   workspaces[],                   |
       |   active_workspace_id,            |
       |   auth: { source, scopes } }      |
       +-----------------------------------+
```

**Business Rules**:
- BR-011-2: For cookie callers, `active_workspace_id` from `bk_active_ws` cookie with visible-workspace validation
- BR-011-5: Cookie callers have `auth.scopes = []` — the cookie session is unscoped
- BR-011-4: Bearer callers: email fetched best-effort; failure → email=null, request still 200

**Code paths**: `app/api/v1/me/route.ts`, `lib/api/auth.ts`

---

### Flow 9 — Plan Enforcement (Community / Cloud / Enterprise)

```
+----------+  POST /api/v1/workspaces  +-------------------+
|  User    |-------------------------->| API Route         |
+----------+                           | (bunkai_bootstrap)|
                                       | Plan defaults to  |
                                       | 'community' on    |
                                       | INSERT            |
                                       +--------+----------+
                                                |
                              +----------+-----------+
                              | workspaces.plan       |
                              | CHECK constraint:     |
                              | 'community'|'cloud'|  |
                              | 'enterprise'          |
                              +-----------+-----------+
                                          |
                              (Phase 2 — not yet active)
                                          |
                              +----------v-----------+
                              | feature_flags table   |
                              | scope: global OR      |
                              | workspace             |
                              | No client read for    |
                              | flag enforcement yet  |
                              +----------------------+
```

**Business Rules**:
- Plan values constrained by CHECK in `workspaces` table
- All new workspaces start `community`
- Feature gating logic exists in schema (feature_flags table) but no flags are read in current codebase (Phase 2)

**Code paths**: `supabase/migrations/0001_tenancy.sql`, `supabase/migrations/0009_cross_cutting.sql`

---

## 4. State Machines

### SM-1: ATC Execution Status (`atcs.status`)

```
                   +----------+
             +---->|  unrun   |<----+
             |     | (default)|     |
             |     +----+-----+     |
             |          |           |
             |   +-------+------+   | [re-run]
             |   |       |      |   |
             | [skip]  [start] [block]
             |   |       |      |   |
             |   v       v      v   |
             | skipped running blocked
             |           |          |
             |     [pass]| [fail]   |
             |       +---+---+      |
             |       |       |      |
             |       v       v      |
             +---- pass    fail ----+
```

| From | To | Event | Effects |
|------|----|-------|---------|
| `unrun` | `running` | Test execution starts | Signals active run |
| `unrun` | `skipped` | Test explicitly skipped | No execution attempted |
| `unrun` | `blocked` | Pre-condition not met | Run cannot proceed |
| `running` | `pass` | All assertions pass | Positive result recorded |
| `running` | `fail` | Any assertion fails | Failure recorded; bug may be filed |
| `pass` / `fail` / `blocked` / `skipped` | `unrun` | Re-run requested | Clears last result for fresh execution |

**Business Rules**:
- Any status can return to `unrun` — there is no terminal state
- Status reflects the LAST known execution result, not a run history (no run table yet in current schema)
- `running` status may be stale if an execution crashed — no timeout/cleanup mechanism in current schema

---

### SM-2: WorkspaceMember Status (`workspace_members.status`)

```
  [invite issued]                [admin suspends]
        |                               |
        v                               v
   +--------+    [accept invite]   +--------+     [reinstate]    +-----------+
   |invited |-------------------->| active |-------------------->| suspended |
   +--------+                     +---+----+                     +-----+-----+
                                      |                                |
                                      +--------------------------------+
                                           [reinstate]
```

| From | To | Event | Effects |
|------|----|-------|---------|
| (none) | `invited` | Admin/owner issues invite | `workspace_invites` row created |
| `invited` | `active` | Invitee accepts invite token | `workspace_invites.accepted_at` stamped |
| `active` | `suspended` | Admin/owner suspends member | Member loses all RLS access |
| `suspended` | `active` | Admin/owner reinstates | RLS access restored |

**Business Rules**:
- Only `admin` and `owner` can INSERT/UPDATE `workspace_members` (RLS)
- `owner` role cannot be granted via invite — only via direct admin action
- A user with no `active` membership row sees NO workspace data at all (RLS enforcement)

---

### SM-3: PAT Lifecycle (`access_tokens`)

```
   [POST /api/v1/tokens]
            |
            v
   +------------------+    [every use]    +------------------+
   |    active        |----------------->|  active          |
   | (revoked_at null,|  last_used_at    |  (same, updated) |
   |  not expired)    |  updated         +------------------+
   +--------+---------+
            |
     +-------+-------+
     |               |
[DELETE /tokens/{id}] [expires_at < now()]
     |               |
     v               v
 +--------+      +--------+
 | revoked|      | expired|
 | (soft  |      | (app-  |
 |  delete)|     |  layer)|
 +--------+      +--------+
```

| From | To | Event | Effects |
|------|----|-------|---------|
| (none) | `active` | POST /api/v1/tokens (cookie only) | Raw token shown once; hash stored |
| `active` | `active` | Bearer request authenticated | `last_used_at` updated (fire-and-forget) |
| `active` | `revoked` | DELETE /api/v1/tokens/{id} | `revoked_at` stamp; row NOT deleted |
| `active` | `expired` | `expires_at < now()` check | App-layer rejection in bearer middleware |

**Business Rules**:
- `revoked` and `expired` are TERMINAL — no path back to `active`
- No physical DELETE policy exists — tokens remain for audit purposes
- A PAT cannot be used to create another PAT (session required for issuance)

---

### SM-4: WorkspaceInvite Lifecycle (`workspace_invites`)

```
   [POST /workspaces/{id}/invites]
            |
            v
   +------------------+
   |    pending       |  (expires_at = now + 7 days)
   +--------+---------+
            |
     +-------+-------+-------+
     |       |               |
[accept]  [revoke]    [expires_at < now()]
     |       |               |
     v       v               v
 +--------+ +-------+   +--------+
 |accepted| |revoked|   |expired |
 +--------+ +-------+   +--------+
```

| From | To | Event | Effects |
|------|----|-------|---------|
| `pending` | `accepted` | POST /invites/accept with valid token | `accepted_at` stamped; `workspace_members` upserted |
| `pending` | `revoked` | Admin/owner calls DELETE on invite | `revoked_at` stamped |
| `pending` | `expired` | `expires_at < now()` | App-layer check; no DB column transition |

**Business Rules**:
- `status` in API response is derived (`pending | accepted | revoked | expired`) — not a column
- Only workspace admin/owner can view invite list (RLS on `workspace_invites`)
- Email matching at acceptance: caller's auth email must equal invite email (case-insensitive)
- Token hash stored in `workspace_invite_secrets` (service_role only) — raw token shown once at issuance

---

### SM-5: IdempotencyKey (`idempotency_keys.status`)

```
   [POST with Idempotency-Key header]
            |
            v
       +----------+
       | pending  |
       +----+-----+
            |
     +-------+-------+
     |               |
[response stored]  [error occurred]
     |               |
     v               v
 +----------+   +--------+
 | succeeded|   | failed |
 +----------+   +--------+
```

**Note**: Idempotency infrastructure exists (`lib/api/idempotency.ts`) but is not currently wired to any API route.

---

## 5. Automatic Processes

### DB Triggers

| Trigger | Table | Fires on | What it does | Why it exists |
|---------|-------|----------|--------------|---------------|
| `atcs_set_updated_at` | `atcs` | BEFORE UPDATE | Sets `updated_at = now()` | Keeps modification timestamp fresh without app code remembering to set it |
| `atcs_refresh_tsv` | `atcs` | BEFORE INSERT OR UPDATE of `title`, `tags` | Rebuilds `tsv = to_tsvector('english', title + tags)` | Powers the GIN-indexed full-text search UI without application-layer sync |

**Functions behind triggers**: `bunkai_set_updated_at()`, `bunkai_atcs_refresh_tsv()` — both `SECURITY DEFINER` with `set search_path = ''` per Supabase linter requirement.

### Cron Jobs

| Job | Schedule | What it does | Why it exists |
|-----|----------|--------------|---------------|
| (none found) | — | — | No cron jobs configured in current migrations or package.json. Idempotency key TTL cleanup (24h rows in `idempotency_keys`) would be a natural candidate — currently no cleanup job exists. |

### Webhooks / Event Handlers

| Hook | Trigger | What it does | Why it exists |
|------|---------|--------------|---------------|
| `recordIssuance` (best-effort) | POST /api/v1/auth/magic-link | Writes to `magic_link_tokens` + `magic_link_token_secrets` | Audit trail for rate-limiting signals; failure swallowed — never blocks auth |
| `touchLastUsed` (fire-and-forget) | Every successful Bearer auth | UPDATE `access_tokens.last_used_at` | Tracks when PATs are in use for auditing and TTL decisions; failure swallowed |
| `console.log` (stdout) | POST /api/v1/workspaces/{id}/invites | Logs `accept_url` to stdout | MVP has no transactional email — inviter must copy the URL from response or server logs |

---

## 6. External Integrations

### Supabase Auth

```
+----------+  signInWithOtp(email)    +--------------------+
| Bunkai   |------------------------->| Supabase Auth      |
| API      |                          | (email provider)   |
+----+-----+                          +----------+---------+
     |                                           |
     |  OTP email sent to user                   |
     |                              User clicks link
     |                                           |
+----v-----+  exchangeCodeForSession  +----------+---------+
| /auth/   |<-------------------------| Supabase OTP       |
| callback |  returns session         | exchange endpoint  |
+----+-----+                          +--------------------+
     |
     | Sets sb-<project-ref>-auth-token cookie
     | Redirects to /projects
```

**Data impact**: Supabase Auth manages `auth.users`. Bunkai reads `auth.uid()` in RLS policies. Session cookies are managed entirely by Supabase SSR (`@supabase/ssr`).

**Failure behavior**: If Supabase Auth is down, magic-link issuance fails with `upstream_error` (502). Bearer PAT auth uses only the Postgres DB — immune to Auth service availability.

**Dependent flows**: Flow 6 (Magic Link Auth), Flow 7 (Headless Signup), Flow 3 (RPC auth context)

---

### Supabase Postgres (Primary DB)

```
+----------+  Supabase JS SDK         +--------------------+
| Bunkai   +------------------------->| PostgREST (RLS on) |
| (Server  |  createClient()          +--------------------+
| Component|                                    |
| / API)   |  createAdminClient()     +--------------------+
+----------+  (SUPABASE_SERVICE_ROLE) | Postgres 17 + RLS  |
             +------------------------> (bypasses RLS)     |
                                      +--------------------+
                                               |
                                      +--------v-----------+
                                      | RPCs (SECURITY     |
                                      | DEFINER / INVOKER) |
                                      | bunkai_bootstrap_  |
                                      |   workspace        |
                                      | bunkai_save_atc    |
                                      +--------------------+
```

**Three client modes**:

| Client | When used | RLS |
|--------|-----------|-----|
| `createClient()` (server) | Browser routes, middleware, session-auth operations | Active — `auth.uid()` from cookie |
| `createAdminClient()` (service_role) | Secret table writes, cross-user queries after explicit auth | Bypassed — explicit membership checks in code |
| `createClient()` (browser) | Client components | Active — session from cookie |

**Failure behavior**: Full application outage. All data reads and writes depend on Supabase Postgres.

---

### Vercel (Hosting)

```
+----------+  Next.js build         +--------------------+
| GitHub   |----------------------->| Vercel CI/CD       |
| (source) |                        +----------+---------+
                                               |
+----------+  HTTP request          +----------+---------+
| User /   |----------------------->| Vercel Edge / CDN  |
| Agent    |                        | Serverless Functions|
+----------+                        +--------------------+
```

**Data impact**: No data stored in Vercel. Environment variables (Supabase keys, app URL) configured in Vercel project settings.

**Failure behavior**: Application unavailable. No failover mechanism documented.

---

### Jira Cloud (Reference Only)

```
+----------+  Store external_id   +--------------------+
| QA Eng   +--------------------->| user_stories table |
| (browser)|  "PROJ-123"          | external_id field  |
+----------+  external_url        +--------------------+
             "https://...         
              atlassian.net/      (NO live API calls)
              browse/PROJ-123"   
```

**Data impact**: `user_stories.external_id` and `user_stories.external_url` store Jira keys as plain text. No live sync, no webhook, no API call from the app to Jira.

**Failure behavior**: No impact — the link is decorative/navigational only.

**Dependent flows**: None that depend on Jira availability.

---

### DBHub (QA Inspection — MCP Tool)

```
+----------+  dbhub.toml config    +--------------------+
| AI Agent |--------------------->| DBHub MCP          |
| / QA Eng |  qa_inspector_ro/rw  | (bytebase)         |
+----------+  credentials         +----------+---------+
                                             |
                                    +--------v-----------+
                                    | Postgres 17        |
                                    | BYPASSRLS          |
                                    | REVOKE on secret   |
                                    | sibling tables     |
                                    +--------------------+
```

**Data impact**: Read-only inspection of all non-secret tables. `qa_inspector_rw` can write test data directly for fixture setup.

**Failure behavior**: QA tooling impact only — application unaffected.

---

## 7. Discovery Gaps

The following items could not be fully verified from available source files:

| Gap | Confidence | Notes |
|-----|-----------|-------|
| `auth/callback` route implementation | Medium | File referenced in `backend.md` as `app/auth/callback/route.ts`. Existence confirmed in backend.md source list but full code not read. OTP exchange logic assumed from Supabase SSR pattern. |
| ATC slug generation strategy | Low | `atcs.slug` is UNIQUE per project but no server-side slug generation was found in any API route or migration. How slugs are initially created is unknown. |
| `qa_inspector_ro` / `qa_inspector_rw` CREATE ROLE DDL | Low | Referenced in migration 0011 `REVOKE` statements but the `CREATE ROLE` DDL is not in any migration file. Provisioned out-of-band via Supabase Studio. |
| Feature flag enforcement logic | Low | `feature_flags` table exists with `enabled` + `payload` columns. No code in current routes reads feature flags. Phase 2 mechanism — no flag keys documented. |
| Plan feature gating matrix | Low | `workspaces.plan` has 3 values (`community / cloud / enterprise`) but no code gates features by plan in the current codebase. Exact limits per plan not documented. |
| `user_view_state.view_kind` valid values | Low | Column is `text` with no CHECK constraint. Valid values not documented in migrations or routes. |
| `activity_log.entity_type` and `action` valid values | Low | No CHECK constraint. No examples in migration comments or route code. |
| ATC run/execution history entity | Low | `atcs.status` tracks the LAST result only. No `test_runs`, `test_run_items`, or `run_results` table exists. "Sprint 2" per `qa-config.ts` — test execution history is not yet persisted. |
| Invite token rotation endpoint | Medium | `POST /api/v1/workspaces/{id}/invites/{inviteId}` (rotate) is listed in the API catalog (`backend.md`) but the route file was not read. Behavior assumed from API description. |
| Staging/production Supabase project reference | Low | Needed to construct cookie name `sb-<project-ref>-auth-token` for test fixture configuration. Not in `.env.example`. |
| `NEXT_PUBLIC_SUPABASE_ANON_KEY` vs `SUPABASE_PUBLISHABLE_KEY` env var mismatch | High | `middleware.ts` reads `NEXT_PUBLIC_SUPABASE_ANON_KEY`; `.env.example` defines `SUPABASE_PUBLISHABLE_KEY`. Both must be set to the same value — setting only one will break the middleware. |
| Idempotency wiring | High | `lib/api/idempotency.ts` exists and `idempotency_keys` table is created in migration 0009, but no `app/api/v1/` route currently calls `beginIdempotentRequest`. The feature is available but dormant. |
| Project / Module CRUD endpoints | Medium | No `POST /api/v1/workspaces/{id}/projects` or module management REST endpoints were found. These operations appear to go through PostgREST directly, which means RLS is the only guard and there is no application-layer validation layer. |

---

*Sources: supabase/migrations/0001–0012, app/api/v1/**, lib/api/**, app/(app)/projects/**, existing context files — business-model.md, domain-glossary.md, functional-specs.md, architecture.md, backend.md*
