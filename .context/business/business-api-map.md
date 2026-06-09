# Business API Map — upex-bunkai-tms

> Last verified against code on 2026-06-08
> Generated: 2026-06-08 | Mode: CREATE | Sources: app/api/v1/**, lib/api/**, middleware.ts, app/auth/callback/route.ts, business-data-map.md, backend.md, architecture.md

---

## 1. Executive Summary

Bunkai TMS exposes a REST API that exists for one core reason: to let QA teams and AI agents author, organize, and execute acceptance test cases without losing the structural link between a test and the business requirement it proves. Every meaningful operation — creating a test case, binding it to acceptance criteria, running it, filing a bug — flows through this API. The system is designed so the browser UI and headless CI pipelines consume exactly the same endpoints; there is no "public API" versus "internal API" distinction.

From a business perspective, the API delivers value across three consumer profiles. Browser-authenticated QA engineers use cookie sessions to manage workspaces, invite teammates, and author ATCs through the Monaco editor. SDET pipelines and AI agents authenticate with Personal Access Tokens (PATs) to drive the same ATC and test-run lifecycle programmatically without human interaction. Engineering managers use the cookie session for workspace administration — member management, plan visibility, and invite lifecycle. All three consumers share the same endpoint surface; auth tier and Postgres RLS together determine what each caller can see or change.

The API is deliberately minimal at this stage: 14 REST endpoints under `/api/v1/` plus health, discovery, and docs routes. ATC creation and all module/story/AC management currently bypass the REST layer and go through Supabase PostgREST directly — this means RLS is the only guard for those operations, with no application-layer validation. Plan enforcement (community/cloud/enterprise feature gating) exists in the schema but is not yet wired in any route code. These are the two most significant architectural gaps with QA implications.

---

## 2. Permission & Auth Model

### Tier Table

| Tier | Who it applies to | How to acquire | Where enforced (code path) |
|------|-------------------|---------------|---------------------------|
| **Public** | Anyone (no session) | No action required | `middleware.ts` (path is not in PROTECTED_PREFIXES) |
| **Session (Cookie)** | Browser users after Magic Link OTP | POST `/api/v1/auth/magic-link` → click email link → GET `/auth/callback` | `middleware.ts` → `supabase.auth.getUser()` on every request; `lib/api/auth.ts` `requireAuth()` cookie branch |
| **Bearer PAT** | CLI, CI pipelines, AI agents | POST `/api/v1/auth/signup` or `/auth/signin` (one round trip) OR POST `/api/v1/tokens` (session required) | `lib/api/auth.ts` `requireAuth()` → `lib/api/middleware/bearer.ts` `requireBearerToken()` |
| **Role-gated (admin/owner)** | Workspace admins and owners only | Active workspace_members row with role `admin` or `owner` | Postgres RLS policies (`bunkai_is_workspace_admin`, `bunkai_is_workspace_owner`) + app-layer 403 mapping |
| **Service role (internal)** | API server only — never external callers | `SUPABASE_SERVICE_ROLE_KEY` env var | `lib/supabase/admin.ts` `createAdminClient()` — bypasses RLS; used only after application-layer auth already verified |

**Scope enforcement**: Bearer callers must carry the right scope for write operations. `requireScopeOrCookie()` in `lib/api/auth.ts` passes cookie callers through (UI already gates) but enforces scope for Bearer callers (`atc:write`, `run:execute`, `workspace:admin`). Cookie sessions are unscoped — `scopes: []` — RLS is the exclusive constraint.

---

### Auth Scheme 1 — Magic Link (Cookie Session)

```
  User (browser)                  Bunkai API              Supabase Auth         Supabase DB
       |                              |                        |                      |
       |-- POST /api/v1/auth/         |                        |                      |
       |   magic-link                 |                        |                      |
       |   { email, next? }           |                        |                      |
       |                              |-- signInWithOtp() ---->|                      |
       |                              |   emailRedirectTo:     |                      |
       |                              |   /auth/callback?next= |                      |
       |                              |                        |-- OTP email sent ---> [user inbox]
       |                              |-- (best-effort) -------|--------------------> INSERT magic_link_tokens
       |<-- { ok: true } (200) -------|                        |                      INSERT magic_link_token_secrets
       |                              |                        |
       |-- (user clicks email link) --|                        |
       |                              |                        |
       |-- GET /auth/callback         |                        |
       |   ?code=<otp>&next=/projects |                        |
       |                              |-- exchangeCodeFor      |
       |                              |   Session(code) ------>|
       |                              |<-- session ------------|
       |                              |                        |
       |<-- 302 redirect to /projects |                        |
       |    Set-Cookie:               |                        |
       |    sb-<ref>-auth-token       |                        |
       |                              |                        |
  [Subsequent browser requests]       |                        |
       |                              |                        |
       |-- GET /projects (cookie) --->| middleware.ts          |
       |                              | createServerClient()   |
       |                              | getUser() ------------>|
       |                              |<-- { user } ----------|
       |<-- 200 (page rendered) ------|                        |
```

Failure modes: OTP errors → 502 `upstream_error` or 429 `rate_limited`. Bad `code` at callback → redirect to `/login?error=otp_exchange_failed`. Missing `code` → `/login?error=missing_code`. Audit write failure → swallowed (never blocks auth).

---

### Auth Scheme 2 — Bearer PAT (Headless)

```
  CLI / Agent                     Bunkai API                  Supabase DB (admin client)
       |                              |                              |
  [One-time provisioning]            |                              |
       |                              |                              |
       |-- POST /api/v1/auth/signup   |                              |
       |   { email, password,         |                              |
       |     pat_scopes, pat_name }   |                              |
       |                              |-- admin.auth.createUser() -->|
       |                              |-- signInWithPassword() ----->|
       |                              |-- mintPat() ---------------->| INSERT access_tokens
       |                              |                              | INSERT access_token_secrets (hash only)
       |<-- 201 { user, session,      |                              |
       |          pat: { token:       |                              |
       |           "bk_pat_<p>.<r>" } |                              |
       |          warning }           |                              |
       |                              |                              |
  [Store token. Session disposable.] |                              |
       |                              |                              |
  [Every subsequent API request]      |                              |
       |                              |                              |
       |-- ANY /api/v1/* ------------>|                              |
       |   Authorization:             | requireAuth()                |
       |   Bearer bk_pat_<p>.<r>      | detects Bearer header        |
       |                              |                              |
       |                              | requireBearerToken():        |
       |                              | 1. strip "bk_pat_" prefix    |
       |                              | 2. split at "." -> p + r     |
       |                              | 3. fullSecret = p + r        |
       |                              | 4. SELECT access_tokens      |
       |                              |    WHERE token_prefix = p -->|
       |                              |<-- candidates (O(1) index) --|
       |                              | 5. SELECT access_token_      |
       |                              |    secrets WHERE id IN c. -->|
       |                              |<-- hashes ------------------|
       |                              | 6. SHA-256(fullSecret)       |
       |                              |    == stored hash?           |
       |                              | 7. revoked_at IS NULL?       |
       |                              | 8. expires_at ok?            |
       |                              | (all failures -> 401)        |
       |                              |-- fire-and-forget UPDATE     |
       |                              |   last_used_at ------------->|
       |<-- response -----------------|                              |
```

**KNOWN BUG (fixed in migration 0012)**: The original bearer implementation hashed only `remainder` (the part after the dot), not `prefix + remainder`. The minting code always hashed the full secret, so the comparison was structurally impossible to pass — every Bearer call returned a silent 401. This was fixed in `lib/api/middleware/bearer.ts` by reconstructing `fullSecret = prefix + remainder` before comparing. **Regression test required**: any fixture that exercises Bearer auth must verify that `bk_pat_<prefix>.<remainder>` authenticates successfully end-to-end.

---

## 3. Critical Business Journeys

### Journey 1 — PAT-Authenticated ATC Creation + AC Linking

**Business purpose**: An AI agent or SDET pipeline creates an acceptance test case and binds it to at least one acceptance criterion — the core quality contract that prevents orphan test cases.

**Endpoints involved**: POST `/api/v1/auth/signup` (or `/auth/signin`) → `bunkai_save_atc` RPC (via PostgREST, not a REST endpoint)

**Entities touched**: `access_tokens`, `access_token_secrets`, `atcs`, `atc_steps`, `atc_assertions`, `atc_acceptance_criteria`

```
  AI Agent / CLI                  Bunkai API                    Supabase DB
       |                              |                              |
  1. [Provision credentials]          |                              |
       |-- POST /api/v1/auth/signup   |                              |
       |   { email, pw, pat_scopes }  |                              |
       |<-- { pat: { token } } -------|-- INSERT access_tokens ----->|
       |                              |   INSERT access_token_secrets|
       |                              |                              |
  2. [Discover workspace + project]   |                              |
       |-- GET /api/v1/me            |                              |
       |   Bearer: bk_pat_...        |-- SELECT workspace_members -->|
       |<-- { workspaces, active_ws } |                              |
       |                              |                              |
  3. [Create ATC via PostgREST]       |                              |
       |-- POST <supabase-url>/rest/  |                              |
       |   v1/rpc/bunkai_save_atc     |                              |
       |   Bearer: <supabase-anon>    |                              |
       |   Cookie: sb-*-auth-token    |                              |
       |   { p_atc_id, p_user_story_id|                              |
       |     p_title, p_layer,        |                              |
       |     p_steps, p_assertions,   |                              |
       |     p_ac_ids: [uuid, ...] }  |-- bunkai_save_atc RPC ------>|
       |                              |   (SECURITY INVOKER)         |
       |                              |                              | UPDATE atcs
       |                              |                              | DELETE + INSERT atc_steps
       |                              |                              | DELETE + INSERT atc_assertions
       |                              |                              | DELETE + INSERT atc_acceptance_criteria
       |                              |                              | TRIGGER: refresh tsv (GIN)
       |                              |                              | TRIGGER: set updated_at
       |<-- void (client re-reads) ---|                              |
```

**Narrative**:
1. Agent provisions itself via `/auth/signup` — one call yields a PAT valid for all subsequent operations (no email verification required for headless users).
2. Agent calls `GET /api/v1/me` with the Bearer PAT to discover which workspace it belongs to and what its active workspace is.
3. ATC creation bypasses the REST layer — it calls the Supabase `bunkai_save_atc` RPC directly via PostgREST. This is intentional (the RPC is a single atomic transaction), but means auth is Supabase-native (anon key + session cookie), NOT the PAT Bearer. An agent running fully headless must first obtain a Supabase session via `/auth/signup` or `/auth/signin` and use the returned `session.access_token` as the PostgREST auth token.
4. `p_ac_ids` must be non-empty at the application layer — the RPC accepts an empty array but ATC-without-AC is a business violation. This is the anchoring moat.

**Discovery note**: There is no `POST /api/v1/atcs` REST endpoint. ATC lifecycle is 100% PostgREST + RPC. See §7 Discovery Gaps.

---

### Journey 2 — Magic Link Auth → Session → Workspace Access

**Business purpose**: A QA engineer authenticates via passwordless email link and reaches their test project tree.

**Endpoints involved**: POST `/api/v1/auth/magic-link` → GET `/auth/callback` → GET `/api/v1/me` → `GET /projects` (UI, RLS-backed)

**Entities touched**: `magic_link_tokens`, `magic_link_token_secrets` (audit), `workspace_members`, `workspaces`, `projects`, `modules`, `user_stories`, `atcs`

```
  Browser                         Bunkai API              Supabase Auth        Supabase DB
     |                                |                        |                    |
  1. |-- POST /api/v1/auth/magic-link |                        |                    |
     |   { email }                    |-- signInWithOtp() ---->|                    |
     |                                |                        |-- email sent -----> [inbox]
     |<-- { ok: true } ---------------|                        |                    |
     |                                |                        |                    |
  2. |-- [user clicks link] --------> GET /auth/callback       |                    |
     |                                ?code=<otp>&next=/proj   |                    |
     |                                |-- exchangeCodeFor      |                    |
     |                                |   Session(code) ------>|                    |
     |                                |<-- session + cookie ---|                    |
     |<-- 302 /projects + cookie -----|                        |                    |
     |                                |                        |                    |
  3. |-- GET /projects (cookie) ----->| middleware.ts          |                    |
     |                                | getUser() ------------>|                    |
     |                                |<-- { user } ----------|                    |
     |                                |-- SELECT workspaces ---|------------------>|
     |                                |   (RLS: membership)    |                    |
     |                                |-- SELECT modules ------>                   |
     |                                |-- SELECT user_stories ->                   |
     |                                |-- SELECT atcs ---------->                  |
     |<-- page rendered (project tree)|                        |                    |
```

**Narrative**:
1. The magic-link request is the only fully public auth endpoint — no credentials required, just an email address.
2. Supabase Auth manages OTP delivery. The callback route at `/auth/callback` is the only exchange point — it validates the code, sets the session cookie, and redirects. Open-redirect guard: `next` must start with `/` and not `//`.
3. On landing at `/projects`, `middleware.ts` runs on every request and calls `supabase.auth.getUser()`. If the session cookie is missing or expired, the user is redirected to `/login?next={path}`.
4. Project tree loading uses parallel Supabase reads (modules → stories → ATCs) inside a React Server Component. No REST endpoint is involved — this is direct Supabase SDK usage.

---

### Journey 3 — Test Run Creation + Item Execution (Pass/Fail/Skip)

**Business purpose**: A QA engineer marks ATCs as pass, fail, skip, or blocked during a test session.

**Endpoints involved**: None via REST — ATC status updates go through Supabase PostgREST directly (UPDATE `atcs.status`).

**Entities touched**: `atcs` (status field), `atc_steps`, `atc_assertions`

```
  Browser (QA Engineer)           PostgREST (RLS-backed)        Supabase DB
       |                               |                             |
  1. [Load ATC list for project]        |                             |
       |-- GET <supabase>/rest/v1/atcs  |                             |
       |   ?project_id=eq.<uuid>        |                             |
       |   Cookie: sb-*-auth-token      |                             |
       |                               |-- SELECT atcs WHERE ... --->|
       |                               |   (RLS: active member)      |
       |<-- [{ id, title, status }] ---|                             |
       |                               |                             |
  2. [QA marks ATC as pass]            |                             |
       |-- PATCH <supabase>/rest/v1/   |                             |
       |   atcs?id=eq.<atc-uuid>       |                             |
       |   { status: "pass" }          |                             |
       |                               |-- UPDATE atcs WHERE id= --->|
       |                               |   (RLS: can_write_workspace)|
       |                               |                             | TRIGGER: set updated_at
       |<-- 200 [{ id, status }] ------|                             |
       |                               |                             |
  3. [QA marks ATC as fail]            |                             |
       |-- PATCH .../atcs?id=eq.<uuid> |                             |
       |   { status: "fail" }          |-- UPDATE atcs ------------->|
       |<-- 200 ----------------------|                             |
```

**Narrative**:
1. ATC status (`unrun/running/pass/fail/skipped/blocked`) is the system's execution state machine. There is no `test_runs` table — status reflects only the LAST known result. Historical execution tracking does not exist yet.
2. Status transitions happen via direct PostgREST PATCH on the `atcs` table. RLS (`bunkai_can_write_workspace`) enforces that only active members with write-capable roles can update.
3. The `status` field has no DB CHECK constraint beyond the app-layer enum. Any string could be written via PostgREST if a caller bypasses the UI. This is a test-data integrity risk.
4. No `test_runs` entity exists — this is a confirmed Phase 2 gap. See §7.

---

### Journey 4 — Workspace Invitation + Member Onboarding

**Business purpose**: A workspace admin grows the team by inviting a new QA engineer, who joins and gains access to workspace data.

**Endpoints involved**: POST `/api/v1/workspaces/{id}/invites` → POST `/api/v1/invites/accept`

**Entities touched**: `workspace_invites`, `workspace_invite_secrets`, `workspace_members`

```
  Admin (browser)                 Bunkai API                    Supabase DB
       |                              |                              |
  1. [Issue invite]                   |                              |
       |-- POST /api/v1/              |                              |
       |   workspaces/{id}/invites    |                              |
       |   Cookie: sb-*-auth-token    | generateInviteToken()        |
       |   { email, role }            | hashInviteToken(rawToken)    |
       |                              |-- INSERT workspace_invites ->|
       |                              |   (RLS: must be admin/owner) |
       |                              |-- INSERT workspace_invite_   |
       |                              |   secrets (admin client) --->|
       |                              |-- console.log(accept_url)    |
       |<-- { invite, token,          |                              |
       |      accept_url, warning }---|                              |
       |                              |                              |
  [Admin shares accept_url out-of-band — no email sent in MVP]
       |                              |                              |
  Invitee (browser, signed-in)        |                              |
       |                              |                              |
  2. [Accept invite]                  |                              |
       |-- POST /api/v1/              |                              |
       |   invites/accept             | hashInviteToken(token)       |
       |   Cookie: sb-*-auth-token    |-- SELECT workspace_invite_   |
       |   { token: rawToken }        |   secrets WHERE hash= ------>|
       |                              |<-- { invite_id } ------------|
       |                              |-- SELECT workspace_invites   |
       |                              |   WHERE id= ---------------->|
       |                              |<-- { email, role, expires } -|
       |                              | Validate:                    |
       |                              |   revoked_at IS NULL         |
       |                              |   accepted_at IS NULL        |
       |                              |   expires_at > now()         |
       |                              |   invite.email == user.email |
       |                              |-- UPSERT workspace_members ->|
       |                              |   { status: active }         |
       |                              |-- UPDATE workspace_invites   |
       |                              |   SET accepted_at = now() -->|
       |<-- { ok, workspace_id, role }|                              |
       |                              |                              |
  [Invitee now sees workspace via RLS membership]
```

**Narrative**:
1. Invite issuance is admin/owner-gated via RLS — a member trying to issue an invite gets a PostgREST 403 mapped to the API's `forbidden` code. The `owner` role cannot be granted via invite (code constraint).
2. The raw token is returned once in the API response and logged to stdout (`console.log`). MVP has no transactional email — the inviter must copy the URL from the response or server logs.
3. Token acceptance validates four conditions in sequence: not revoked, not already accepted, not expired, email match (case-insensitive). Any violation returns a `conflict` or `forbidden`.
4. The UPSERT on `workspace_members` promotes an `invited` status to `active`. The member gains full RLS access to workspace data immediately after acceptance.

---

### Journey 5 — PAT Issuance + Revocation

**Business purpose**: A QA engineer mints a long-lived token for their CI pipeline and later revokes it when it is no longer needed.

**Endpoints involved**: POST `/api/v1/tokens` → GET `/api/v1/tokens` → DELETE `/api/v1/tokens/{id}`

**Entities touched**: `access_tokens`, `access_token_secrets`

```
  Browser (session-authenticated)  Bunkai API                    Supabase DB
       |                               |                              |
  1. [Mint PAT]                        |                              |
       |-- POST /api/v1/tokens         |                              |
       |   Cookie: sb-*-auth-token     | generateSecret(32 bytes)     |
       |   { scopes, name,             | tokenPrefix = secret[0:12]   |
       |     expires_in_days }         | hash = SHA-256(secret)       |
       |                               |-- INSERT access_tokens ------>|
       |                               |   (admin client)              |
       |                               |-- INSERT access_token_secrets>|
       |<-- 201 { id, token:           |                              |
       |    "bk_pat_<p>.<r>",          |                              |
       |    scopes, warning } ---------|                              |
       |                               |                              |
  [Token stored by caller — never retrievable again]
       |                               |                              |
  2. [List PATs — no secrets]          |                              |
       |-- GET /api/v1/tokens          |                              |
       |   Cookie: sb-*-auth-token     |-- SELECT access_tokens ----->|
       |                               |   (RLS: uid = user_id)       |
       |<-- { tokens: [...] }          |   (no hash in select)        |
       |                               |                              |
  3. [Revoke PAT]                      |                              |
       |-- DELETE /api/v1/tokens/{id}  |                              |
       |   Cookie: sb-*-auth-token     |-- UPDATE access_tokens ------>|
       |                               |   SET revoked_at = now()     |
       |                               |   WHERE id = ? AND           |
       |                               |   revoked_at IS NULL         |
       |<-- 204 No Content ------------|   (RLS: uid = user_id)       |
```

**Narrative**:
1. PAT issuance REQUIRES a cookie session — a PAT cannot mint another PAT. This prevents credential proliferation from compromised tokens.
2. The `bk_pat_` family prefix enables GitHub and GitGuardian secret-scanning to auto-detect and alert on leaked tokens in code repositories.
3. Revocation is a soft-delete: `revoked_at` is stamped, the row stays for audit. The bearer middleware rejects any token where `revoked_at IS NOT NULL`.
4. GET `/api/v1/tokens` never returns the hash — the `access_token_secrets` sibling table is never joined in the listing query.

---

### Journey 6 — Workspace Creation

**Business purpose**: A user creates a new isolated tenant (workspace) and becomes its owner, ready to invite teammates and start testing.

**Endpoints involved**: POST `/api/v1/workspaces`

**Entities touched**: `workspaces`, `workspace_members`

```
  User (browser)                  Bunkai API                    Supabase DB
       |                              |                              |
       |-- POST /api/v1/workspaces    |                              |
       |   Cookie: sb-*-auth-token    | Zod validate:                |
       |   { name, slug }             |   SLUG_REGEX                 |
       |                              |   RESERVED_SLUGS check       |
       |                              |                              |
       |                              |-- RPC bunkai_bootstrap_ ---->|
       |                              |   workspace(slug, name)      | (SECURITY DEFINER)
       |                              |                              | INSERT workspaces
       |                              |                              | INSERT workspace_members
       |                              |                              |   { role: owner, status: active }
       |                              |                              | RETURNS workspace_id uuid
       |                              |<-- workspace_id -------------|
       |                              |-- SELECT workspaces -------->|
       |                              |   WHERE id = workspace_id    |
       |<-- 201 { workspace } --------|                              |
```

**Narrative**:
1. The RPC `bunkai_bootstrap_workspace` is `SECURITY DEFINER` — it runs as the function owner and can bypass RLS to atomically create both the workspace and the owner membership in a single transaction.
2. Slug validation at the app layer catches reserved words (`api`, `admin`, `login`, etc.) before the DB is touched. The DB enforces global uniqueness via a UNIQUE constraint.
3. New workspaces always start with `plan = 'community'`. Plan upgrades are not yet implemented in any API route.

---

### Journey 7 — Headless Agent Bootstrap (Signup + Workspace + Project)

**Business purpose**: An AI agent provisions itself from scratch — creates a user, gets a PAT, creates a workspace, and is ready to author test cases in a single sequence.

**Endpoints involved**: POST `/api/v1/auth/signup` → POST `/api/v1/workspaces` → (PostgREST for project/module creation)

**Entities touched**: `access_tokens`, `access_token_secrets`, `workspaces`, `workspace_members`, `projects`, `modules`

```
  AI Agent                        Bunkai API                    Supabase DB
       |                              |                              |
  1.   |-- POST /api/v1/auth/signup   |                              |
       |   { email, password,         |-- admin.auth.createUser() -->|
       |     pat_scopes, pat_name }   |   (email_confirm: true)      |
       |                              |-- signInWithPassword() ------>|
       |                              |-- mintPat() ---------------->|
       |<-- 201 { user, session, pat }|                              |
       |    "bk_pat_<p>.<r>"          |                              |
       |                              |                              |
  2.   |-- POST /api/v1/workspaces    |                              |
       |   Cookie: <session cookie>   |-- bunkai_bootstrap_workspace>|
       |   { name, slug }             |                              |
       |<-- 201 { workspace }---------|                              |
       |                              |                              |
  3.   [Create project + modules via PostgREST — no REST endpoint]
       |-- POST <supabase>/rest/v1/   |                              |
       |   projects { workspace_id,   |                              |
       |   name, slug }               |-- INSERT projects (RLS) ---->|
       |<-- 201 { project } ----------|                              |
       |                              |                              |
  4.   [Use PAT for all subsequent API calls]
       |-- GET /api/v1/me             |                              |
       |   Authorization: Bearer ...  |-- bearer.ts verify -------->|
       |<-- { user, workspaces }      |                              |
```

**Narrative**:
1. The signup endpoint uses `email_confirm: true` via the admin auth API — no confirmation email, no human click required. This is intentional for CI/agent provisioning.
2. The session returned in the signup response is needed for workspace creation (cookie auth). The PAT cannot create a workspace directly — POST `/api/v1/workspaces` accepts cookie only.
3. Project and module creation have no REST endpoints — they go through PostgREST with the Supabase anon key + session token. This is the current architectural gap.
4. Once a workspace exists, the PAT can drive all ATC-scoped operations (with `atc:write` scope).

---

## 4. Architecture Behind the API

```
  +--------------------+   +--------------------+   +--------------------+
  | Browser (QA Eng /  |   | CLI / AI Agent      |   | CI Pipeline        |
  | Manager)           |   | (PAT Bearer)        |   | (PAT Bearer)       |
  +--------+-----------+   +--------+------------+   +--------+-----------+
           |                        |                          |
           | HTTPS (Cookie)         | HTTPS (Bearer)           | HTTPS (Bearer)
           |                        |                          |
           +------------------------+--------------------------+
                                    |
  +------------------------------+--+----------------------------------+
  |                              Vercel (CDN + Serverless)            |
  |                                                                    |
  |  +---------------------------+   +-----------------------------+  |
  |  | Next.js Edge Middleware   |   | API Discovery / Docs        |  |
  |  | middleware.ts             |   | GET /api/v1   (banner)      |  |
  |  | - Supabase session check  |   | GET /api/openapi (spec)     |  |
  |  | - Redirect unauth browser |   | GET /api/docs  (Scalar UI)  |  |
  |  | - Cookie refresh          |   +-----------------------------+  |
  |  +---------------------------+                                    |
  |                                                                    |
  |  +---------------------------------------------------------------+ |
  |  | Next.js App Router — Serverless Function per route           | |
  |  |                                                               | |
  |  | withApiHandler (lib/api/handler.ts)                          | |
  |  |   - x-request-id injection                                   | |
  |  |   - structured JSON logging to stdout                        | |
  |  |   - ApiError / ZodError → envelope                           | |
  |  |                                                               | |
  |  | requireAuth (lib/api/auth.ts)                                | |
  |  |   Bearer -> requireBearerToken (lib/api/middleware/bearer.ts) | |
  |  |   Cookie  -> supabase.auth.getUser()                         | |
  |  |                                                               | |
  |  | Route Handlers (app/api/v1/**)                               | |
  |  |   Auth routes     /auth/magic-link  /auth/signup  /signin   | |
  |  |   Identity        /me  /me/active-workspace                  | |
  |  |   Workspaces      /workspaces  /workspaces/{id}              | |
  |  |   Invites         /workspaces/{id}/invites  /invites/accept  | |
  |  |   Tokens (PAT)    /tokens  /tokens/{id}                      | |
  |  |   Health          /health                                    | |
  |  +---------------------------------------------------------------+ |
  +------------------------------+----------------------------------+--+
                                 |
              +------------------+------------------+
              |                                     |
  +-----------+-----------+           +-------------+-----------+
  | Supabase Auth         |           | Supabase Postgres 17    |
  | (Magic Link OTP)      |           | (Primary data store)    |
  | signInWithOtp()       |           |                         |
  | exchangeCodeForSession|           | RLS on all 14 tables    |
  | admin.auth.createUser |           | SECURITY DEFINER helpers|
  |                       |           |   bunkai_is_workspace_  |
  | Failure: 502 upstream |           |   member / admin / owner|
  | (Bearer PAT auth is   |           |                         |
  |  DB-only, unaffected) |           | RPCs:                   |
  +-----------------------+           |   bunkai_bootstrap_ws   |
                                      |   bunkai_save_atc       |
                                      |                         |
                                      | PostgREST (direct):     |
                                      |   projects, modules,    |
                                      |   user_stories, ACs,    |
                                      |   atcs (status update)  |
                                      |                         |
                                      | Triggers:               |
                                      |   atcs_set_updated_at   |
                                      |   atcs_refresh_tsv (GIN)|
                                      +-------------------------+
```

### Component Table

| Component | Role | Persistence / Integrations touched | Why it matters for QA |
|-----------|------|------------------------------------|----------------------|
| `middleware.ts` (Next.js Edge) | Session guard for browser routes | Supabase Auth (getUser) | If the middleware fails or misconfigures PROTECTED_PREFIXES, unauthenticated users reach protected pages. First defense layer. |
| `withApiHandler` (`lib/api/handler.ts`) | Request envelope, logging, error normalization | stdout (Vercel logs) | All 4xx/5xx responses flow through here. The `x-request-id` is the correlation key for bug reports and test assertions. |
| `requireAuth` (`lib/api/auth.ts`) | Dual-mode auth dispatcher | Supabase Auth (cookie path) / `access_tokens` + `access_token_secrets` (bearer path) | The seam between the two auth tiers. Bearer-first detection means a stale cookie never shadows an explicit PAT. |
| `requireBearerToken` (`lib/api/middleware/bearer.ts`) | PAT verification | `access_tokens`, `access_token_secrets` (admin client) | Contains the fixed bug (prefix+remainder hashing). The regression test surface for PAT auth correctness. |
| `mintPat` (`lib/api/pat.ts`) | PAT generation | `access_tokens`, `access_token_secrets` (admin client) | Called by three endpoints (signup, signin, tokens). Secret generation and the sibling-table write are atomic across both inserts. |
| Route handlers (`app/api/v1/**`) | Business logic | Supabase DB via createClient() or createAdminClient() | Thin handlers — Zod validation + RLS delegation + RPC calls. Most logic is in DB (RLS + RPCs). |
| Supabase PostgREST (direct) | ATC, project, module, story, AC CRUD | Supabase Postgres 17 (RLS active) | No app-layer validation for direct PostgREST writes. RLS is the only guard. Schema-level constraints (FK, UNIQUE, CHECK) are the safety net. |
| `bunkai_bootstrap_workspace` RPC | Atomic workspace + owner creation | `workspaces`, `workspace_members` | SECURITY DEFINER — bypasses RLS. Used in one path only. Failure is non-transient (slug uniqueness 23505 → 409). |
| `bunkai_save_atc` RPC | Full-replace ATC save | `atcs`, `atc_steps`, `atc_assertions`, `atc_acceptance_criteria` | SECURITY INVOKER — RLS still applies. The only write path that atomically handles all ATC sub-entities. Any partial save is impossible by design. |
| `lib/api/idempotency.ts` | POST replay protection | `idempotency_keys` table | Infrastructure exists but zero routes use it. Dead code risk — any future consumer must call `beginIdempotentRequest` + `recordIdempotencyResult`. |

---

## 5. External Integrations

| Service | Trigger | Direction | Failure mode (user-visible) | Journeys affected |
|---------|---------|-----------|----------------------------|-------------------|
| Supabase Auth (Magic Link) | POST `/api/v1/auth/magic-link` | Outbound sync | 429 → `rate_limited` (429); other errors → `upstream_error` (502); OTP delivery failure silent from API perspective | Magic Link Auth (Journey 2) |
| Supabase Auth (OTP Exchange) | GET `/auth/callback?code=` | Inbound (from email link) | Exchange failure → redirect to `/login?error=otp_exchange_failed`; browser sees login page, not 5xx | Magic Link Auth (Journey 2) |
| Supabase Auth (Admin — user create/signin) | POST `/api/v1/auth/signup`, `/auth/signin` | Outbound sync | `upstream_error` (502) or `unauthorized` (401); request fails hard | Headless Bootstrap (Journey 7), PAT auth (all bearer journeys) |
| Supabase Postgres 17 (PostgREST + SDK) | Every API request, every PostgREST call | Outbound sync | Full application outage — all reads/writes depend on this service | All journeys |
| Vercel (Serverless hosting) | HTTP request routing | Infrastructure | Application unavailable; no failover documented | All journeys |
| Email provider (via Supabase) | Magic link OTP delivery | Outbound async (managed by Supabase Auth) | Silent to Bunkai API — email delivery status not tracked; user never receives link but API returns 200 | Magic Link Auth (Journey 2) |
| Jira Cloud | User story creation (UI form, `external_id` / `external_url` fields) | Reference only — no API calls | No failure mode — field is plain text; Jira availability irrelevant | None (decorative link) |
| DBHub MCP (QA tooling) | AI agent / QA engineer DB inspection | Outbound (QA tooling, not app) | QA tooling unavailable; application unaffected | None (test tooling only) |

**Notable absences**:
- `RESEND_API_KEY` is present in `.env.example` but no Resend SDK call exists in any route — transactional email is planned but not implemented. Invite links are logged to stdout only.
- `N8N_API_URL` / `N8N_API_KEY` exist in `.env.example` — n8n is a QA repo integration, not used in the target application.
- `TAVILY_API_KEY` same — QA repo only.
- `SUPABASE_JWT_SECRET` is in `.env.example` as optional — no custom JWT claim signing implemented yet.

---

## 6. Cross-References

### Data-Map Entities This API Exposes

The following entities from `.context/business/business-data-map.md` are directly exposed or modified through REST endpoints:

| Entity | Exposed via | REST endpoint |
|--------|-------------|---------------|
| `workspaces` | GET/POST/PATCH | `/api/v1/workspaces`, `/api/v1/workspaces/{id}` |
| `workspace_members` | Indirectly (via invite accept) | `/api/v1/invites/accept` |
| `workspace_invites` + `workspace_invite_secrets` | GET/POST/PATCH/DELETE | `/api/v1/workspaces/{id}/invites`, `/api/v1/workspaces/{id}/invites/{inviteId}` |
| `access_tokens` + `access_token_secrets` | GET/POST/DELETE | `/api/v1/tokens`, `/api/v1/tokens/{id}` |
| `magic_link_tokens` + `magic_link_token_secrets` | Write-only (audit, best-effort) | POST `/api/v1/auth/magic-link` (side effect) |
| `idempotency_keys` | Read/Write (wired but unused) | No current route calls `beginIdempotentRequest` |
| `atcs`, `atc_steps`, `atc_assertions`, `atc_acceptance_criteria` | PostgREST + RPC (no REST) | `<supabase-url>/rest/v1/rpc/bunkai_save_atc` |
| `projects`, `modules`, `user_stories`, `acceptance_criteria` | PostgREST only | `<supabase-url>/rest/v1/{table}` |
| `feature_flags` | Read-only (no routes yet) | Phase 2 — not wired |
| `activity_log` | Append-only (service_role writes) | No direct exposure |

Full entity descriptions, state machines, and relationship narratives: `.context/business/business-data-map.md`

### OpenAPI Spec

- **Generated file**: `public/openapi.json` in the target repo (`upex-bunkai-tms`)
- **Served at**: `GET /api/openapi` (static, force-cached 300 seconds)
- **Interactive docs**: `GET /api/docs` (Scalar UI)
- **Regenerate**: `bun run openapi:gen` in the target repo

### TypeScript Types

- **Generated file**: `lib/types/supabase.ts` in the target repo
- **Regenerate**: `bun run types:gen`
- **QA boilerplate sync**: `bun run api:sync` → `api/schemas/` in this repo

### Related Context Files

| File | What it contains |
|------|-----------------|
| `.context/business/business-data-map.md` | Entity schemas, state machines, all business flows, DB triggers |
| `.context/infrastructure/backend.md` | Full endpoint catalog with exact request shapes and status codes |
| `.context/SRS/architecture.md` | C4 diagrams, RLS strategy, security model, tech stack |

---

## 7. Discovery Gaps

| Gap | Severity | Evidence | Notes |
|-----|----------|----------|-------|
| **No REST endpoints for ATC / Project / Module / Story / AC CRUD** | HIGH | No `app/api/v1/atcs/`, `projects/`, `modules/` route files exist | All creation and CRUD for core test entities goes through PostgREST directly. RLS is the only guard — no Zod validation, no application-layer business rules (e.g. AC binding enforcement) at this layer. A caller with `member` RLS access could write an ATC with an empty `p_ac_ids` array via raw PostgREST. |
| **PAT Bearer bug regression** | HIGH | Confirmed fixed in `lib/api/middleware/bearer.ts` (`fullSecret = prefix + remainder`) and migration `0012_drop_legacy_token_hashes.sql` | The original implementation hashed only `remainder`, not `prefix + remainder`. Every Bearer call silently returned 401. Requires a regression ATC that mints a PAT, makes a Bearer request, and asserts 200. |
| **idempotency_keys infrastructure is dead** | MEDIUM | `lib/api/idempotency.ts` is complete (begin/record/discard lifecycle); zero `app/api/v1/` routes call `beginIdempotentRequest` | The feature is not yet available to callers despite the table and code existing. Any future activation must wire all three lifecycle calls per endpoint. Idempotency key TTL cleanup (24h rows) has no cron job. |
| **Transactional email not implemented** | MEDIUM | `RESEND_API_KEY` in `.env.example` but no Resend import in any route | Workspace invite links are returned in the API response body and logged to `console.log` (Vercel stdout). In production, the inviter must relay the URL manually. This is an MVP-acknowledged gap. |
| **Plan feature gating not enforced** | MEDIUM | `workspaces.plan` has a CHECK constraint (`community/cloud/enterprise`); `feature_flags` table exists with `enabled`/`payload`; no route reads feature flags | Plan enforcement is schema-ready but application-dormant. No rate limits, no ATC limits, no feature walls exist in any current route. Testing plan-gated features is not possible via the API today. |
| **`NEXT_PUBLIC_SUPABASE_ANON_KEY` vs `SUPABASE_PUBLISHABLE_KEY` env name mismatch** | HIGH | `middleware.ts` reads `NEXT_PUBLIC_SUPABASE_ANON_KEY`; `.env.example` defines `SUPABASE_PUBLISHABLE_KEY`; `lib/env.ts` validates `NEXT_PUBLIC_SUPABASE_ANON_KEY` | Both must be set to the same value or middleware will crash. Setting only `SUPABASE_PUBLISHABLE_KEY` in `.env` will silently break the middleware in dev. |
| **`SUPABASE_SECRET_KEY` vs `SUPABASE_SERVICE_ROLE_KEY` env name mismatch** | HIGH | `.env.example` uses `SUPABASE_SECRET_KEY`; `lib/env.ts` reads `SUPABASE_SERVICE_ROLE_KEY` | Admin client will not initialize if only the `.env.example` name is set. All routes using `createAdminClient()` will fail. |
| **ATC slug generation strategy unknown** | LOW | `atcs.slug` has a UNIQUE per-project constraint; no server-side auto-generation found in any route or migration | How slugs are assigned on ATC creation is not visible in the REST or RPC layer. May be client-side generation (front-end assigns the slug before calling `bunkai_save_atc`). |
| **`qa_inspector_ro` / `qa_inspector_rw` CREATE ROLE DDL missing from migrations** | LOW | Referenced in migration `0011_split_token_secrets.sql` REVOKE statements; CREATE ROLE not in any `.sql` file | These roles must be provisioned out-of-band via Supabase Studio or a separate infra script before QA fixtures that use BYPASSRLS will work. |
| **Supabase project reference (`<project-ref>`) not in `.env.example`** | LOW | Cookie name `sb-<project-ref>-auth-token` requires the ref; `.env.example` has `NEXT_PUBLIC_SUPABASE_URL` (`https://<project-ref>.supabase.co`) but does not call out the ref explicitly | QA fixtures that set/assert session cookies must parse the ref from `NEXT_PUBLIC_SUPABASE_URL`. |
| **`GET /api/v1/workspaces/{id}` does not support Bearer** | LOW | Route code calls `createClient()` (cookie only) without `requireAuth()`; Bearer callers will get a 401 | Inconsistency with `GET /api/v1/workspaces` (which supports both). Headless clients that need a single workspace detail must use the list endpoint and filter. |
| **No test_runs / test_run_items entity** | LOW | `atcs.status` is the only execution state; no run history table exists | Test execution history cannot be queried. Re-running a test overwrites the previous status silently. Confirmed Phase 2 feature. |
| **`activity_log` entity_type and action values undocumented** | LOW | Column is `text`, no CHECK constraint; no examples in migrations or routes | Audit log is append-only but its vocabulary is uncontrolled. Asserting audit entries in tests requires knowing what values the service_role middleware writes. |
| **business-feature-map.md not yet generated** | INFO | `.context/business/business-feature-map.md` does not exist | Journey-to-feature-ID cross-references in §3 omitted. Run `/business-feature-map` to generate and then enrich this document's journey sections with FEAT-NNN pointers. |

---

*Sources: app/api/v1/**, lib/api/**, middleware.ts, app/auth/callback/route.ts, supabase/migrations/0001–0012, .context/business/business-data-map.md, .context/infrastructure/backend.md, .context/SRS/architecture.md*
