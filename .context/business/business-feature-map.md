# Business Feature Map — upex-bunkai-tms

> Generated: 2026-06-08 | Mode: CREATE | Sources: supabase/migrations/0001–0012, app/api/v1/**, app/(app)/**, app/(auth)/**, components/**, lib/**, package.json, .context/business/business-data-map.md, .context/SRS/functional-specs.md, .context/infrastructure/backend.md

```
+-----------------------------------------------------------------------+
|                         BUNKAI TMS (分解)                             |
|   Multi-tenant Test Management System — Feature Catalog               |
|   Next.js 15 · Supabase Postgres 17 · Bun · TypeScript               |
+-----------------------------------------------------------------------+
```

---

## 1. Inventory Summary

| Category  | Features | Count | Status                              |
|-----------|----------|-------|-------------------------------------|
| Core      | Auth & Identity, Workspace & Tenancy, ATC Authoring, Project Tree Navigation | 15 | Stable |
| Secondary | Invite Management, PAT Management, Members UI, QA Onboarding, API Docs, OpenAPI | 9  | Stable |
| Beta      | Command Palette Search, Active Workspace Preference | 2 | Partial — stub/skeleton visible but not wired to backend search |
| Planned   | OAuth (GitHub/Google), Project Creation UI, Test Run History, Feature Flag Enforcement, Idempotency Wiring | 5 | Planned / In Development |

**Total features: 31** (15 Core + 9 Secondary + 2 Beta + 5 Planned)

---

## 2. Feature Catalog by Domain

---

### Domain A — Auth & Identity

#### Feature: Magic Link Authentication (Browser)

| Aspect           | Value |
|------------------|-------|
| **ID**           | FEAT-001 |
| **Status**       | Stable |
| **Endpoints**    | `POST /api/v1/auth/magic-link`, `GET /auth/callback` |
| **UI**           | `/login` page → `MagicLinkForm` component → "Check your inbox" state |
| **Users**        | Any unauthenticated user |
| **Dependencies** | Supabase Auth (email OTP provider), `magic_link_tokens` table (best-effort audit) |
| **Evidence**     | `app/api/v1/auth/magic-link/route.ts`, `app/(auth)/login/`, `app/auth/callback/route.ts` |

**Capabilities:**
- [x] Email OTP issuance via `supabase.auth.signInWithOtp()`
- [x] Open-redirect guard on `next` param (must start with `/`, not `//`)
- [x] Supabase 429 propagated as `rate_limited` (not masked as 200)
- [x] Best-effort audit row written to `magic_link_tokens` + `magic_link_token_secrets`
- [x] Default post-auth redirect to `/projects`
- [x] OTP exchange via `exchangeCodeForSession(code)` at `/auth/callback`
- [ ] Email actually sent in MVP (Supabase manages email; not configurable in app)

---

#### Feature: Headless Signup (CLI / AI Agent Bootstrap)

| Aspect           | Value |
|------------------|-------|
| **ID**           | FEAT-002 |
| **Status**       | Stable |
| **Endpoints**    | `POST /api/v1/auth/signup` |
| **UI**           | None — headless API only |
| **Users**        | CI bots, AI agents, SDET scripts |
| **Dependencies** | Supabase Auth admin SDK, `mintPat()` lib, `access_tokens` + `access_token_secrets` tables |
| **Evidence**     | `app/api/v1/auth/signup/route.ts`, `lib/api/pat.ts` |

**Capabilities:**
- [x] Create user with `email_confirm: true` (no confirmation email — intentional)
- [x] Sign in immediately and mint PAT in the same round trip
- [x] Return `{ user, session, pat }` in 201 response with persistent `warning` field
- [x] Conflict (409) if email already registered
- [x] `pat_scopes` defaults to all allowed scopes if omitted
- [x] `pat_expires_in_days` max 365; null = never expires
- [ ] Duplicate email path should redirect to `/auth/signin` — callers must handle 409 themselves

---

#### Feature: Headless Sign-In (CLI / Agent)

| Aspect           | Value |
|------------------|-------|
| **ID**           | FEAT-003 |
| **Status**       | Stable |
| **Endpoints**    | `POST /api/v1/auth/signin` |
| **UI**           | None — headless API only |
| **Users**        | CI bots, AI agents, existing QA bot users |
| **Dependencies** | `supabase.auth.signInWithPassword()`, `mintPat()` |
| **Evidence**     | `app/api/v1/auth/signin/route.ts`, `lib/api/pat.ts` |

**Capabilities:**
- [x] Authenticate with email + password
- [x] Uniform 401 on wrong email or wrong password (no email enumeration)
- [x] Returns `{ user, session, pat }` — PAT minted in one round trip
- [x] Same PAT options as FEAT-002 (scopes, expiry, name)

---

#### Feature: Identity Introspection (`/me`)

| Aspect           | Value |
|------------------|-------|
| **ID**           | FEAT-004 |
| **Status**       | Stable |
| **Endpoints**    | `GET /api/v1/me` |
| **UI**           | None — used by SDK integrations and agent bootstrap |
| **Users**        | Any authenticated user (cookie or Bearer) |
| **Dependencies** | `lib/api/auth.ts`, `access_tokens` table, `workspace_members` table |
| **Evidence**     | `app/api/v1/me/route.ts`, `lib/api/auth.ts` |

**Capabilities:**
- [x] Returns `user.id`, `user.email`, `workspaces[]`, `active_workspace_id`, `auth.source`, `auth.scopes`
- [x] Cookie callers: `active_workspace_id` resolved from `bk_active_ws` cookie with visible-workspace fallback
- [x] Bearer callers: `active_workspace_id` from token's `workspace_id` field; fallback to oldest visible workspace
- [x] Bearer callers: email fetched best-effort (null on failure, request still 200)
- [x] Cookie callers: `auth.scopes = []` (unscoped — UI gates apply)
- [x] Dual-path auth: both cookie session and Bearer PAT supported

---

#### Feature: Active Workspace Preference

| Aspect           | Value |
|------------------|-------|
| **ID**           | FEAT-005 |
| **Status**       | Beta (wired but no UI switcher beyond `WorkspaceSwitcher` component) |
| **Endpoints**    | `POST /api/v1/me/active-workspace` |
| **UI**           | `WorkspaceSwitcher` component in `Topbar` |
| **Users**        | Authenticated browser users (cookie only) |
| **Dependencies** | `lib/api/workspace-cookie.ts`, `bk_active_ws` cookie |
| **Evidence**     | `app/api/v1/me/active-workspace/route.ts`, `components/layout/WorkspaceSwitcher.tsx` |

**Capabilities:**
- [x] Sets `bk_active_ws` cookie (httpOnly, SameSite=Lax, maxAge=90 days)
- [x] Validates caller is a visible member of the target workspace
- [x] Cookie is a UI preference — does NOT affect JWT or RLS
- [ ] Multi-workspace picker UI: Phase E — current UI only shows current workspace

---

#### Feature: OAuth Login (GitHub / Google)

| Aspect           | Value |
|------------------|-------|
| **ID**           | FEAT-006 |
| **Status**       | Planned |
| **Endpoints**    | None yet |
| **UI**           | Buttons visible on `/login` but disabled (`opacity-60`, `cursor-not-allowed`) |
| **Users**        | All users |
| **Dependencies** | Supabase Auth OAuth providers (not yet configured) |
| **Evidence**     | `app/(auth)/login/page.tsx` — buttons with `title="OAuth ships next sprint"` |

**Capabilities:**
- [ ] GitHub OAuth login
- [ ] Google OAuth login

---

### Domain B — Workspace & Tenancy

#### Feature: Workspace Creation

| Aspect           | Value |
|------------------|-------|
| **ID**           | FEAT-007 |
| **Status**       | Stable |
| **Endpoints**    | `POST /api/v1/workspaces` |
| **UI**           | `/onboarding` → `OnboardingForm` (slug live-preview + validation) |
| **Users**        | Any authenticated user without an existing active workspace |
| **Dependencies** | `bunkai_bootstrap_workspace` RPC (SECURITY DEFINER, atomic) |
| **Evidence**     | `app/api/v1/workspaces/route.ts`, `app/(app)/onboarding/`, `supabase/migrations/0006_bootstrap_workspace.sql` |

**Capabilities:**
- [x] Slug regex enforced: `^[a-z0-9][a-z0-9-]{1,38}[a-z0-9]$`
- [x] Reserved slug list blocked: `admin`, `api`, `app`, `auth`, `docs`, `invites`, `login`, `logout`, `onboarding`, `projects`, `public`, `qa`, `settings`, `static`, `workspaces`, `_next`
- [x] Globally unique slug (409 on collision)
- [x] Atomic creation — workspace + owner membership in single transaction
- [x] Creator auto-assigned `owner` role
- [x] Default plan: `community`
- [x] Onboarding short-circuits to `/projects` if user already has active membership
- [x] Live slug auto-derive from workspace name with `slugify()`

---

#### Feature: Workspace Retrieval and Update

| Aspect           | Value |
|------------------|-------|
| **ID**           | FEAT-008 |
| **Status**       | Stable |
| **Endpoints**    | `GET /api/v1/workspaces`, `GET /api/v1/workspaces/{id}`, `PATCH /api/v1/workspaces/{id}` |
| **UI**           | None dedicated — data consumed by `WorkspaceSwitcher` + `MembersPage` |
| **Users**        | Active workspace members (list/get); owner only (PATCH) |
| **Dependencies** | `workspaces` table, RLS `workspaces_update_owner` policy |
| **Evidence**     | `app/api/v1/workspaces/route.ts`, `app/api/v1/workspaces/[id]/route.ts` |

**Capabilities:**
- [x] `GET /workspaces` — lists only workspaces where caller is active member (RLS for cookie; admin+join for bearer)
- [x] `GET /workspaces/{id}` — 404 if not visible (cookie auth only — no Bearer support on single-workspace GET)
- [x] `PATCH /workspaces/{id}` — updates `name` only; owner-only via RLS; empty PATCH → 400
- [ ] Workspace deletion — RLS policy exists in DB (`workspaces_delete_owner`) but no DELETE route exposed
- [ ] `plan` field is read-only from API — no upgrade flow in current routes

---

#### Feature: Member Suspension / Reinstatement

| Aspect           | Value |
|------------------|-------|
| **ID**           | FEAT-009 |
| **Status**       | Planned (DB supports it via `workspace_members.status` transitions; no API route or UI found) |
| **Endpoints**    | None yet |
| **UI**           | `MembersClient` shows member list + status but no suspend/reinstate action |
| **Users**        | Admin / Owner |
| **Dependencies** | `workspace_members` table — `status` CHECK constraint supports `suspended` |
| **Evidence**     | `supabase/migrations/0001_tenancy.sql`, `app/(app)/workspaces/[id]/members/members-client.tsx` |

**Capabilities:**
- [ ] Suspend member (set `status = 'suspended'`)
- [ ] Reinstate member (set `status = 'active'`)

---

### Domain C — Team Invite Management

#### Feature: Workspace Invite Issuance

| Aspect           | Value |
|------------------|-------|
| **ID**           | FEAT-010 |
| **Status**       | Stable |
| **Endpoints**    | `POST /api/v1/workspaces/{id}/invites` |
| **UI**           | `/workspaces/{id}/members` → `MembersClient` invite form (email + role selector) |
| **Users**        | Admin / Owner |
| **Dependencies** | `workspace_invites` + `workspace_invite_secrets` tables, `generateInviteToken()` + `hashInviteToken()` |
| **Evidence**     | `app/api/v1/workspaces/[id]/invites/route.ts`, `app/(app)/workspaces/[id]/members/` |

**Capabilities:**
- [x] Admin/owner only (RLS `workspace_invites_insert_admin`)
- [x] `owner` role cannot be granted via invite (CHECK constraint: `viewer | member | admin`)
- [x] Raw token + `accept_url` returned once in 201 response
- [x] Token hash stored in `workspace_invite_secrets` (service_role only)
- [x] Accept URL auto-copied to clipboard by UI (`navigator.clipboard`)
- [x] Default expiry: 7 days
- [x] Default role: `member`
- [x] MVP: no transactional email — link shown once in response + toast

---

#### Feature: Workspace Invite List

| Aspect           | Value |
|------------------|-------|
| **ID**           | FEAT-011 |
| **Status**       | Stable |
| **Endpoints**    | `GET /api/v1/workspaces/{id}/invites` |
| **UI**           | `/workspaces/{id}/members` → invite list with derived status badges |
| **Users**        | Admin / Owner |
| **Dependencies** | `workspace_invites` table |
| **Evidence**     | `app/api/v1/workspaces/[id]/invites/route.ts`, `app/(app)/workspaces/[id]/members/members-client.tsx` |

**Capabilities:**
- [x] Returns list with derived `status` field: `pending | accepted | revoked | expired`
- [x] Ordered by `created_at` descending
- [x] Shows only workspace admins/owners (RLS `workspace_invites_select_admin`)

---

#### Feature: Invite Token Rotation

| Aspect           | Value |
|------------------|-------|
| **ID**           | FEAT-012 |
| **Status**       | Stable |
| **Endpoints**    | `POST /api/v1/workspaces/{id}/invites/{inviteId}` |
| **UI**           | "Rotate link" button on pending invites in `MembersClient` |
| **Users**        | Admin / Owner |
| **Dependencies** | `workspace_invites` + `workspace_invite_secrets` tables |
| **Evidence**     | `app/api/v1/workspaces/[id]/invites/[inviteId]/route.ts` |

**Capabilities:**
- [x] Generates new raw token + new secret hash
- [x] Resets `expires_at` to `now + 7 days`
- [x] Clears `accepted_at` and `revoked_at` (re-opens a previously consumed invite)
- [x] Returns new `accept_url` once — auto-copied to clipboard in UI
- [x] RLS-gated: admin/owner of workspace only

---

#### Feature: Invite Revocation

| Aspect           | Value |
|------------------|-------|
| **ID**           | FEAT-013 |
| **Status**       | Stable |
| **Endpoints**    | `DELETE /api/v1/workspaces/{id}/invites/{inviteId}` |
| **UI**           | "Revoke" button on pending invites in `MembersClient` |
| **Users**        | Admin / Owner |
| **Dependencies** | `workspace_invites` table |
| **Evidence**     | `app/api/v1/workspaces/[id]/invites/[inviteId]/route.ts` |

**Capabilities:**
- [x] Stamps `revoked_at` on invite row (soft-delete — no physical DELETE)
- [x] RLS-gated (admin/owner only)
- [x] Revoked invites show `status: 'revoked'` in list

---

#### Feature: Invite Acceptance

| Aspect           | Value |
|------------------|-------|
| **ID**           | FEAT-014 |
| **Status**       | Stable |
| **Endpoints**    | `POST /api/v1/invites/accept` |
| **UI**           | `/invites/accept?token=...` → `AcceptClient` component |
| **Users**        | Authenticated invitee (must be signed in) |
| **Dependencies** | `workspace_invites`, `workspace_invite_secrets`, `workspace_members` tables |
| **Evidence**     | `app/api/v1/invites/accept/route.ts`, `app/invites/accept/` |

**Capabilities:**
- [x] Validates token: not revoked, not accepted, not expired, email match (case-insensitive)
- [x] Upserts `workspace_members` row (`invited → active`)
- [x] Stamps `accepted_at` on invite row
- [x] Token hash looked up in `workspace_invite_secrets` (service_role)
- [x] `?token=` extracted at server-side render — not exposed in client-side URL after acceptance

---

### Domain D — Personal Access Token (PAT) Management

#### Feature: PAT Issuance

| Aspect           | Value |
|------------------|-------|
| **ID**           | FEAT-015 |
| **Status**       | Stable |
| **Endpoints**    | `POST /api/v1/tokens` |
| **UI**           | No dedicated UI — API only (browser users use hybrid path via `cookieMintSnippet`) |
| **Users**        | Session-authenticated users (cookie required — PAT cannot create PAT) |
| **Dependencies** | `access_tokens` + `access_token_secrets` tables, `lib/api/pat.ts` |
| **Evidence**     | `app/api/v1/tokens/route.ts`, `lib/api/pat.ts` |

**Capabilities:**
- [x] Token format: `bk_pat_<12-char-prefix>.<secret>` (family prefix enables secret scanning)
- [x] 256-bit random secret; only SHA-256 hash stored in `access_token_secrets`
- [x] Raw secret shown exactly once in 201 response with mandatory `warning` field
- [x] O(1) lookup by indexed `token_prefix`; constant-time SHA-256 comparison
- [x] Scopes: `atc:read | atc:write | run:execute | workspace:admin` (at least 1 required)
- [x] Optional `workspace_id` scoping (null = global/cross-workspace)
- [x] Optional expiry: `expires_in_days` max 365; null = never expires
- [x] PAT issuance requires cookie session — no PAT-to-PAT issuance

---

#### Feature: PAT List

| Aspect           | Value |
|------------------|-------|
| **ID**           | FEAT-016 |
| **Status**       | Stable |
| **Endpoints**    | `GET /api/v1/tokens` |
| **UI**           | None dedicated |
| **Users**        | Authenticated users (cookie) |
| **Dependencies** | `access_tokens` table, RLS owner policy |
| **Evidence**     | `app/api/v1/tokens/route.ts` |

**Capabilities:**
- [x] Returns caller's own tokens only (RLS `access_tokens_select_own`)
- [x] No hash or secret material in response
- [x] Shows `last_used_at`, `expires_at`, `revoked_at`, `scopes`

---

#### Feature: PAT Revocation

| Aspect           | Value |
|------------------|-------|
| **ID**           | FEAT-017 |
| **Status**       | Stable |
| **Endpoints**    | `DELETE /api/v1/tokens/{id}` |
| **UI**           | None dedicated |
| **Users**        | Authenticated users (cookie) |
| **Dependencies** | `access_tokens` table |
| **Evidence**     | `app/api/v1/tokens/[id]/route.ts` |

**Capabilities:**
- [x] Soft-delete: stamps `revoked_at`, no physical DELETE (audit trail preserved)
- [x] Only caller can revoke own tokens (RLS)
- [x] Revoked tokens immediately rejected by bearer middleware

---

### Domain E — Projects & Modules

#### Feature: Project Navigation and ATC Browse

| Aspect           | Value |
|------------------|-------|
| **ID**           | FEAT-018 |
| **Status**       | Stable |
| **Endpoints**    | None (PostgREST reads in RSC) |
| **UI**           | `/projects/{projectSlug}` — `Topbar` + `Sidebar` (module tree) + `AtcTable` |
| **Users**        | Active workspace members (all roles) |
| **Dependencies** | `projects`, `modules`, `user_stories`, `acceptance_criteria`, `atcs` tables; `buildModuleTree()` lib |
| **Evidence**     | `app/(app)/projects/[projectSlug]/page.tsx`, `lib/tree.ts`, `components/layout/Sidebar.tsx` |

**Capabilities:**
- [x] Project loaded by slug (RLS narrows to caller's workspace)
- [x] Modules, user stories, ATCs, ACs all fetched in parallel (`Promise.all`)
- [x] `buildModuleTree()` assembles client-side tree from flat data
- [x] Sidebar renders module tree with module → story → ATC navigation
- [x] `AtcTable` renders all ATCs with module path, status, layer columns
- [x] `CommandPalette` (search stub via ⌘K) — input visible but no backend search wired
- [x] "New ATC" and "New Test" buttons visible in Topbar — no creation flow wired yet

---

#### Feature: Projects Index (Auto-Redirect)

| Aspect           | Value |
|------------------|-------|
| **ID**           | FEAT-019 |
| **Status**       | Stable |
| **Endpoints**    | None — server-side redirect |
| **UI**           | `/projects` — auto-redirects to first project or shows empty state |
| **Users**        | Authenticated users |
| **Dependencies** | `workspace_members`, `projects` tables |
| **Evidence**     | `app/(app)/projects/page.tsx` |

**Capabilities:**
- [x] No active membership → redirect to `/onboarding`
- [x] Has projects → redirect to first project (by `created_at` ascending)
- [x] No projects yet → empty-state placeholder ("Project creation UI ships in Phase E")

---

#### Feature: Project Creation

| Aspect           | Value |
|------------------|-------|
| **ID**           | FEAT-020 |
| **Status**       | Planned ("ships in Phase E" per code comment) |
| **Endpoints**    | None yet (DB tables + RLS ready: `projects_insert_workspace_role_member_plus`) |
| **UI**           | Empty-state card mentions Phase E; no form |
| **Users**        | Members, admins, owners |
| **Dependencies** | `projects` table (schema ready), `modules` table |
| **Evidence**     | `app/(app)/projects/page.tsx` comment, `supabase/migrations/0002_projects_modules.sql` |

**Capabilities:**
- [ ] Create project (name, slug, description)
- [ ] Create first module
- [ ] Assign to workspace

---

#### Feature: Module Management

| Aspect           | Value |
|------------------|-------|
| **ID**           | FEAT-021 |
| **Status**       | Planned (DB fully ready; no REST endpoint; PostgREST direct access only) |
| **Endpoints**    | None (direct PostgREST from server components) |
| **UI**           | Module tree visible in Sidebar (read-only for current sprint; no create/rename/reorder UI) |
| **Users**        | Members, admins, owners |
| **Dependencies** | `modules` table, RLS helpers |
| **Evidence**     | `supabase/migrations/0002_projects_modules.sql`, `app/(app)/projects/[projectSlug]/page.tsx` |

**Capabilities:**
- [x] Read: modules loaded for project tree display
- [ ] Create module (no UI or REST endpoint)
- [ ] Rename module
- [ ] Reorder siblings (position field exists in schema)
- [ ] Move module to different parent

---

### Domain F — User Stories & Acceptance Criteria

#### Feature: User Story Management

| Aspect           | Value |
|------------------|-------|
| **ID**           | FEAT-022 |
| **Status**       | Planned (DB fully ready; no REST endpoint; PostgREST direct access only) |
| **Endpoints**    | None (PostgREST direct) |
| **UI**           | Stories visible in Sidebar tree + AC picker in ATC editor (read-only); no dedicated story editor page |
| **Users**        | Members, admins, owners |
| **Dependencies** | `user_stories` table, `acceptance_criteria` table |
| **Evidence**     | `supabase/migrations/0003_authoring.sql`, `app/(app)/projects/[projectSlug]/atcs/[atcId]/page.tsx` |

**Capabilities:**
- [x] Read: stories loaded for project tree + ATC editor AC picker
- [x] Optional Jira reference (`external_id`, `external_url` — stored as plain text, no live sync)
- [ ] Create story (no UI or REST endpoint)
- [ ] Edit story (title, description, Jira link)
- [ ] Delete story (blocked by ON DELETE RESTRICT if ATCs are bound)

---

#### Feature: Acceptance Criteria Management

| Aspect           | Value |
|------------------|-------|
| **ID**           | FEAT-023 |
| **Status**       | Planned (DB fully ready; no REST endpoint; PostgREST direct access only) |
| **Endpoints**    | None (PostgREST direct) |
| **UI**           | AC picker in ATC editor (read-only — select for binding; no AC creation form) |
| **Users**        | Members, admins, owners |
| **Dependencies** | `acceptance_criteria` table, `atc_acceptance_criteria` M:N table |
| **Evidence**     | `supabase/migrations/0003_authoring.sql`, `app/(app)/projects/[projectSlug]/atcs/[atcId]/page.tsx` |

**Capabilities:**
- [x] Read: all ACs for project loaded in AC picker (ordered by position)
- [ ] Create AC under a story (no UI or REST endpoint; BK-15 planned)
- [ ] Edit AC (title, description, position)
- [ ] Delete AC (cascades to `atc_acceptance_criteria` bindings — may orphan ATCs)

---

### Domain G — ATC Library

#### Feature: ATC Editor (Authoring + Versioning + AC Linking)

| Aspect           | Value |
|------------------|-------|
| **ID**           | FEAT-024 |
| **Status**       | Stable |
| **Endpoints**    | `bunkai_save_atc` RPC (called via Next.js Server Action `saveAtcAction`) |
| **UI**           | `/projects/{projectSlug}/atcs/{atcId}` — `AtcEditor` component (Monaco editor, `AnchoringPanel`, `StepEditor`, assertions YAML editor) |
| **Users**        | Active members (role >= `member`) |
| **Dependencies** | `atcs`, `atc_steps`, `atc_assertions`, `atc_acceptance_criteria` tables; `bunkai_save_atc` RPC; `@monaco-editor/react` |
| **Evidence**     | `app/(app)/projects/[projectSlug]/atcs/[atcId]/page.tsx`, `app/(app)/projects/[projectSlug]/atcs/[atcId]/actions.ts`, `components/atcs/AtcEditor.tsx`, `supabase/migrations/0007_save_atc.sql` |

**Capabilities:**
- [x] Edit ATC title (required, validated before save)
- [x] Set layer: `UI | API | Unit`
- [x] Manage free-form tags (`text[]`, included in full-text search index)
- [x] Bind to a User Story (`user_story_id` required before save — application-layer guard in `saveAtcAction`)
- [x] Select ≥1 Acceptance Criterion via `AnchoringPanel` (application-layer guard: `acIds.length === 0` → error)
- [x] Edit steps in Monaco (Markdown format parsed by `parseStepsMarkdown()`)
- [x] Edit assertions in Monaco (YAML format parsed by `parseAssertionsYaml()`)
- [x] Full-replace atomic save via `bunkai_save_atc` RPC (single transaction)
- [x] Version increments on every save (`atcs.version + 1`)
- [x] GIN full-text search index refreshed by trigger on title + tags
- [x] `revalidatePath` on save — RSC data refreshed after mutation
- [ ] ATC creation (new ATC from scratch) — "New ATC" button visible but no creation flow wired
- [ ] ATC deletion — no route or UI found
- [ ] ATC search UI (CommandPalette stub present; no backend search route wired)

**KEY GAP — bunkai_save_atc empty ac_ids**: The DB-level RPC (`migration/0007`) does NOT validate that `p_ac_ids` is non-empty. It proceeds with `if p_ac_ids is not null`. The application-layer guard in `saveAtcAction` (`acIds.length === 0 → error`) is the ONLY enforcement. A direct RPC call bypassing the Server Action can create an ATC with zero AC bindings, violating the anchoring moat.

---

#### Feature: ATC Full-Text Search (GIN Index)

| Aspect           | Value |
|------------------|-------|
| **ID**           | FEAT-025 |
| **Status**       | Beta (index and trigger are live; no search API endpoint; CommandPalette UI stub present) |
| **Endpoints**    | None yet |
| **UI**           | `CommandPalette` (⌘K) — search input visible, placeholder "Search ATCs, modules, user stories…" but no backend call wired |
| **Users**        | All workspace members |
| **Dependencies** | `atcs.tsv` GIN column, `bunkai_atcs_refresh_tsv()` trigger |
| **Evidence**     | `supabase/migrations/0004_atcs.sql`, `components/layout/CommandPalette.tsx` |

**Capabilities:**
- [x] GIN index on `atcs.tsv` (title + tags) for `to_tsvector('english', ...)` queries
- [x] Trigger keeps `tsv` fresh on every INSERT or title/tags UPDATE
- [ ] REST endpoint for full-text ATC search
- [ ] CommandPalette wired to search endpoint

---

### Domain H — Test Execution

#### Feature: Test Run History (ATC Status Tracking)

| Aspect           | Value |
|------------------|-------|
| **ID**           | FEAT-026 |
| **Status**       | Planned (status field exists on `atcs` table; no run table; no execution API) |
| **Endpoints**    | None yet — `run:execute` scope reserved in PAT system |
| **UI**           | ATC status (`unrun | pass | fail | blocked | skipped | running`) visible in `AtcTable` |
| **Users**        | PAT holders with `run:execute` scope (Sprint 2) |
| **Dependencies** | `atcs.status` column (current last-result only; no history table) |
| **Evidence**     | `app/qa/qa-config.ts` (`patScopes` entry: "Iniciar runs + postear resultados de steps (Sprint 2)"), `lib/types.ts` (`AtcStatus` type) |

**Capabilities:**
- [x] ATC status field: `unrun | pass | fail | blocked | skipped | running` (displayed in `AtcTable`)
- [x] `run:execute` scope defined in PAT scopes — reserved for Sprint 2
- [ ] Test run entity (`test_runs`, `test_run_items` tables — not yet in any migration)
- [ ] Execute run API endpoint (`POST /api/v1/runs`)
- [ ] Per-step result recording
- [ ] Run history (current schema only tracks last result on `atcs.status`)
- [ ] Idempotency cleanup job for `idempotency_keys` table 24h TTL rows

---

### Domain I — Platform Infrastructure

#### Feature: API Discovery Banner

| Aspect           | Value |
|------------------|-------|
| **ID**           | FEAT-027 |
| **Status**       | Stable |
| **Endpoints**    | `GET /api/v1`, `OPTIONS /api/v1` |
| **UI**           | None — machine-readable JSON |
| **Users**        | Any client |
| **Dependencies** | None |
| **Evidence**     | `app/api/v1/route.ts` |

**Capabilities:**
- [x] Returns version, openapi spec URL, docs URL
- [x] OPTIONS → 204 CORS preflight with `authorization, content-type, idempotency-key, x-request-id` headers

---

#### Feature: Health / Liveness Probe

| Aspect           | Value |
|------------------|-------|
| **ID**           | FEAT-028 |
| **Status**       | Stable |
| **Endpoints**    | `GET /api/v1/health` |
| **UI**           | None |
| **Users**        | CI/CD, load balancers, uptime monitors |
| **Dependencies** | None (no DB call) |
| **Evidence**     | `app/api/v1/health/route.ts` |

**Capabilities:**
- [x] Public endpoint (no auth required)
- [x] Returns environment info + timestamp

---

#### Feature: OpenAPI Spec + Interactive Docs

| Aspect           | Value |
|------------------|-------|
| **ID**           | FEAT-029 |
| **Status**       | Stable |
| **Endpoints**    | `GET /api/openapi` (spec JSON), `GET /api/docs` (Scalar UI) |
| **UI**           | `/api/docs` — Scalar interactive docs (`@scalar/api-reference-react`) |
| **Users**        | Developers, AI agents, SDK consumers |
| **Dependencies** | `lib/openapi/registry.ts`, `public/openapi.json`, `bun run openapi:gen` script |
| **Evidence**     | `app/api/openapi/route.ts`, `app/api/docs/page.tsx` |

**Capabilities:**
- [x] Static OpenAPI JSON served at `/api/openapi` (force-cached 300s)
- [x] Scalar interactive docs at `/api/docs`
- [x] Spec generated by `bun run openapi:gen` (script reads `route.openapi.ts` files alongside each route)
- [x] Each route has a `route.openapi.ts` sibling defining request/response schemas

---

#### Feature: Error Envelope + Request Tracing

| Aspect           | Value |
|------------------|-------|
| **ID**           | FEAT-030 |
| **Status**       | Stable |
| **Endpoints**    | Cross-cutting — all `/api/v1/*` routes |
| **UI**           | None — surfaced in API clients and browser devtools |
| **Users**        | All API consumers |
| **Dependencies** | `lib/api/handler.ts`, `lib/api/error-envelope.ts`, `lib/api/logging.ts` |
| **Evidence**     | `lib/api/handler.ts`, `lib/api/error-envelope.ts` |

**Capabilities:**
- [x] `x-request-id` header on ALL responses (inbound propagated or UUID minted)
- [x] Uniform error envelope: `{ error: { code, message, details?, request_id? } }`
- [x] 9 typed error codes mapped to HTTP status (400/401/403/404/409/422/429/500/502)
- [x] Zod validation failures → `validation_failed` (422) with `details` array
- [x] Structured JSON logging to stdout (Vercel-indexable): method, path, status, duration_ms, error_code

---

#### Feature: Idempotency (POST Replay Protection)

| Aspect           | Value |
|------------------|-------|
| **ID**           | FEAT-031 |
| **Status**       | Planned — infrastructure complete; zero routes wired |
| **Endpoints**    | Cross-cutting — intended for POST endpoints via `Idempotency-Key` header |
| **UI**           | None |
| **Users**        | API clients (CLI, CI, agents) |
| **Dependencies** | `idempotency_keys` table, `lib/api/idempotency.ts` |
| **Evidence**     | `lib/api/idempotency.ts` (fully implemented), `supabase/migrations/0009_cross_cutting.sql` |

**Capabilities:**
- [x] `beginIdempotentRequest()` / `recordIdempotencyResult()` / `discardIdempotencyResult()` implemented
- [x] `idempotency_keys` table with `(user_id, endpoint, key)` unique constraint + 24h TTL
- [x] Deterministic payload hashing (`stableStringify + sha256`)
- [x] Replay detection: same key + same hash + succeeded → returns cached response
- [x] Conflict: same key + different hash → 409
- [ ] ZERO routes currently call `beginIdempotentRequest()` — feature is dormant
- [ ] No TTL cleanup job for expired `idempotency_keys` rows

---

### Domain J — QA Onboarding & Developer Experience

#### Feature: Testability Guide (`/qa`)

| Aspect           | Value |
|------------------|-------|
| **ID**           | FEAT-032 |
| **Status**       | Stable |
| **Endpoints**    | None (public page) |
| **UI**           | `/qa` — `QaShell` with `AuthMethods`, `ArchDiagram`, `EnvSetup`, `TwoWayTabs` |
| **Users**        | QA engineers, AI agents, SDET onboarding |
| **Dependencies** | `app/qa/qa-config.ts`, `app/qa/_components/` |
| **Evidence**     | `app/qa/page.tsx`, `app/qa/qa-config.ts` |

**Capabilities:**
- [x] Public page — no auth required
- [x] Language: Spanish (`lang: 'es'` in `qaConfig`)
- [x] Auth method snippets (cookie, Bearer PAT, signup/signin, cookie-mint hybrid)
- [x] Playwright fixture snippets (scripted magic-link regression + cookie→PAT hybrid bridge)
- [x] MCP config blocks for Claude Code and OpenCode (DBHub, OpenAPI, Postman, Playwright)
- [x] PAT scope reference table (4 scopes + purpose)
- [x] API endpoint catalog
- [x] DBHub database roles reference (`qa_inspector_ro` / `qa_inspector_rw`)
- [x] Env variable slot list (no literals — only slot names)
- [x] Credentials source pointer to Jira Epic BK-29 (not inlined)
- [x] Design system tokens page at `/design-tokens` (dev reference)

---

#### Feature: Activity Log (Audit Trail)

| Aspect           | Value |
|------------------|-------|
| **ID**           | FEAT-033 |
| **Status**       | Secondary — write path exists (service_role writes); no read API |
| **Endpoints**    | None (DB only via `qa_inspector_ro`) |
| **UI**           | None |
| **Users**        | Engineering ops via DBHub `qa_inspector_ro` |
| **Dependencies** | `activity_log` table (append-only, RLS: workspace members can SELECT for their workspace) |
| **Evidence**     | `supabase/migrations/0009_cross_cutting.sql` |

**Capabilities:**
- [x] Append-only table (no UPDATE/DELETE client policies)
- [x] Written by service_role via logging middleware
- [x] Workspace members can SELECT their workspace's audit rows
- [ ] No REST endpoint for reading activity log
- [ ] `entity_type` and `action` values not enumerated in schema (no CHECK constraint)

---

#### Feature: Feature Flag Management

| Aspect           | Value |
|------------------|-------|
| **ID**           | FEAT-034 |
| **Status**       | Planned — schema ready; no code reads flags yet |
| **Endpoints**    | None |
| **UI**           | None |
| **Users**        | Engineering ops via Supabase Studio |
| **Dependencies** | `feature_flags` table (global and workspace-scoped) |
| **Evidence**     | `supabase/migrations/0009_cross_cutting.sql` |

**Capabilities:**
- [x] Table supports global and workspace-scoped flags with optional jsonb payload
- [x] Global flags: readable by any authenticated user
- [x] Workspace flags: readable by workspace members
- [ ] Zero flags currently read in any route or component
- [ ] No API or admin UI for flag management

---

#### Feature: User View State Persistence

| Aspect           | Value |
|------------------|-------|
| **ID**           | FEAT-035 |
| **Status**       | Planned — schema ready; no code writes or reads `user_view_state` |
| **Endpoints**    | None (intended: PostgREST direct) |
| **UI**           | None currently |
| **Users**        | Authenticated users |
| **Dependencies** | `user_view_state` table (PK: `user_id + project_id + view_kind`) |
| **Evidence**     | `supabase/migrations/0009_cross_cutting.sql` |

**Capabilities:**
- [x] Schema supports per-user, per-project, per-view-kind JSON state blob
- [ ] No code reads or writes this table
- [ ] `view_kind` valid values not documented (no CHECK constraint)

---

## 3. CRUD Matrix

| Entity | Create | Read | Update | Delete | Primary Access Path | Notes |
|--------|--------|------|--------|--------|---------------------|-------|
| `workspaces` | ✅ `POST /workspaces` | ✅ `GET /workspaces`, `GET /workspaces/{id}` | ✅ `PATCH /workspaces/{id}` (name only, owner only) | ⚠️ DB only (owner policy exists; no DELETE route) | REST API + `bunkai_bootstrap_workspace` RPC | `owner_user_id` immutable |
| `workspace_members` | ✅ Via invite accept (upsert) | ✅ `GET /workspaces/{id}/members` (UI page) | ⚠️ DB only (suspend/reinstate — no REST route) | ⚠️ DB only (cascade on workspace delete) | PostgREST direct from `MembersPage` RSC | No standalone PATCH member route |
| `workspace_invites` | ✅ `POST /workspaces/{id}/invites` | ✅ `GET /workspaces/{id}/invites` | ✅ `POST /invites/{id}` (rotate token + reset expiry) | ✅ `DELETE /workspaces/{id}/invites/{id}` (soft-delete via `revoked_at`) | REST API | `status` is derived — not a column |
| `workspace_invite_secrets` | ✅ Via invite issuance/rotation | ❌ No client read (service_role only) | ✅ Via token rotation | ❌ No client delete | service_role only | 0011 sibling table |
| `projects` | ❌ No REST route (schema + RLS ready; Phase E) | ✅ PostgREST via RSC | ⚠️ DB only | ⚠️ DB only (cascade) | PostgREST direct | "ships in Phase E" |
| `modules` | ❌ No REST route (schema + RLS ready) | ✅ PostgREST via RSC | ❌ No route | ⚠️ DB only | PostgREST direct | No UI for create/rename/reorder |
| `user_stories` | ❌ No REST route (schema + RLS ready; BK-15) | ✅ PostgREST via RSC + ATC editor | ❌ No route | ⚠️ DB only (RESTRICT if ATCs bound) | PostgREST direct | Jira link optional |
| `acceptance_criteria` | ❌ No REST route | ✅ PostgREST via ATC editor AC picker | ❌ No route | ⚠️ DB only (CASCADE to `atc_ac` bindings) | PostgREST direct | BK-15 planned |
| `atcs` | ❌ No CREATE route (editor opens existing ATC; no "new ATC" flow wired) | ✅ PostgREST via RSC + ATC editor page | ✅ `bunkai_save_atc` RPC (via Server Action) | ❌ No route | PostgREST + RPC | Status update only via ATC save |
| `atc_steps` | ✅ Via `bunkai_save_atc` (full-replace) | ✅ PostgREST via ATC editor | ✅ Via `bunkai_save_atc` | ✅ Via `bunkai_save_atc` (full-replace DELETE+INSERT) | RPC only | No standalone CRUD |
| `atc_assertions` | ✅ Via `bunkai_save_atc` | ✅ PostgREST via ATC editor | ✅ Via `bunkai_save_atc` | ✅ Via `bunkai_save_atc` | RPC only | No standalone CRUD |
| `atc_acceptance_criteria` | ✅ Via `bunkai_save_atc` | ✅ PostgREST via ATC editor | ✅ Via `bunkai_save_atc` (full-replace) | ✅ Via `bunkai_save_atc` | RPC only | Empty array allowed at DB level — app-layer guards only |
| `access_tokens` | ✅ `POST /tokens`, `POST /auth/signup`, `POST /auth/signin` | ✅ `GET /tokens` | ⚠️ Soft-delete only via `DELETE /tokens/{id}` | ❌ No DELETE policy intentional (audit trail) | REST API | `last_used_at` fire-and-forget update |
| `access_token_secrets` | ✅ Via `mintPat()` (service_role) | ❌ service_role only | ⚠️ Via token rotation (service_role) | ❌ No client delete | service_role only | 0011 sibling table |
| `activity_log` | ✅ service_role writes only | ⚠️ DB only (`qa_inspector_ro`) | ❌ Append-only (no UPDATE policies) | ❌ Append-only | DB only | No REST read endpoint |
| `feature_flags` | ❌ No client path (Studio/migrations) | ⚠️ DB only (no REST endpoint) | ❌ No client path | ❌ No client path | DB only | Phase 2 — nothing reads flags |
| `user_view_state` | ❌ No code path yet | ❌ No code path yet | ❌ No code path yet | ⚠️ Via CASCADE | Schema only | Phase 2 — nothing reads/writes |
| `idempotency_keys` | ❌ No routes wired yet | ⚠️ Owner can SELECT own keys | ⚠️ Status update by middleware | ❌ No explicit delete | Schema + lib only | TTL cleanup job missing |
| `magic_link_tokens` | ✅ Best-effort via auth magic-link route | ❌ service_role only | ❌ Read-only | ❌ No delete policy | service_role only | `consumed_at` not yet updated |
| `magic_link_token_secrets` | ✅ Via magic-link issuance | ❌ service_role only | ❌ No update | ❌ No delete | service_role only | 0011 sibling table |

**CRUD coverage**: 8 of 20 entities have full or near-full CRUD via REST/RPC. 6 entities are schema-only (no application-layer CRUD routes). 6 entities are partially accessible.
**Approximate REST coverage**: ~40% of entities have REST CRUD; ~60% require PostgREST direct access or DB-only operations.

---

## 4. API Endpoint Inventory

### Auth & Identity

| Method | Endpoint | Purpose | Auth |
|--------|----------|---------|------|
| POST | `/api/v1/auth/magic-link` | Trigger magic-link OTP email | None |
| GET | `/auth/callback` | OTP exchange → set session cookie → redirect | None (OTP exchange) |
| POST | `/api/v1/auth/signup` | Create QA user + mint PAT (headless bootstrap) | None |
| POST | `/api/v1/auth/signin` | Email+password sign in + mint PAT | None |
| GET | `/api/v1/me` | Identity + workspaces + active workspace | Cookie or Bearer |
| POST | `/api/v1/me/active-workspace` | Switch active workspace (sets `bk_active_ws` cookie) | Cookie only |

### Workspace Management

| Method | Endpoint | Purpose | Auth |
|--------|----------|---------|------|
| GET | `/api/v1/workspaces` | List caller's workspaces | Cookie or Bearer |
| POST | `/api/v1/workspaces` | Create workspace (calls `bunkai_bootstrap_workspace` RPC) | Cookie |
| GET | `/api/v1/workspaces/{id}` | Get single workspace | Cookie only |
| PATCH | `/api/v1/workspaces/{id}` | Update workspace name (owner only) | Cookie |

### Team / Invite Management

| Method | Endpoint | Purpose | Auth |
|--------|----------|---------|------|
| GET | `/api/v1/workspaces/{id}/invites` | List invites with derived status | Cookie (admin+) |
| POST | `/api/v1/workspaces/{id}/invites` | Issue invite — returns raw token + accept_url once | Cookie (admin+) |
| POST | `/api/v1/workspaces/{id}/invites/{inviteId}` | Rotate invite token (new secret + reset expiry) | Cookie (admin+) |
| DELETE | `/api/v1/workspaces/{id}/invites/{inviteId}` | Revoke invite (stamps `revoked_at`) | Cookie (admin+) |
| POST | `/api/v1/invites/accept` | Accept invite by token (email match required) | Cookie |

### PAT Management

| Method | Endpoint | Purpose | Auth |
|--------|----------|---------|------|
| GET | `/api/v1/tokens` | List caller's PATs (no secrets) | Cookie |
| POST | `/api/v1/tokens` | Mint PAT (session only — no PAT-to-PAT) | Cookie |
| DELETE | `/api/v1/tokens/{id}` | Revoke PAT (soft-delete) | Cookie |

### Platform / Infrastructure

| Method | Endpoint | Purpose | Auth |
|--------|----------|---------|------|
| GET | `/api/v1` | API discovery banner (version, openapi, docs) | None |
| OPTIONS | `/api/v1` | CORS preflight | None |
| GET | `/api/v1/health` | Liveness probe — returns env + timestamp | None |
| GET | `/api/openapi` | OpenAPI JSON spec (static, force-cached 300s) | None |
| GET | `/api/docs` | Interactive Scalar docs UI | None |

**Total REST endpoints: 22** (21 via routes + 1 non-v1 `/auth/callback`)

**Notable: NO REST endpoints exist for**: projects CRUD, modules CRUD, user stories CRUD, acceptance criteria CRUD, ATC creation/deletion, ATC search, test runs, activity log reads, feature flag reads/writes.

---

## 5. UI Component Inventory

### Forms

| Component | Page | Purpose | API Calls |
|-----------|------|---------|-----------|
| `MagicLinkForm` | `/login` | Magic-link email submission | `POST /api/v1/auth/magic-link` |
| `OnboardingForm` | `/onboarding` | Workspace name + slug creation with live slugify | `POST /api/v1/workspaces` |
| `MembersClient` (invite form) | `/workspaces/{id}/members` | Email + role input → generate invite link | `POST /workspaces/{id}/invites` |
| `AtcEditor` (Monaco fields) | `/projects/{slug}/atcs/{id}` | Title, layer, tags, steps (Markdown), assertions (YAML) | `bunkai_save_atc` via Server Action |
| `AnchoringPanel` | `/projects/{slug}/atcs/{id}` | Select user story + bound ACs from picker | None (state management only) |
| `AcceptClient` | `/invites/accept` | Token submission + redirect on success | `POST /api/v1/invites/accept` |

### Dashboards / Views

| Component | Page | Purpose | Data Source |
|-----------|------|---------|-------------|
| `AtcTable` | `/projects/{slug}` | All ATCs tabular view with module path, status, layer | PostgREST via RSC |
| `Sidebar` | `/projects/{slug}` | Module tree navigation (collapsible, hierarchical) | PostgREST via RSC + `buildModuleTree()` |
| `MembersClient` (member list) | `/workspaces/{id}/members` | Active members list (user_id, role, status) | PostgREST via RSC |
| `MembersClient` (invite list) | `/workspaces/{id}/members` | Pending/accepted/revoked/expired invites | PostgREST via RSC |
| `QaShell` | `/qa` | Testability guide with auth snippets, MCP configs, DB roles | Static (`qa-config.ts`) |
| Root page (`/`) | `/` | Splash/redirect — unauthenticated users redirected to `/login` | None |

### Action Buttons / Dialogs

| Component | Trigger | Action | API Call |
|-----------|---------|--------|----------|
| "Generate invite link" button | `MembersClient` form submit | Issue invite → copy URL to clipboard | `POST /workspaces/{id}/invites` |
| "Rotate link" button | Pending invite row | Rotate invite token → copy new URL | `POST /workspaces/{id}/invites/{id}` |
| "Revoke" button | Pending invite row | Stamp `revoked_at` on invite | `DELETE /workspaces/{id}/invites/{id}` |
| Save button | `AtcEditor` | Atomic ATC save via Server Action | `bunkai_save_atc` RPC |
| "New ATC" / "New Test" buttons | Project `Topbar` | No-op (stub — flow not wired) | None |
| `CommandPalette` (⌘K) | Topbar search button | Input renders but no search call wired | None |
| `WorkspaceSwitcher` | Topbar left | Switch active workspace | `POST /api/v1/me/active-workspace` |

---

## 6. Third-Party Integrations

| Service | Purpose | Package | Status | Features Using It |
|---------|---------|---------|--------|-------------------|
| Supabase Auth | Magic-link OTP, session management, `auth.users` lifecycle | `@supabase/supabase-js ^2.106.0`, `@supabase/ssr ^0.10.3` | Active | FEAT-001, FEAT-002, FEAT-003, FEAT-004 |
| Supabase Postgres 17 | Primary DB — all entities, RLS, RPCs, PostgREST | `@supabase/supabase-js` | Active | All features |
| Vercel | Serverless hosting, CDN, env management | Next.js build target | Active | All features |
| Monaco Editor | In-browser code editor for steps (Markdown) and assertions (YAML) | `@monaco-editor/react ^4.7.0` | Active | FEAT-024 |
| Scalar | Interactive OpenAPI docs UI | `@scalar/api-reference-react ^0.9.38` | Active | FEAT-029 |
| Sonner | Toast notifications | `sonner ^2.0.7` | Active | FEAT-010, FEAT-012, FEAT-013, FEAT-014, FEAT-024 |
| TanStack Table | Table rendering for `AtcTable` | `@tanstack/react-table ^8.21.3` | Active | FEAT-018 |
| Radix UI | Headless UI primitives (Dialog, DropdownMenu, Tabs, Tooltip) | `@radix-ui/react-*` | Active | Multiple UI components |
| cmdk | Command menu (CommandPalette) | `cmdk ^1.1.1` | Active | FEAT-025 (stub) |
| Lucide React | Icon library | `lucide-react ^1.16.0` | Active | All UI components |
| Zod | Runtime schema validation | `zod ^4.4.3` | Active | All API routes |
| `@asteasolutions/zod-to-openapi` | OpenAPI spec generation from Zod schemas | `^8.5.0` | Active | FEAT-029 |
| Husky + lint-staged | Pre-commit hooks (lint + format) | `husky ^9.1.7` | Active | Dev workflow |
| Jira Cloud | External issue tracker (reference only — no live API calls) | None (field storage only) | Reference only | FEAT-022 (`external_id`, `external_url`) |
| Resend | Transactional email (configured in env; not yet wired in app) | `RESEND_API_KEY` env var present | Planned | None currently wired |
| n8n | Workflow automation MCP | `N8N_API_URL`, `N8N_API_KEY` env vars | Planned / MCP only | None currently wired |
| Tavily | Web search MCP | `TAVILY_API_KEY` env var | MCP only | `/qa` guide reference |
| DBHub | DB inspection MCP for QA (`qa_inspector_ro/_rw` roles) | `@bytebase/dbhub@latest` | Active (MCP) | FEAT-032 (qa guide), test fixture setup |
| OpenAPI MCP | API exploration via `@ivotoby/openapi-mcp-server` | Configured in `/qa` guide | Active (MCP) | FEAT-032 |
| Playwright MCP | Browser automation for AI agents | `@playwright/mcp@latest` | Active (MCP) | FEAT-032 |
| Postman MCP | API collection management | `https://mcp.postman.com/mcp` | Active (MCP) | FEAT-032 |

---

## 7. Feature Flags and WIP

### Environment Variable Feature Gates

| Variable | Description | Default | Notes |
|----------|-------------|---------|-------|
| `SUPABASE_JWT_SECRET` | Optional in MVP; required for custom JWT claim signing | Not set | Phase 2 |
| `RESEND_API_KEY` | Transactional email — invite emails not yet implemented | Not set | MVP has no transactional email; invite link returned in API response |
| `N8N_API_URL` / `N8N_API_KEY` | n8n workflow automation MCP | Not set | MCP integration only; no app-level usage |
| `ATLASSIAN_*` | Jira Cloud integration (URL, email, API token) | Not set | QA scripting only; no live sync from app |

**No `FEATURE_`, `ENABLE_`, or `BETA_` prefixed env vars found in `.env.example` or `lib/env.ts`.**
Feature gating is managed via the `feature_flags` DB table (not env vars), and no flags are currently read in any route or component.

### Planned Features (Evidence from Code / Comments)

| Planned Feature | Evidence | Estimated Status |
|-----------------|----------|-----------------|
| Project Creation UI | `app/(app)/projects/page.tsx` comment: "Project creation UI ships in Phase E" | Phase E |
| OAuth (GitHub / Google) | `app/(auth)/login/page.tsx` disabled buttons: `title="OAuth ships next sprint"` | Next sprint |
| Test Run Execution History | `app/qa/qa-config.ts` `patScopes` entry: "Sprint 2"; `lib/types.ts` `AtcStatus` enum exists | Sprint 2 |
| ATC Search (REST) | `CommandPalette` stub wired to ⌘K; GIN index live; no backend route | Next sprint |
| ATC Creation Flow | "New ATC" / "New Test" buttons visible in `Topbar`; no creation modal or route | Next sprint |
| Feature Flag Enforcement | `feature_flags` table ready; ZERO reads in code | Phase 2 |
| Idempotency Wiring | `lib/api/idempotency.ts` fully implemented; ZERO routes call it | Next sprint |
| User View State (Column Prefs) | `user_view_state` table ready; ZERO reads/writes in code | Phase 2 (Wave 6) |
| Member Suspension UI | `workspace_members.status` CHECK supports `suspended`; no API route or UI action | Unplanned |
| Transactional Email | `RESEND_API_KEY` present in env; code comment in `MembersClient`: "MVP does not send email" | Next sprint |
| Idempotency Key TTL Cleanup | `expires_at` column + index on `idempotency_keys`; no cron job found | Unplanned |
| `magic_link_tokens.consumed_at` update | Column exists in schema; no code path sets it | Unplanned |

---

## 8. QA Relevance

### Feature Test Coverage Matrix

| Feature ID | Feature | Unit | Integration | E2E | Risk | Priority |
|------------|---------|------|-------------|-----|------|----------|
| FEAT-001 | Magic Link Auth | — | ⚠️ Auth callback unverified | ⚠️ Stub only (magic-link passwordless; intercepting OTP out of scope) | HIGH | P1 |
| FEAT-002 | Headless Signup | — | ✅ (curl-testable) | ✅ Via PAT bootstrap | HIGH | P1 |
| FEAT-003 | Headless Signin | — | ✅ (curl-testable) | ✅ Via PAT | HIGH | P1 |
| FEAT-004 | Identity (`/me`) | — | ✅ (cookie + bearer paths) | ✅ | HIGH | P1 |
| FEAT-005 | Active Workspace Pref | — | ⚠️ No test | ⚠️ No test | LOW | P3 |
| FEAT-007 | Workspace Creation | — | ✅ (409 slug collision) | ✅ Onboarding flow | HIGH | P1 |
| FEAT-008 | Workspace Retrieval/Update | — | ⚠️ PATCH owner-only | ❌ No E2E | MEDIUM | P2 |
| FEAT-010 | Invite Issuance | — | ✅ (role/validation) | ✅ Members page | HIGH | P1 |
| FEAT-011 | Invite List | — | ⚠️ | ⚠️ | MEDIUM | P2 |
| FEAT-012 | Token Rotation | — | ✅ | ⚠️ | MEDIUM | P2 |
| FEAT-013 | Invite Revocation | — | ✅ | ⚠️ | MEDIUM | P2 |
| FEAT-014 | Invite Acceptance | — | ✅ (email mismatch, expiry) | ✅ Accept page | HIGH | P1 |
| FEAT-015 | PAT Issuance | — | ✅ (scope validation) | ✅ | HIGH | P1 |
| FEAT-016 | PAT List | — | ✅ | ⚠️ | LOW | P3 |
| FEAT-017 | PAT Revocation | — | ✅ | ✅ | HIGH | P1 |
| FEAT-018 | Project Navigation / ATC Browse | — | ⚠️ RSC reads | ⚠️ | HIGH | P1 |
| FEAT-024 | ATC Authoring + Versioning | — | ✅ `bunkai_save_atc` | ⚠️ Monaco-heavy | HIGH | P1 |
| FEAT-027 | API Discovery | — | ✅ | — | LOW | P3 |
| FEAT-028 | Health Probe | — | ✅ | — | LOW | P3 |
| FEAT-029 | OpenAPI / Docs | — | ✅ | ✅ | LOW | P3 |
| FEAT-030 | Error Envelope / Tracing | — | ✅ (all routes) | ⚠️ | MEDIUM | P2 |
| FEAT-031 | Idempotency | — | ❌ Not wired | ❌ | N/A | Blocked |
| FEAT-032 | QA Onboarding (`/qa`) | — | — | ✅ Public page | MEDIUM | P2 |
| FEAT-033 | Activity Log | — | ⚠️ DB only | ❌ | LOW | P3 |

### High-Risk Features (Prioritize Testing)

| Feature | Risk Level | Risk Reason |
|---------|-----------|-------------|
| FEAT-001 Magic Link Auth | HIGH | Auth gate for all browser users; OTP callback flow unverified from code |
| FEAT-002 Headless Signup | HIGH | Security: creates users with `email_confirm: true`; PAT shown once |
| FEAT-003 Headless Signin | HIGH | Uniform 401 anti-enum; PAT lifetime management |
| FEAT-010 Invite Issuance | HIGH | Owner role cannot be granted via invite (CHECK enforced, not just app-layer); token shown once |
| FEAT-014 Invite Acceptance | HIGH | Email-match validation; multi-tenant boundary enforcement at invite redemption |
| FEAT-015 PAT Issuance | HIGH | Security: no PAT-to-PAT; 256-bit secret; scope constraints |
| FEAT-017 PAT Revocation | HIGH | Irreversible soft-delete; no recovery path |
| FEAT-024 ATC Authoring | HIGH | Anchoring moat: empty `ac_ids` NOT validated at DB level (RPC gap); application-only guard |
| FEAT-007 Workspace Creation | HIGH | Atomic RPC; slug uniqueness; reserved slug list; owner assignment |
| RLS Isolation (all entities) | CRITICAL | Multi-tenant RLS must prevent cross-workspace data leakage; `BYPASSRLS` QA roles correctly scoped |

### Key QA-Specific Findings

- **Anchoring moat gap** (FEAT-024): `bunkai_save_atc` RPC accepts `p_ac_ids uuid[]` and only checks `if p_ac_ids is not null`. An empty array `'{}'::uuid[]` is not null — it passes the check and creates an unanchored ATC. Application guard in `saveAtcAction` prevents this from the UI, but any direct RPC call (e.g., via `qa_inspector_rw` or a headless SDET script) can bypass it. **Must test via RPC directly.**

- **Idempotency dead code** (FEAT-031): `lib/api/idempotency.ts` is fully implemented and the `idempotency_keys` table exists, but ZERO routes call `beginIdempotentRequest()`. Clients sending `Idempotency-Key` headers receive no replay protection. **Do not test idempotency behavior in current sprint.**

- **NEXT_PUBLIC_SUPABASE_ANON_KEY vs SUPABASE_PUBLISHABLE_KEY mismatch**: `middleware.ts` reads `NEXT_PUBLIC_SUPABASE_ANON_KEY`; `.env.example` defines `SUPABASE_PUBLISHABLE_KEY`. Both must be set to the same value or middleware fails silently. **Critical environment setup defect.**

- **`GET /api/v1/workspaces/{id}` Bearer gap**: Route uses `createClient()` (cookie-only) — no Bearer support. Documented in `backend.md` discovery gaps but not surfaced in OpenAPI spec.

- **No TTL cleanup for `idempotency_keys`**: Rows accumulate indefinitely. No cron job configured.

---

## 9. Discovery Gaps

The following items could not be fully verified from source files, or represent confirmed uncertainty:

| Gap | Confidence | Severity | Notes |
|-----|-----------|----------|-------|
| ATC creation flow (new ATC from scratch) | Medium | HIGH | "New ATC" button visible in `Topbar` but no modal, route, or Server Action found. It is unknown whether ATCs are created via PostgREST direct INSERT or if there is an unreachable route. |
| `auth/callback` route implementation | High | MEDIUM | File confirmed at `app/auth/callback/route.ts` but content not read during this pass. OTP exchange logic inferred from `backend.md` + Supabase SSR pattern. |
| ATC slug auto-generation | Low | MEDIUM | `atcs.slug` is `UNIQUE per project` but no slug-generation code found in any route, migration, or lib file. How slugs are set on INSERT is unknown. |
| `bunkai_save_atc` empty `ac_ids` DB enforcement | High | HIGH | **Confirmed gap**: RPC only checks `if p_ac_ids is not null`. Empty array passes. See QA relevance above. |
| `qa_inspector_ro` / `qa_inspector_rw` CREATE ROLE DDL | Low | MEDIUM | REVOKE statements in migration 0011 reference these roles, but their `CREATE ROLE` DDL is absent from all migration files. Provisioned out-of-band via Supabase Studio. |
| Supabase project reference (`<project-ref>`) | Low | HIGH | Required to construct session cookie name `sb-<project-ref>-auth-token` for Playwright fixture configuration. Not found in `.env.example`. Must be obtained from team. |
| `feature_flags` key names | None | LOW | `feature_flags.key` is text with no CHECK — valid flag keys are undocumented. No flags are read anywhere in current code. |
| `user_view_state.view_kind` valid values | None | LOW | Column is unconstrained text. No examples found in migrations, routes, or components. |
| `activity_log.entity_type` and `action` valid values | None | LOW | No CHECK constraint. No examples in migration comments or route code. |
| Test run / execution history entities | None | HIGH | `run:execute` scope reserved; "Sprint 2" referenced in `qa-config.ts`. No `test_runs`, `test_run_items`, or equivalent table found in any migration (0001–0012). |
| `magic_link_tokens.consumed_at` update path | None | LOW | Column exists in schema but no code path sets it — magic link OTP exchange goes directly through Supabase Auth and does not call back to update the audit row. |
| Multi-workspace UI | None | MEDIUM | Current `/projects` redirects to first project; `WorkspaceSwitcher` component exists but no multi-workspace picker UI. Planned for Phase E. |
| Workspace deletion route | None | LOW | DB `workspaces_delete_owner` policy exists but no `DELETE /api/v1/workspaces/{id}` route found. |
| `GET /api/v1/workspaces/{id}` Bearer support | High | MEDIUM | Confirmed cookie-only — uses `createClient()` without Bearer fallback. Undocumented constraint vs OpenAPI spec intent. |
| ATC DELETE route | None | MEDIUM | No `DELETE /api/v1/atcs/{id}` or similar route found. ATCs can only be deleted via PostgREST direct or cascade. |

---

## Cross-Reference: Data Map → Feature Map

| Business Flow (business-data-map.md) | Mapped Feature(s) |
|--------------------------------------|-------------------|
| Flow 1: Workspace Creation + Member Invitation | FEAT-007, FEAT-010, FEAT-011, FEAT-012, FEAT-013, FEAT-014 |
| Flow 2: User Story Creation with Acceptance Criteria | FEAT-022, FEAT-023 (both Planned — PostgREST direct only) |
| Flow 3: ATC Creation + Versioning + AC Linking | FEAT-024 |
| Flow 4: Project Tree Loading | FEAT-018, FEAT-019 |
| Flow 5: PAT Generation + Authentication | FEAT-015, FEAT-016, FEAT-017 |
| Flow 6: Magic Link Authentication | FEAT-001 |
| Flow 7: Headless Signup | FEAT-002 |
| Flow 8: Identity Introspection (`/me`) | FEAT-004 |
| Flow 9: Plan Enforcement | FEAT-034 (Planned — no enforcement code active) |
| SM-1: ATC Execution Status | FEAT-026 (Planned — Sprint 2) |
| SM-2: WorkspaceMember Status | FEAT-009 (Planned — no suspend/reinstate route) |
| SM-3: PAT Lifecycle | FEAT-015, FEAT-017 |
| SM-4: WorkspaceInvite Lifecycle | FEAT-010, FEAT-011, FEAT-012, FEAT-013, FEAT-014 |
| SM-5: IdempotencyKey | FEAT-031 (Planned — infrastructure present, zero routes wired) |

**Orphaned entities (schema exists; no feature mapped to them via API or UI)**:
- `feature_flags` → FEAT-034 (Phase 2 only)
- `user_view_state` → FEAT-035 (Phase 2 only)
- `activity_log` (write) → service_role only; no REST read

**Entities without REST CRUD (PostgREST direct or DB-only)**:
- `projects` (Create, Update, Delete)
- `modules` (Create, Update, Delete)
- `user_stories` (all CRUD)
- `acceptance_criteria` (all CRUD)

---

*Sources: supabase/migrations/0001–0012, app/api/v1/**, app/(app)/**, app/(auth)/**, components/**, lib/**, package.json, .context/business/business-data-map.md, .context/SRS/functional-specs.md, .context/infrastructure/backend.md*
