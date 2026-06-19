# BK-101 — Acceptance Test Plan (QA)

> Jira field: `customfield_10067` · [View in Jira](https://jira.upexgalaxy.com/browse/BK-101)

# Acceptance Test Plan — BK-101

# TMS-Workspace | View the workspaces I belong to

***Story******:*** BK-101
***Epic******:*** BK-85 — Account & Settings
***Sprint******:*** Current
***Date******:*** 2026-06-11
***Modality******:*** jira-native
***Risk Level******:*** HIGH (auth · RLS · multi-tenancy)
***Shift-Left Short-Circuit******:*** YES — label `shift-left-reviewed` 2026-06-10, within 30 days

---

## Scope Decision

> ***WARNING:**** ****UI tests are BLOCKED — BK-87 (Settings Hub) not shipped.*** Code inspection of `upex-bunkai-tms/app/` confirms no `/settings` route or workspace-list page exists. All UI outlines (P-01 to P-05, B-01 to B-03, I-03) are deferred until BK-87 merges and deploys.
`role`*** field BLOCKED — API does not return it.*** Both `GET /api/v1/me` and `GET /api/v1/workspaces` return `id, slug, name, owner*user*id, plan, created*at`. The `MemberRole` enum (`viewer / member / admin / owner`) is not exposed. Only `owner` can be inferred client-side via `owner*user_id === user.id`. Outlines requiring role assertion are deferred.
***Executable scope******:****** API + DB layer only.*** 9 outlines executable. 6 outlines blocked (documented below).

***Active workspace — RESOLVED******:*** `GET /api/v1/me` returns `active*workspace*id`. Mechanism: httpOnly cookie `bk*active*ws` (set by `POST /api/v1/me/active-workspace`); cookie session reads it directly; Bearer path falls back to `oldest workspace by created_at`.

---

## Phase 4 — Executable Test Outlines

### Coverage Summary

| Type | Count | Notes |
| --- | --- | --- |
| Positive | 2 | API happy path — workspace list + active workspace update |
| Negative | 4 | 401 unauth · 403 non-member · suspended excluded · invited excluded |
| Boundary | 1 | Active workspace fallback to oldest workspace |
| Integration | 2 | Cookie-session + Bearer token auth paths + DB cross-validation |
| ***Total executable**** | ****9*** | API surface only |
| Blocked | 6 | UI (BK-87) + role field (API gap) |

---

### P-01: Should return workspace list and active*workspace*id via cookie-session

- ***Type******:*** Positive · Integration
- ***Priority******:*** Critical
- ***Test level******:*** API + DB
- ***AC******:*** I-01 (revised)

***Preconditions******:***

- User `qa.bot.chiavassa@gmail.com` is authenticated (cookie session established)
- User has ≥ 1 active workspace membership in staging DB

***Test steps******:***

1. `GET https://staging-upexbunkai.vercel.app/api/v1/me` with session cookie
2. Verify: HTTP 200
3. Verify: response body contains `user.id`, `user.email == "qa.bot.chiavassa@gmail.com"`
4. Verify: `workspaces` array is non-empty
5. Verify: each workspace entry has `id`, `slug`, `name`, `owner*user*id`, `plan`, `created_at`
6. Verify: `active*workspace*id` is a valid UUID present in the `workspaces` array
7. DB cross-check: query `workspace*members` where `user*id = <user.id> AND status = 'active'` — count must match `workspaces.length`

***Expected result******:***

```json
{
  "user": { "id": "<uuid>", "email": "qa.bot.chiavassa@gmail.com" },
  "workspaces": [{ "id": "<uuid>", "slug": "<slug>", "name": "<name>", "owner*user*id": "<uuid>", "plan": "free", "created_at": "<iso>" }],
  "active*workspace*id": "<uuid>",
  "auth": { "source": "cookie", "scopes": [] }
}
```

***Test data******:***

```json
{ "email": "qa.bot.chiavassa@gmail.com", "env": "staging", "auth_method": "cookie-session" }
```

***Post-conditions******:*** None. Read-only operation.

---

### P-02: Should return workspace list with status=active filter via Bearer token

- ***Type******:*** Positive · Integration
- ***Priority******:*** Critical
- ***Test level******:*** API + DB
- ***AC******:*** I-02

***Preconditions******:***

- Valid PAT (`API_TOKEN` from `.env`) with staging credentials
- User has ≥ 1 active workspace membership

***Test steps******:***

1. `GET https://staging-upexbunkai.vercel.app/api/v1/me` with `Authorization: Bearer <API_TOKEN>`
2. Verify: HTTP 200
3. Verify: `auth.source == "bearer"`
4. Verify: `workspaces` array contains only workspaces where `workspace_members.status = 'active'`
5. DB cross-check: query `workspace*members` where `user*id = <user.id> AND status = 'active'` — IDs must match
6. Verify: `active*workspace*id` is present and is one of the returned workspace IDs (or oldest by `created_at` if no bound workspace)

***Expected result******:*** Same shape as P-01; `auth.source` must be `"bearer"`.

***Test data******:***

```json
{ "token": "<API*TOKEN from .env>", "env": "staging", "auth*method": "bearer" }
```

---

### P-03: Should set active*workspace*id when valid workspace is posted

- ***Type******:*** Positive
- ***Priority******:*** High
- ***Test level******:*** API
- ***AC******:*** Resolved gap from shift-left (active workspace data contract)

***Preconditions******:***

- User authenticated via cookie session
- User is a member of ≥ 2 workspaces

***Test steps******:***

1. `GET /api/v1/me` → record current `active*workspace*id` as `ws_before`
2. Select a workspace from `workspaces` array where `id ≠ ws*before` → call it `ws*target`
3. `POST /api/v1/me/active-workspace` with `{ "workspace*id": "<ws*target>" }` and cookie
4. Verify: HTTP 200
5. Verify: response body `{ "ok": true, "active*workspace*id": "<ws_target>" }`
6. Verify: response sets httpOnly cookie `bk*active*ws = <ws_target>`
7. `GET /api/v1/me` again with the updated cookie → verify `active*workspace*id == ws_target`

***Expected result******:*** Cookie rotated; `/api/v1/me` reflects the new active workspace.

***Test data******:***

```json
{ "precondition": "user must belong to ≥ 2 workspaces in staging" }
```

***Post-conditions******:*** Active workspace has changed — reset if subsequent tests assume a specific active workspace.

---

### N-01: Should return 401 when request has no authentication

- ***Type******:*** Negative
- ***Priority******:*** Critical
- ***Test level******:*** API
- ***AC******:*** N-03 (shift-left outline)

***Preconditions******:*** None (intentionally unauthenticated)

***Test steps******:***

1. `GET /api/v1/me` with no `Authorization` header and no session cookie
2. Verify: HTTP 401
3. Verify: response body contains error code (e.g. `{ "error": "unauthorized", ... }`)
4. Repeat for `GET /api/v1/workspaces` — also expected 401

***Expected result******:***

```json
{ "error": "unauthorized", "message": "..." }
```

---

### N-02: Should return 403 when posting a non-member workspace as active

- ***Type******:*** Negative · Security
- ***Priority******:*** Critical
- ***Test level******:*** API
- ***AC******:*** POST /api/v1/me/active-workspace authorization check

***Preconditions******:***

- User authenticated via cookie session
- A workspace exists in DB that the authenticated user is NOT a member of (`ws_foreign`)

***Test steps******:***

1. `POST /api/v1/me/active-workspace` with `{ "workspace*id": "<ws*foreign>" }` and cookie
2. Verify: HTTP 403
3. Verify: response body contains `{ "error": "forbidden", ... }`
4. Verify: `bk*active*ws` cookie is NOT updated (still holds previous value)
5. `GET /api/v1/me` → verify `active*workspace*id` unchanged

***Expected result******:***

```json
{ "error": "forbidden", "message": "You are not a member of that workspace." }
```

***Test data******:***

```json
{ "precondition": "requires a workspace ID from a different user in staging DB" }
```

---

### N-03: Should exclude suspended-membership workspace from Bearer token response

- ***Type******:*** Negative
- ***Priority******:*** High
- ***Test level******:*** API + DB
- ***AC******:*** N-01 (shift-left outline)

***Preconditions******:***

- A user exists with 1 active membership + 1 suspended membership in `workspace_members`
- PAT available for that user
- (If test user doesn't exist: DB seed via `workspace_members` UPDATE `status = 'suspended'`)

***Test steps******:***

1. `GET /api/v1/me` with Bearer token for the test user
2. Verify: HTTP 200
3. Verify: `workspaces` array does NOT contain the workspace where `membership.status = 'suspended'`
4. Verify: `workspaces` array DOES contain the workspace where `membership.status = 'active'`
5. DB check: confirm `workspace_members.status = 'suspended'` for the excluded workspace

***Expected result******:*** Only active-membership workspaces in response array.

***Test data******:***

```json
{ "precondition": "requires test user with suspended membership in staging DB" }
```

***Note******:*** If no test user with suspended membership exists, flag as BLOCKED-DATA and document the seed needed.

---

### N-04: Should exclude invited-membership workspace from Bearer token response

- ***Type******:*** Negative
- ***Priority******:*** Medium
- ***Test level******:*** API + DB
- ***AC******:*** N-02 (shift-left outline)

***Preconditions******:***

- A user exists with 1 active membership + 1 invited (status = 'invited') membership
- PAT available for that user

***Test steps******:***

1. `GET /api/v1/me` with Bearer token for the test user
2. Verify: HTTP 200
3. Verify: `workspaces` array does NOT contain the workspace where `membership.status = 'invited'`
4. Verify: `workspaces` array DOES contain the workspace where `membership.status = 'active'`

***Expected result******:*** Only active-membership workspaces in response.

***Note******:*** If no test user with invited membership exists, flag as BLOCKED-DATA.

---

### B-01: Should fallback active*workspace*id to oldest workspace when Bearer PAT has no bound workspace

- ***Type******:*** Boundary
- ***Priority******:*** Medium
- ***Test level******:*** API
- ***AC******:*** Active workspace fallback behavior

***Preconditions******:***

- A Bearer PAT that was created without a workspace-scope binding (or the bound workspace was deleted)
- User has ≥ 2 workspaces to make the fallback deterministic

***Test steps******:***

1. `GET /api/v1/me` with unbound Bearer PAT
2. Verify: HTTP 200
3. Verify: `active*workspace*id` equals the `id` of the workspace with the earliest `created_at` in the returned `workspaces` array

***Expected result******:*** `active*workspace*id` matches `workspaces.sort((a,b) => a.created*at - b.created*at)[0].id`.

---

### I-01: Should reflect DB active status in API response (cookie path)

- ***Type******:*** Integration · DB
- ***Priority******:*** High
- ***Test level******:*** API + DB
- ***AC******:*** I-01 + DB cross-validation

***Preconditions******:***

- User authenticated via cookie session
- DBHub MCP connected to staging DB

***Test steps******:***

1. `GET /api/v1/me` with cookie → record returned `workspaces` IDs
2. Query DB: `SELECT workspace*id FROM workspace*members WHERE user*id = '<user*id>' AND status = 'active'`
3. Verify: API `workspaces[].id` set ≡ DB `workspace_id` set (exact match, no extra, no missing)
4. Verify: `active*workspace*id` is contained in both sets

***Expected result******:*** Perfect set equality between API response and DB query.

---

## Blocked Outlines

> ***ERROR:*** The following outlines from the shift-left ATP CANNOT be executed until blockers are resolved.

| Outline | Blocker | Unblocked by |
| --- | --- | --- |
| P-01 Workspace list renders (UI) | BK-87 Settings Hub not shipped; no `/settings` route in `app/` | BK-87 merge + deploy |
| P-02 Role labels render (UI) | BK-87 not shipped + API missing `role` field | BK-87 + API extension |
| P-03 Active workspace visually marked (UI) | BK-87 not shipped (API mechanism exists: `active*workspace*id`) | BK-87 deploy |
| P-04 Single workspace renders cleanly (UI) | BK-87 not shipped | BK-87 deploy |
| P-05 Owner role label (UI + API) | BK-87 not shipped + API missing explicit `role` field (only `owner*user*id` available) | BK-87 + API extension |
| B-01 Zero workspaces empty state (UI) | BK-87 not shipped | BK-87 deploy |
| B-02 Maximum workspaces (UI) | BK-87 not shipped | BK-87 deploy |
| B-03 Loading state (UI) | BK-87 not shipped | BK-87 deploy |
| I-03 Navigation from Settings Hub | BK-87 not shipped | BK-87 deploy |

---

## Phase 5 — Test Data Strategy

| Data type | Outline | Source | Strategy |
| --- | --- | --- | --- |
| qa.bot.chiavassa@gmail.com session | P-01, P-03, N-01, N-02, I-01 | `.env` STAGING*USER*EMAIL/PASSWORD | Discover — exists in staging |
| Bearer PAT | P-02, B-01 | `.env` API_TOKEN | Discover — pre-generated |
| User with suspended membership | N-03 | staging `workspace_members` | Probe DB first; Generate (UPDATE status) if absent |
| User with invited membership | N-04 | staging `workspace_members` | Probe DB first; Generate (insert row with status='invited') if absent |
| Foreign workspace (non-member) | N-02 | staging `workspaces` | Discover — any workspace the QA bot is not a member of |
| Unbound Bearer PAT | B-01 | `POST /api/v1/auth/signup` | Generate — create fresh PAT without workspace scope |

***Cleanup******:*** N-03, N-04 may require DB restoration after test. Document row IDs before modifying.

---

## Open Items for Dev

1. `role`*** field missing from API*** — `GET /api/v1/me` and `GET /api/v1/workspaces` do not return the user's role per workspace. AC 1 requires role labels ("Rol: Admin"). Recommend extending the query to include a `workspace_members` join and returning `role: MemberRole`.
2. ***BK-87 dependency*** — all UI tests blocked until the Settings Hub route ships. Once `app/(app)/settings/workspaces/page.tsx` (or equivalent) is deployed, resume the blocked outlines above.

---
_Synced from Jira by sync-jira-issues_
