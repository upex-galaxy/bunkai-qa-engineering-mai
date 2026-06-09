# User Journeys — upex-bunkai-tms

> Generated: 2026-06-08 | Phase: Project Discovery Phase 2 — Architecture
> Sources: app/(auth)/*, app/(app)/*, app/api/v1/*, middleware.ts, app/invites/accept/page.tsx

---

## Journey 1: First-Time Signup + Workspace Creation (Onboarding)

**Actor**: New QA Engineer (any role — will become `owner`)
**Entry point**: `/login`
**Auth requirement**: None (public)

**Steps**:
1. User visits `/login` — sees magic-link form + disabled OAuth buttons.
2. Enters work email → client calls `POST /api/v1/auth/magic-link` with `{ email, next: "/onboarding" }`.
3. Supabase sends OTP email; user clicks the link → `/auth/callback?next=/onboarding` exchanges token → sets session cookie.
4. Middleware at `/onboarding` detects no active workspace membership.
5. User enters workspace `slug` + `name` in `OnboardingForm`.
6. Client calls `bunkai_bootstrap_workspace(slug, name)` RPC (or `POST /api/v1/workspaces`).
7. RPC atomically inserts into `workspaces` and `workspace_members` (role=`owner`, status=`active`).
8. Redirect to `/projects`.

**Exit**: `workspaces` row created, `workspace_members` row with `role=owner` + `status=active` created. User lands on `/projects`.

Source: app/(app)/onboarding/page.tsx, migrations/0006_bootstrap_workspace.sql, app/api/v1/workspaces/route.ts

---

## Journey 2: Headless QA Bot Provisioning (CI / AI Agent)

**Actor**: SDET or AI Agent (no browser)
**Entry point**: `POST /api/v1/auth/signup`
**Auth requirement**: None (public endpoint)

**Steps**:
1. Agent calls `POST /api/v1/auth/signup` with `{ email, password, pat_scopes: ["atc:read","atc:write"] }`.
2. Admin client creates user with `email_confirm: true` (bypasses confirmation email).
3. Handler signs the new user in immediately — Supabase session populated.
4. Handler calls `mintPat(...)` → inserts into `access_tokens` + `access_token_secrets`.
5. Response returns `{ user, session, pat: { token, scopes } }` — PAT is shown exactly once.
6. Agent stores `bk_pat_<prefix>.<secret>` and uses it for all subsequent requests via `Authorization: Bearer`.

**Exit**: `auth.users` row created, `access_tokens` row created with requested scopes. Agent has a long-lived Bearer token.

Source: app/api/v1/auth/signup/route.ts, lib/api/pat.ts, migrations/0008_access_tokens.sql

---

## Journey 3: Team Member Invitation + Acceptance

**Actor (sender)**: Workspace Admin or Owner
**Actor (receiver)**: Invitee (any email, may or may not have an account)
**Entry point (sender)**: `POST /api/v1/workspaces/{id}/invites`
**Entry point (receiver)**: `/invites/accept?token=...`

**Steps (sender)**:
1. Admin navigates to `workspaces/[id]/members` — sees current member + invite lists.
2. Admin calls `POST /api/v1/workspaces/{id}/invites` with `{ email, role: "member" }`.
3. Handler: generates raw token → SHA-256 hash → inserts `workspace_invites` row + `workspace_invite_secrets` row.
4. Handler returns `{ invite, token, accept_url, warning: "Copy this URL now..." }`.
5. Admin copies the `accept_url` and shares it with the invitee (MVP: no email — logged to stdout).

**Steps (receiver)**:
1. Invitee receives the `accept_url`, visits `/invites/accept?token=<raw>`.
2. If not logged in, must log in first (magic-link or password).
3. `AcceptClient` calls `POST /api/v1/invites/accept` with `{ token }`.
4. Handler: SHA-256 hash → lookup in `workspace_invite_secrets` → load invite metadata.
5. Validates: not revoked, not already accepted, not expired, email matches caller's auth email.
6. Upserts `workspace_members` (role from invite, status=`active`), stamps `accepted_at` on invite.
7. Redirect to `/projects`.

**Exit**: `workspace_members` row with the assigned role and `status=active`. `workspace_invites.accepted_at` is stamped.

Source: app/api/v1/workspaces/[id]/invites/route.ts, app/api/v1/invites/accept/route.ts, app/invites/accept/page.tsx

---

## Journey 4: ATC Authoring (Core Workflow)

**Actor**: QA Engineer (member or above)
**Entry point**: `/projects/[projectSlug]/atcs/[atcId]`
**Auth requirement**: Active session (cookie) — middleware guards `/projects/*`

**Steps**:
1. QA navigates to `/projects/[projectSlug]` → sees `AtcTable` listing all ATCs with module path + status.
2. Clicks "New ATC" (`N` keyboard shortcut) → navigates to a new ATC editor.
3. In `AtcEditorPage`: server loads project, ATC, steps, assertions, bound ACs, module data, all stories+ACs in project.
4. QA fills: title, layer (UI / API / Unit), tags, user story binding.
5. QA selects one or more Acceptance Criteria from the AC picker (enforces anchoring moat).
6. QA adds ordered steps (`content`, optional `input_data`, optional `expected`).
7. QA adds ordered assertions (`content`).
8. QA saves → `saveAtcAction` calls `bunkai_save_atc` RPC.
9. RPC atomically: updates ATC header, increments `version`, deletes+reinserts steps, deletes+reinserts assertions, deletes+reinserts AC bindings. Triggers fire: `bunkai_set_updated_at()` + `bunkai_atcs_refresh_tsv()`.

**Exit**: `atcs` row updated (version incremented), `atc_steps`, `atc_assertions`, `atc_acceptance_criteria` rows replaced atomically. Full-text `tsv` column refreshed.

Source: app/(app)/projects/[projectSlug]/atcs/[atcId]/page.tsx, migrations/0007_save_atc.sql, migrations/0004_atcs.sql

---

## Journey 5: API Consumer — ATC Read (CI/Agent Headless)

**Actor**: SDET or AI Agent using Bearer PAT
**Entry point**: `GET /api/v1/me` (identity + workspace context)
**Auth requirement**: `Authorization: Bearer bk_pat_*`

**Steps**:
1. Agent sends `GET /api/v1/me` with Bearer header.
2. `requireAuth` detects `Bearer` prefix → calls `requireBearerToken`.
3. Bearer middleware: parses prefix from token, queries `access_tokens` by `token_prefix`, constant-time SHA-256 compare against `access_token_secrets.hash`. Checks `revoked_at` and `expires_at`.
4. Returns `{ user, workspaces, active_workspace_id, auth: { source: "bearer", scopes } }`.
5. Agent uses `workspace_id` to scope subsequent queries.
6. Agent reads `GET /api/openapi` for full endpoint catalog.
7. Agent queries the DB directly via DBHub MCP using `qa_inspector_ro` role if deeper data access needed.

**Exit**: Agent has workspace context and can drive all API operations within its token scopes.

Source: lib/api/auth.ts, lib/api/middleware/bearer.ts, app/api/v1/me/route.ts, app/qa/qa-config.ts

---

## Route Map

| Route | Feature | Auth Required | Actor |
|-------|---------|--------------|-------|
| `/login` | Magic-link form | None | Anonymous |
| `/auth/callback` | OTP exchange → session cookie | None | Supabase redirect |
| `/onboarding` | Workspace creation form | Session cookie | New user (no workspace yet) |
| `/projects` | Projects index → auto-redirect to first project | Session cookie | Any active member |
| `/projects/[projectSlug]` | ATC table + module tree sidebar | Session cookie | Any active member |
| `/projects/[projectSlug]/atcs/[atcId]` | ATC editor (Monaco) | Session cookie | member+ |
| `/workspaces/[id]/members` | Member + invite management | Session cookie (admin data: admin/owner) | admin+ |
| `/invites/accept` | Invite redemption landing | None (token in query) | Invitee |
| `/qa` | Testability guide (Playwright fixtures, MCP config, PAT bootstrap) | None | QA/Agent |
| `/api/docs` | Scalar interactive API documentation | None | Any |
| `/api/openapi` | OpenAPI JSON spec | None | Any |
| `/api/v1/health` | Liveness probe | None | Any |
| `/api/v1/me` | Identity + workspaces + active workspace | Cookie or Bearer | Authenticated |
| `/api/v1/workspaces` | List/create workspaces | Cookie or Bearer | Authenticated |
| `/api/v1/workspaces/[id]` | Get/patch workspace | Cookie or Bearer | member+ (GET), owner (PATCH) |
| `/api/v1/workspaces/[id]/invites` | List/create invites | Cookie or Bearer | admin+ |
| `/api/v1/workspaces/[id]/invites/[inviteId]` | Revoke single invite | Cookie or Bearer | admin+ |
| `/api/v1/invites/accept` | Accept invite by raw token | Cookie (must be signed in) | Invitee |
| `/api/v1/tokens` | Mint/list PATs | Cookie (mint); Cookie or Bearer (list) | Authenticated |
| `/api/v1/tokens/[id]` | Revoke PAT | Cookie or Bearer | Token owner |
| `/api/v1/auth/magic-link` | Trigger OTP email | None | Anonymous |
| `/api/v1/auth/signup` | Headless user + PAT provisioning | None | CLI/agent |
| `/api/v1/auth/signin` | Headless sign-in + fresh PAT | None | CLI/agent |
| `/api/v1/me/active-workspace` | Set active workspace cookie | Cookie or Bearer | Authenticated |
| `/design-tokens` | Internal design system reference page | None | Developer |

Source: middleware.ts, app/(app)/*, app/(auth)/*, app/api/v1/*, app/qa/page.tsx

---

## Discovery Gaps

- [ ] Project creation journey — `/projects/page.tsx` shows an "empty-state placeholder" with the message "Project creation UI ships in Phase E." No UI or API endpoint for project creation was found in current routes. The `bunkai_bootstrap_workspace` RPC creates workspaces only. Project creation likely requires a future `POST /api/v1/workspaces/{id}/projects` route.
- [ ] Multi-workspace switch journey — `WorkspaceSwitcher` component exists in the project page but the full switcher UI and multi-workspace routing (`/projects/{workspaceSlug}/{projectSlug}`) is noted as Phase E.
- [ ] Auth callback route — `/auth/callback` directory was not found in the explored routes; the OTP exchange handler must exist but was not in the `find` results. Likely at `app/(auth)/callback/route.ts` or `app/auth/callback/route.ts`.
- [ ] ATC deletion journey — no "delete ATC" route or server action was found in current page files. The DB schema has `ON DELETE RESTRICT` on `atcs.user_story_id`, which would block story deletion if ATCs exist.
- [ ] Acceptance criteria management journey (FR-008 / BK-15) — the `acceptance_criteria` table has full CRUD RLS policies, and the ATC editor uses ACs as a picker, but no standalone AC management UI page was found at `projects/[projectSlug]/stories/[storyId]/criteria`. The feature may be headless (API-only) or embedded in the story editor which is not yet built.
