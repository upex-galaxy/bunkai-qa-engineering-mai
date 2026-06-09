# User Personas — upex-bunkai-tms

> Generated: 2026-06-08 | Phase: Project Discovery Phase 2 — Architecture
> Sources: DESIGN.md §1–§2, app/(auth)/login/page.tsx, app/qa/qa-config.ts, migrations/0001_tenancy.sql, migrations/0008_access_tokens.sql

---

## Persona 1: Jordan — QA Engineer

- **Role in system**: `member` (typical), may also be `admin` in smaller teams
- **Primary goals**:
  - Author Acceptance Test Cases linked to user stories and acceptance criteria
  - Browse the module tree to navigate test coverage across a project
  - Track ATC status (unrun / pass / fail / blocked / skipped)
  - Search for existing ATCs by title or tags before creating duplicates
- **Key workflows**:
  - Logs in via magic link
  - Navigates to `projects/[projectSlug]` → browses ATC table
  - Opens `projects/[projectSlug]/atcs/[atcId]` → edits in Monaco editor
  - Binds ATC to one or more acceptance criteria using the AC picker
  - Adds ordered steps with `input_data` + `expected` fields
  - Adds discrete assertions
  - Saves — `bunkai_save_atc` RPC fires atomically (header + steps + assertions + AC bindings)
- **Pain points** (derived from product design rationale):
  - Generic tools don't enforce ATC-to-AC linkage → test debt accumulates
  - Spreadsheet-based TMS is slow at scale (hundreds of ATCs)
  - No full-text search in spreadsheets

Source: app/(app)/projects/[projectSlug]/atcs/[atcId]/page.tsx, DESIGN.md §1, migrations/0007_save_atc.sql

---

## Persona 2: Alex — SDET / Automation Engineer

- **Role in system**: `member` with PAT scopes `atc:read` + `atc:write` + `run:execute`
- **Primary goals**:
  - Mint long-lived PATs for CI pipelines
  - Query ATCs via REST API to drive automated test execution
  - Post run results back to the system (Sprint 2 feature — run model not yet implemented)
  - Bootstrap headless QA bot accounts without going through the browser
- **Key workflows**:
  - Uses `POST /api/v1/auth/signup` or `POST /api/v1/auth/signin` to get a session + PAT in one call
  - Uses `POST /api/v1/tokens` to mint additional scoped PATs
  - Queries `GET /api/v1/workspaces` + `GET /api/v1/me` to resolve workspace context
  - Reads `GET /api/openapi` to understand available endpoints
  - Consults `/qa` testability guide for Playwright fixture snippets and MCP config
- **Pain points**:
  - Cookie-only auth systems block headless CI consumers
  - No PAT scoping means over-privileged CI tokens

Source: app/qa/qa-config.ts (api + playwright sections), app/api/v1/auth/signup/route.ts, lib/api/pat.ts, migrations/0008_access_tokens.sql

---

## Persona 3: Sam — Engineering Manager / Workspace Admin

- **Role in system**: `admin` or `owner`
- **Primary goals**:
  - Invite team members to the workspace with appropriate roles
  - Monitor workspace-level activity via the activity log
  - Manage existing invites (list, revoke if needed)
  - Oversee test coverage across projects
- **Key workflows**:
  - Creates the workspace during onboarding (`bunkai_bootstrap_workspace` RPC)
  - Navigates to `workspaces/[id]/members` → views members + pending invites
  - Issues an invite via `POST /api/v1/workspaces/{id}/invites` → shares accept URL with teammate
  - Revokes stale invites via `DELETE /api/v1/workspaces/{id}/invites/{inviteId}`
  - Reads activity log for audit trail
- **Pain points**:
  - Lack of visibility into who last touched a test case
  - No role-based access means everyone has write access

Source: app/(app)/workspaces/[id]/members/page.tsx, app/api/v1/workspaces/[id]/invites/route.ts, migrations/0010_workspace_invites.sql, migrations/0009_cross_cutting.sql

---

## Persona 4: Claude/AI Agent — Agentic QA Consumer

- **Role in system**: Programmatic actor with `member`-level PAT scopes
- **Primary goals**:
  - Read ATC catalog to understand what test cases already exist before authoring
  - Write new ATCs or update existing ones as part of an agentic QA workflow
  - Execute ATCs (Sprint 2 — not yet available)
  - Bootstrap itself with credentials at session start
- **Key workflows**:
  - Uses `POST /api/v1/auth/signin` to obtain a session + PAT
  - Reads `GET /api/openapi` for the full API contract
  - Reads `/qa` testability guide for Playwright MCP config and fixture patterns
  - Uses DBHub MCP (`qa_inspector_ro` role) for direct DB introspection
  - Uses Playwright MCP for browser-based ATC authoring and verification
- **Pain points** (design rationale):
  - Headless consumers blocked by browser-only auth
  - No machine-readable testability guide

Source: app/qa/qa-config.ts (mcp block, playwright block), app/qa/page.tsx, DESIGN.md §1, lib/api/pat.ts

---

## Role Hierarchy

| Role | SELECT own workspace data | INSERT/UPDATE/DELETE project data | Manage workspace members | Invite teammates | Delete workspace |
|------|--------------------------|----------------------------------|--------------------------|-----------------|-----------------|
| `viewer` | Yes | No | No | No | No |
| `member` | Yes | Yes | No | No | No |
| `admin` | Yes | Yes | Yes (not owner) | Yes (up to admin) | No |
| `owner` | Yes | Yes | Yes | Yes (up to admin) | Yes |

Notes:
- `owner` role is assigned exactly once per workspace at bootstrap via `bunkai_bootstrap_workspace` RPC. It cannot be granted via invite (check constraint on `workspace_invites.role` excludes `owner`).
- All roles require `workspace_members.status = 'active'` — invited/suspended members have no data access.
- RLS helper functions (`bunkai_is_workspace_member`, `bunkai_can_write_workspace`, `bunkai_is_workspace_admin`, `bunkai_is_workspace_owner`) enforce these rules at the DB layer.

Source: migrations/0001_tenancy.sql, migrations/0005_rls_helpers.sql, migrations/0010_workspace_invites.sql

---

## Discovery Gaps

- [ ] Viewer-specific UI — the `viewer` role exists in the DB but no dedicated read-only view or navigation guard was found in the current page routes. It is unclear if viewers can access the ATC editor page or only the list view.
- [ ] Role change workflow — no API endpoint or UI was found for promoting/demoting an existing member's role. The `workspace_members.role` field can be updated via RLS-gated UPDATE, but no explicit route exists in `app/api/v1/`.
- [ ] Multi-workspace user experience — `GET /api/v1/me` returns all workspaces, but the project route shape is `projects/[projectSlug]` (no workspace slug prefix). Multi-workspace navigation is partially handled by `WorkspaceSwitcher` component but the full UX is Phase E.
