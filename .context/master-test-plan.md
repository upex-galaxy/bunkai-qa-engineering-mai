# Master Test Plan — upex-bunkai-tms

> Generated: 2026-06-19 | Mode: UPDATE (from 2026-06-08 original)
> Sources: business-data-map.md, business-feature-map.md, sprint-testing sessions BK-5/BK-7/BK-8/BK-9/BK-15/BK-18/BK-19
> Audience: QA engineer onboarding to Bunkai TMS

```
+-------------------------------------------------------------------------+
|                        BUNKAI TMS (分解)                                |
|   What to test in this system, and why it matters                       |
|   Multi-tenant Test Management System — Risk-Ranked Testing Strategy    |
|   Updated with empirical sprint-testing evidence (June 2026)            |
+-------------------------------------------------------------------------+
```

---

## 1. Executive Risk Map

Bunkai's most fragile areas are not the features you can see — they're the structural guarantees that are only partially enforced. The anchoring moat (the core product promise that every ATC must link to a real business requirement) is enforced by application code only; the database accepts empty AC arrays from any caller that bypasses the Server Action. Multi-tenant isolation is RLS-only for roughly 60% of entities, which means a single flawed policy is a data-leakage incident across every workspace on the instance. Sprint testing sessions (BK-5 through BK-19) have now confirmed several predicted risks and uncovered two new CRITICAL failure modes: `workspace:admin` scope self-issuable by any authenticated user regardless of role (BK-117/134/135 — 136 active privilege-escalated tokens in staging), and the ATC PATCH endpoint returning 412 PRECONDITION_FAILED while the mutation silently commits (BK-96 — split-brain state that makes `PATCH /atcs/{id}` untrusted by any well-behaved consumer).

Beyond structural risks, bugs confirmed in sprint testing reveal a recurring pattern: server-side validation is frequently missing or mismatched relative to documented specs. The description byte-cap defect (BK-143: 50,000 bytes decimal enforced instead of 51,200 binary KiB) and the missing submit guard (BK-99) are examples of spec drift that accumulates silently.

| Priority | Flow / Area                            | Why it matters                                                     | Depends on / Affects                                       |
|----------|----------------------------------------|--------------------------------------------------------------------|------------------------------------------------------------|
| CRITICAL | ATC Anchoring Moat (RPC bypass)        | Core product promise: every test proves a requirement — bypass destroys that guarantee | bunkai_save_atc RPC, atc_acceptance_criteria, SDET/agent headless callers |
| CRITICAL | Multi-tenant RLS isolation             | Cross-workspace data leakage — all 60% PostgREST-only entities rely solely on RLS | projects, modules, user_stories, acceptance_criteria, atcs |
| CRITICAL | PAT Bearer authentication              | Security regression (BK-84 confirmed only /me + /workspaces accepted Bearer; fixed but regression target) | All API flows behind Bearer auth, SDET tooling, AI agents |
| CRITICAL | workspace:admin scope privilege escalation | Member-role users can self-issue admin-scoped PATs (BK-117/134) — 136 active escalated tokens in staging | PAT issuance endpoint, workspace admin operations, multi-tenant security |
| CRITICAL | ATC PATCH If-Match split-brain         | PATCH /atcs/{id} returns 412 while mutation commits fully (BK-96) — API contract broken, retries corrupt data | ATC edit flow, SDET pipelines, optimistic locking |
| CRITICAL | Magic Link OTP — reuse window          | consumed_at never stamped → used link technically replayable      | Supabase Auth session issuance, all browser authentication |
| HIGH     | Workspace Creation + Slug isolation    | Tenant root bootstrap; slug collision or reserved-slug bypass = namespace pollution | All downstream data scoped to workspace |
| HIGH     | Invite Acceptance + Email validation   | BK-62 confirmed unconditional upsert demoted workspace owner to member; fixed but regression needed | workspace_members, RLS access for all invited users |
| HIGH     | Invite Issuance — member deduplication | BK-60 confirmed no email uniqueness check; fixed but regression needed; original design allows privilege escalation via duplicate invite | workspace_invites, role grants |
| HIGH     | Headless Signup / Sign-In + PAT mint   | Primary path for CI bots and AI agents; PAT shown once — lose it, lose the integration | access_tokens, SDET pipelines, agent onboarding |
| HIGH     | ATC Save — concurrent saves / version race | Full-replace RPC + optimistic version counter; two concurrent saves silently overwrite each other | bunkai_save_atc, atc_steps, atc_assertions, atc_acceptance_criteria |
| HIGH     | Idempotency infrastructure — dormant   | ATC saves are not idempotent; network retries from CI/agents can create duplicate content or corrupt step order | bunkai_save_atc, all POST endpoints |

---

## 2. What to Test First and Why

---

### 2.1 ATC Anchoring Moat — RPC Bypass

**Why it matters**
The product's core differentiator is structural prevention of orphan test cases — every ATC must prove at least one acceptance criterion. If that guarantee can be bypassed, Bunkai becomes just another spreadsheet tool. Any SDET, AI agent, or CI pipeline that calls `bunkai_save_atc` directly (which is entirely normal usage — it's a public RPC) can pass an empty `ac_ids` array and create a legally stored ATC with zero business rationale. The data looks correct in the DB. The UI shows it with no warning. Downstream consumers never know the link was missing.

**What commonly breaks**
The application-layer guard in `saveAtcAction` catches this path in the browser UI. It breaks when any caller bypasses that Server Action — direct PostgREST calls, `qa_inspector_rw` fixture setup scripts, SDET automation using the Supabase client directly, and AI agents calling the RPC natively. The RPC's own guard only checks `if p_ac_ids is not null`; an empty array `'{}'::uuid[]` is not null and passes straight through to the DELETE+INSERT transaction.

**Dependencies**
Requires an active workspace, project, module, user story, and at least one AC to exist (to prove you can bypass the binding). The `qa_inspector_rw` DBHub role provides direct RPC access for testing this path.

**What an experienced QA would check**
- Call `bunkai_save_atc` directly via the Supabase RPC interface with `ac_ids = '{}'::uuid[]` — verify the save either rejects with a clear error or, if it succeeds, flag it immediately as a critical defect.
- After a successful save from the UI path, verify `atc_acceptance_criteria` has at least one row matching the ATC — the trigger does not enforce this, so the row can silently disappear if the RPC is called wrong.
- Verify that deleting the last AC bound to an ATC via PostgREST (which cascades to `atc_acceptance_criteria`) does not leave the ATC orphaned without any UI or API warning.
- Confirm that the `ON DELETE RESTRICT` on `atcs.user_story_id` actually blocks story deletion when ATCs exist — test the rejection response, not just "assume it works."
- Try creating an ATC with `ac_ids` pointing to ACs from a different project/workspace and confirm RLS blocks the cross-workspace binding, not just the save action.

---

### 2.2 Multi-Tenant RLS Isolation

**Why it matters**
Roughly 60% of entities — projects, modules, user stories, acceptance criteria, ATCs — have no REST endpoint. Their only protection is Row Level Security. If a single RLS policy is misconfigured or a join in the helper function has an unintentional gap, a member of Workspace A can read, write, or corrupt data belonging to Workspace B. This is a compliance and trust incident. The policy chain is long: `atcs → projects → workspaces`, `acceptance_criteria → user_stories → modules → projects → workspaces` — each join is an opportunity for a gap.

**What commonly breaks**
Policies that rely on multi-hop joins are the most error-prone. The helper `bunkai_is_workspace_member` needs to stay in sync with the entity tree — if a new intermediate table is added, the policy chain may silently miss it. The `SECURITY DEFINER` RPCs (`bunkai_bootstrap_workspace`, `bunkai_atcs_refresh_tsv`) bypass RLS by design — if these functions are called with user-supplied data without explicit workspace validation, they become a privilege escalation vector. The `BYPASSRLS` flag on `qa_inspector_ro`/`qa_inspector_rw` database roles means QA tooling itself can observe the bypass — that's correct, but the REVOKE on secret sibling tables needs to hold.

**Dependencies**
Two test users across two isolated workspaces, both active members. Direct PostgREST access via the Supabase JS SDK. The DBHub `qa_inspector_ro` role to observe what the DB actually enforces vs. what the app believes.

**What an experienced QA would check**
- Authenticate as User A in Workspace A, then attempt a raw PostgREST SELECT on `atcs` — confirm that zero rows from Workspace B are returned.
- Attempt to INSERT a module/story/ATC with a `project_id` that belongs to Workspace B — the INSERT should fail with a policy violation, not silently succeed.
- Verify that calling `bunkai_save_atc` RPC (SECURITY INVOKER — RLS still applies) as a Workspace A user with a `p_atc_id` from Workspace B returns an unauthorized error, not a silent update to the foreign ATC.
- Confirm that a suspended workspace member (`workspace_members.status = 'suspended'`) immediately loses PostgREST read access — suspension is currently DB-schema-only with no REST API, so test it via direct DB manipulation and verify the RLS membership check reacts.
- Check that `bunkai_bootstrap_workspace` cannot be called to bootstrap a workspace for a different user — even though it's SECURITY DEFINER, verify the `auth.uid()` binding in the function is correct.

---

### 2.3 PAT Bearer Authentication — Regression

**Why it matters**
Every headless integration — CI pipelines, SDET scripts, AI agent bootstraps — authenticates via PAT Bearer tokens. Sprint testing session BK-7 (BK-84/92/93) confirmed a staging-wide `requireAuth` middleware regression: valid PATs authenticated successfully on `GET /api/v1/me` and `GET /api/v1/workspaces` but were rejected with 401 on every member-only route — imports, projects, modules, tokens. The bug was closed but represents a fragile path that breaks silently and wastes hours of SDET debugging.

**What commonly breaks**
The token parsing logic in `lib/api/middleware/bearer.ts` strips the `bk_pat_` prefix, splits on the first `.`, and uses `prefix + remainder` as the full secret for SHA-256 comparison. Any off-by-one in the string manipulation (wrong split index, wrong slice length) produces a hash that never matches, resulting in uniform 401s with no diagnostic output. Scope enforcement also has a gap: `workspace:admin` scope is self-issuable by any role (BK-117) — see §2.8 for the dedicated section on that defect.

**Dependencies**
A freshly minted PAT (post-fix), a scope-restricted PAT (e.g., `atc:read` only), and a revoked PAT for negative tests.

**What an experienced QA would check**
- Mint a new PAT via `POST /api/v1/tokens`, then immediately call `GET /api/v1/me` with the token in an `Authorization: Bearer` header — confirm correct user identity and workspace list.
- Use the same PAT on `POST /api/v1/workspaces/{id}/projects`, `POST /api/v1/projects/{id}/modules`, and `GET /api/v1/tokens` — all must return responses matching auth expectations, not 401 (BK-84 regression check).
- Call an endpoint that requires `atc:write` scope using a PAT that only has `atc:read` — verify the response is 403 `forbidden`, not 401 `unauthorized`.
- Revoke a PAT via `DELETE /api/v1/tokens/{id}`, then attempt to use it — confirm 401, not a partial success.
- Call an endpoint with an expired PAT — confirm 401 with no data leak.
- Confirm `GET /api/v1/workspaces/{id}` (single workspace) returns 401 when called with a Bearer token — this endpoint is intentionally cookie-only per confirmed architecture gap (BK-84 root cause analysis).

---

### 2.4 Magic Link OTP — Session Integrity

**Why it matters**
Magic-link is the only browser-facing authentication path. If the OTP exchange fails, all browser users are locked out. If `consumed_at` is never stamped (confirmed gap), a used magic link is technically replayable — someone with access to the email client could re-click the link and open a new session. For a multi-tenant TMS storing proprietary test suites, unauthorized session hijacking has real customer impact.

**What commonly breaks**
The OTP exchange at `/auth/callback` delegates to Supabase Auth's `exchangeCodeForSession(code)`. This path was not directly read during discovery — the implementation is inferred from the Supabase SSR pattern. If Supabase Auth changes behavior around code-to-session exchange or the callback route has a redirect misconfiguration, users get bounced to an error page with no clear recovery path. The `next` parameter open-redirect guard also lives here — a malformed value should redirect to `/projects`, not an attacker-controlled domain.

**Dependencies**
A real or stubbed Supabase Auth environment. Magic-link flows are harder to automate because the OTP arrives by email — in test environments, use the Supabase `admin.auth.generateLink()` API or direct OTP injection.

**What an experienced QA would check**
- Trigger a magic link, confirm the response is `{ ok: true }` and a 200 — even if the email delivery fails, the API response should not leak Supabase errors.
- Verify the `next` redirect guard: POST with `next=//evil.com` and confirm the callback ultimately lands on `/projects`, not the external domain.
- Confirm that Supabase rate limiting (429) propagates as `rate_limited` in the API response and is never masked as a 200 (BR-001-3).
- Verify the audit write to `magic_link_tokens` is truly best-effort: if you simulate a DB failure during the audit INSERT, the magic-link response should still return 200, not 500.
- Check that `consumed_at` remains null after a successful OTP exchange — this is the confirmed gap. Document the state and track it as a defect to be fixed before production auth data is considered trustworthy for audit purposes.

---

### 2.5 Workspace Creation + Slug Enforcement

**Why it matters**
The workspace is the tenant root. If slug uniqueness can be bypassed — via a race condition between two users creating the same slug, or via a reserved slug not being blocked — two tenants end up with colliding namespaces. Worse, a slug that matches a system path (`/api`, `/admin`, `/auth`) could shadow real routes in a Next.js App Router deployment.

**What commonly breaks**
Slug validation runs in application code (Zod) and again as a uniqueness check in the `bunkai_bootstrap_workspace` RPC. The reserved-slug list is hardcoded in the route — if someone adds a new page route without updating the reserved list, the namespace collision is undetected. Concurrent creation of the same slug can sometimes slip past the uniqueness check if the DB constraint isn't atomic enough (it is a unique constraint in Postgres, so this should be safe — but it's worth verifying the 409 response body is correct, not an uncaught DB exception). BK-51/54 confirmed reserved project slug rejection had gaps in an earlier sprint.

**Dependencies**
Authenticated user with a valid cookie session.

**What an experienced QA would check**
- Attempt to create a workspace with each reserved slug (`admin`, `api`, `auth`, `qa`, etc.) — each should return a clear validation error, not a 409 conflict.
- Submit the creation form with a slug that has leading/trailing hyphens or uppercase letters — Zod should reject it before the DB call.
- Verify the `bunkai_bootstrap_workspace` atomicity: confirm that if the RPC fails midway (simulate via a DB error in testing), no orphaned workspace row exists without an owner membership row.
- Confirm that after successful creation, the creating user has `role=owner` and `status=active` in `workspace_members` — not `invited`.
- Try creating a workspace with the name that would slugify to a reserved value — verify the live-preview in the onboarding UI shows the slug correctly before submission and that the API rejects it.

---

### 2.6 Invite Acceptance + Email Boundary

**Why it matters**
The invite acceptance flow is the front door for every new team member. The email-match validation is the only thing preventing User A from accepting an invite intended for User B. Sprint testing BK-5 confirmed two CRITICAL defects in this flow: (1) BK-62 — the `workspace_members.upsert` at `app/api/v1/invites/accept/route.ts:77-87` sets `role = invite.role` unconditionally, which caused the workspace owner who accidentally accepted a member-role invite to be demoted to `member` (confirmed on staging workspace aed86386); (2) BK-60 — no email uniqueness check against active members before invite INSERT, allowing re-invitation of existing members with a different (potentially higher) role. Both bugs were closed, but the patterns they represent — unconditional upsert, missing pre-insert checks — are regression targets.

**What commonly breaks**
Case-insensitive email comparison is documented (BR-007-6) but needs explicit testing. Token expiry is derived at the application layer (`expires_at < now()`) rather than a DB column transition — which means if the server clock skews, invite behavior changes without any visible signal. The MVP also does not send email, so the invite token is only available in the 201 response.

**Dependencies**
Two different user accounts (inviter and invitee), both authenticated via cookie session.

**What an experienced QA would check**
- Accept an invite with the exact email match — verify `workspace_members.status` transitions from `invited` to `active` and `workspace_invites.accepted_at` is stamped.
- Attempt acceptance with the correct token but a different email account — confirm 403 `forbidden` and verify no `workspace_members` row is created or modified.
- Attempt acceptance with an expired token (manipulate `expires_at` via DBHub) — confirm 403, not 500.
- Attempt acceptance after revocation — confirm the revoked status is checked before expiry.
- Attempt to accept the same valid invite twice — the second acceptance should upsert idempotently (same user, same workspace), not create a duplicate or throw an error.
- **Regression for BK-62**: As an owner, accept a member-role invite for your own email. Verify your role in `workspace_members` is NOT demoted — the upsert must check and preserve the higher role.
- **Regression for BK-60**: Attempt to invite an email address that is already an active member. Verify the response is 409, not 201.

---

### 2.7 Headless Signup + PAT Bootstrap

**Why it matters**
This is the onboarding path for all CI bots, SDET pipelines, and AI agents. It creates a user, signs them in, and mints a PAT in one round trip. If any step fails silently — user created but sign-in fails, sign-in succeeds but PAT mint fails — the caller receives a partial response and may not realize the PAT they saved is invalid. The PAT is shown exactly once: if it's malformed in the response, the only recovery is creating a new user.

**What commonly breaks**
`email_confirm: true` bypasses the normal Supabase email confirmation flow — this is intentional but means the bot user is active with no email verification. If Supabase Auth's admin SDK changes the behavior of `createUser` with `email_confirm: true`, the entire headless provisioning flow breaks silently. The conflict (409) case — when the email already exists — is important because agents running in idempotent retry loops will hit it and need to be redirected to `/auth/signin` instead.

**Dependencies**
No auth required (public endpoint). A disposable email address or a random UUID-based email for each test run.

**What an experienced QA would check**
- Perform a clean signup and verify the response includes `user.id`, `session.access_token`, `pat.token` (format `bk_pat_<12>.<remainder>`), and the `warning` field.
- Immediately use the returned PAT to call `GET /api/v1/me` — confirm the PAT is immediately valid, not subject to a propagation delay.
- Submit the same email a second time — confirm 409 `conflict` and verify the error message tells the caller to use `/auth/signin`.
- Submit a signup without `pat_name` or `pat_scopes` — verify defaults (all scopes, no expiry) are applied correctly.
- Submit with `pat_expires_in_days=366` — confirm 422 validation failure with a clear message about the 365-day limit.

---

### 2.8 PAT Scope Privilege Escalation — workspace:admin Self-Issuable

**Why it matters**
`POST /api/v1/tokens` does not enforce role-based scope restrictions. Sprint testing BK-109 (BK-117/134/135) confirmed that any authenticated user — regardless of their role — can self-issue a PAT with `workspace:admin` scope. DB inspection of staging showed 136 active workspace:admin PATs, including 19 belonging to a confirmed member-role user. Because `workspace_id = NULL` is accepted on token issuance, these tokens grant workspace-admin access across ALL workspaces, not just the user's own. This is an open defect with a known escalation path at scale.

**What commonly breaks**
The `POST /api/v1/tokens` handler creates tokens for any authenticated user without checking the user's role in `workspace_members` or validating that `workspace:admin` scope is only grantable by admin/owner-role users. The check is entirely absent from `app/api/v1/tokens/route.ts`.

**Dependencies**
A user with `member` role in at least one workspace. A cookie session for that user (PAT cannot create PAT, so cookie session required for issuance).

**What an experienced QA would check**
- As a `member`-role user, call `POST /api/v1/tokens` with `scopes: ["workspace:admin"]` — verify the response is 403 `forbidden`, not 201 (this is the regression test for the open defect BK-117).
- As an `admin`-role user, call the same endpoint with `workspace:admin` scope — verify 201 is returned (positive path must still work after the fix).
- As a `viewer`-role user, confirm `workspace:admin` scope is rejected.
- Verify the error response conforms to the standard error envelope: `{ error: { code: "forbidden", message: "workspace:admin scope requires admin or owner role" } }`.
- After the fix lands, query `access_tokens` via `qa_inspector_ro` to confirm no new `workspace:admin` tokens are being created by member-role users.

---

### 2.9 ATC PATCH If-Match Split-Brain

**Why it matters**
The ATC edit endpoint (`PATCH /api/v1/atcs/{id}`) uses an `If-Match` header for optimistic locking — the caller sends the current version number, and the server either applies the patch or returns 409 if the version is stale. Sprint testing BK-18 (BK-96) confirmed a critical split-brain defect: when `PATCH` is called with the CORRECT `If-Match` value, the server returns 412 PRECONDITION_FAILED with a non-JSON Vercel platform error page — but the mutation commits fully in the database. The ATC version increments, steps are replaced, assertions are cleared, and an `atc.updated` audit row is written. The client believes the edit failed; the database shows it succeeded.

**Why this is CRITICAL for headless consumers**
A well-behaved API client that receives 412 will retry the request. The retry now sends `If-Match: 1` (the original version), but the server has already incremented to version 2 — so the retry gets a genuine 409 conflict, and the client enters an error loop. The data is corrupted by step content duplication across retry attempts. This is not a theoretical risk — it blocks BK-19, BK-21, and BK-23, and any SDET automation that wraps the PATCH endpoint.

**Dependencies**
A PAT with `atc:write` scope. An existing ATC at a known version.

**What an experienced QA would check**
- Send `PATCH /api/v1/atcs/{id}` with a valid `If-Match` header matching the current version — verify the response is 200 with the updated ATC, NOT 412 (regression check for BK-96).
- Send `PATCH` with a stale `If-Match` value (version - 1) — verify 409 conflict with `details.current_version` in the response body.
- After a successful PATCH, query `atcs` directly via DBHub and verify `version` incremented, `title` updated, steps and assertions match the request body exactly.
- Send two concurrent PATCH requests for the same ATC — one should succeed (200), one should get 409. Verify the winning write's content is fully persisted and the losing write's content did not partially overwrite.
- Confirm the error response on 412 is a proper JSON error envelope, not a raw Vercel platform error page (this was the secondary defect in BK-96).

---

## 3. State Machines That Matter

---

### SM-1: ATC Status — Stale `running` State

The ATC status machine is operationally important for Sprint 2 execution tracking. Right now, `running` status can become permanently stale: if an execution crashes or the browser is closed mid-run, the ATC stays in `running` indefinitely. There is no timeout, no cleanup job, and no DB-level mechanism to detect or resolve a stuck `running` state. Any future test execution feature that filters by status will surface stale `running` ATCs as "in progress" when they're actually dead.

The transition that matters most is `running → unrun` (re-run signal). Right now this transition exists in the state machine definition but there is no API endpoint to trigger it. The test execution model is Sprint 2, but if you build execution features on top of the current `atcs.status` column without addressing the stale-running scenario, you'll have real run counts that don't match reported run counts.

You'll want to guard the `unrun → running` transition carefully when execution arrives — it should not be possible to move an ATC to `running` if it's already in `running`, or at minimum the system should log a warning that a stale run was clobbered.

---

### SM-2: WorkspaceMember Status — Suspension Without API Coverage

Membership status (`active`, `invited`, `suspended`) is fully supported at the DB level but suspension and reinstatement have no API routes. This means the only way to suspend a member today is via direct DB manipulation through Supabase Studio or DBHub. The RLS policies check `status = 'active'` — so a suspended member immediately loses data access when the status changes, which is correct behavior. But without an API or UI, a workspace owner cannot suspend a bad actor without direct database access. Test this gap explicitly so it's documented as a defect before a real incident makes it urgent.

Terminal states to verify: there is no path from `suspended` to `owner` — a suspended owner would be an inconsistent state. Verify the DB constraints or RLS policies prevent this edge case.

---

### SM-3: PAT Lifecycle — Revocation Is Irreversible

PAT revocation (`revoked_at` stamped) is a terminal state. There is no unrevoke path. The same is true for expiry. This is by design for audit purposes — but the consequence is that a mis-revoked PAT for a CI pipeline requires a new `POST /api/v1/tokens` call with a valid cookie session, which a headless agent cannot do on its own. The `revoked` and `expired` states need to return identical uniform 401 responses — never reveal which check failed.

The `last_used_at` fire-and-forget update is worth testing for a subtle failure mode: if the background update throws an unhandled exception, does it propagate and poison the auth response, or does it genuinely fail silently? The documented intent is silent failure — verify the implementation matches.

---

### SM-4: WorkspaceInvite — Expiry Is Application-Layer Only

Invite expiry is not a DB column transition — it's an `expires_at < now()` check at the application layer. This means: no DB trigger resets the invite to `expired` status, the `status` field in the API response is computed at read time, and an invite that's "expired" from the app's perspective can theoretically still be accepted if the application-layer check is bypassed. Verify the expiry check is applied consistently in the acceptance route, the list route (status display), and any invite detail endpoint — they all need to agree on what "expired" means.

The token rotation feature (POST on an existing invite) resets `expires_at`, `accepted_at`, and `revoked_at`. This means a previously-accepted invite can be re-opened. Verify that re-opening a once-accepted invite doesn't allow the original invitee to accept again (they're already a member) AND doesn't allow a different user with the new token to accept using the original invitee's email requirement.

---

## 4. Silent Killers — Automated Processes

---

### SK-1: ATC Full-Text Search Trigger (`bunkai_atcs_refresh_tsv`)

**What it does**: Fires on every ATC INSERT or UPDATE of `title`/`tags`. Rebuilds the `tsv` GIN index column from `to_tsvector('english', title + tags)`.

**What breaks if it fails**: Search becomes stale. ATCs saved after the failure don't appear in search results. The failure is completely invisible — no error surfaces in the UI, no alert fires, the save completes with a 200 and the editor shows success. The next team member who tries to find an ATC by title or tag comes up empty and assumes the ATC doesn't exist.

**How failure is detected today**: It isn't. There is no monitoring, no smoke test, no alert on this trigger. You'd only notice during a QA session when a recently-saved ATC doesn't appear in search.

**Recommended QA strategy**: After every ATC save, immediately run a PostgREST query against `atcs` with a `tsv @@ to_tsquery(...)` filter using the exact title just saved. If the row doesn't appear, the trigger fired incorrectly or didn't fire at all. This should be a standard assertion in any test that creates or updates an ATC.

---

### SK-2: `magic_link_tokens.consumed_at` Never Stamped

**What it does**: Should mark an OTP token as consumed when the user clicks the magic link and exchanges the code. Currently, no code path sets `consumed_at`.

**What breaks**: The audit trail is structurally incomplete. Any rate-limiting logic or replay-detection built on `consumed_at` in Phase 2 will silently use null values and behave as if no tokens were ever consumed. More critically, a used OTP link remains "available" in the audit records — if you add replay detection later, it will be retroactively reading a permanently dirty dataset.

**How failure is detected today**: Only by querying `magic_link_tokens` directly and observing that `consumed_at` is always null after successful OTP exchanges.

**Recommended QA strategy**: File a defect immediately. Track this column as a known integrity gap in test-session notes. Any future sprint that touches auth audit logic should verify this is fixed before building on top of it.

---

### SK-3: `touchLastUsed` Fire-and-Forget on Bearer Auth

**What it does**: After every successful PAT authentication, fires a background UPDATE to stamp `access_tokens.last_used_at`. Documented as fire-and-forget — failure is swallowed.

**What breaks if it fails silently**: Audit data for PAT usage becomes unreliable. If a team is making security decisions based on `last_used_at` to identify stale tokens for rotation, those decisions are based on wrong data. Worse, because the failure is swallowed, there's no log entry or alert — you can't know this is happening without directly inspecting `access_tokens` rows and correlating with server access logs.

**How failure is detected today**: Not detected. No monitoring on this path.

**Recommended QA strategy**: After a PAT-authenticated request, query `access_tokens.last_used_at` for the token and verify it was updated. Do this as a post-condition check in PAT authentication tests.

---

### SK-4: Activity Log Write — service_role Only, No Read Endpoint

**What it does**: Logging middleware writes audit entries to `activity_log` via service_role after every significant action.

**What breaks if it fails**: Audit trail goes dark. Compliance and security review have no paper trail for the affected period. Because writes are service_role and there is no REST read endpoint, the only way to verify the log is functioning is via DBHub. There are no alerts, no monitoring, and no heartbeat on this table.

**How failure is detected today**: Not detected until a security review asks "what happened to workspace X" and the log is empty.

**Recommended QA strategy**: After actions like workspace creation, invite issuance, and PAT minting, query `activity_log` directly via `qa_inspector_ro` to confirm rows were written. Include this as a post-condition assertion in smoke tests.

---

### SK-5: Idempotency Key Table — Accumulating Without Cleanup

**What it does**: `idempotency_keys` rows are inserted on POST requests. They have a 24-hour TTL via `expires_at`, but there is no cron job that deletes expired rows.

**What breaks**: The table grows indefinitely. At scale, this becomes a table bloat issue that can slow down the uniqueness constraint check on `(user_id, endpoint, key)`. This is currently a non-issue because idempotency isn't wired to any route — but when wiring happens in the next sprint, this becomes day-one technical debt.

**How failure is detected today**: Not detected until the table size causes performance degradation.

**Recommended QA strategy**: Flag this as a defect in the backlog before idempotency wiring ships. Any sprint that wires idempotency should include a cron cleanup job as a prerequisite.

---

## 5. External Integrations — Failure Points

---

### Supabase Auth

**Which flows stop if it goes down**: Magic link issuance (FEAT-001) and all browser authentication are completely blocked. The UI becomes a login page that can never proceed. Headless signup and signin (FEAT-002, FEAT-003) also fail — they use the Supabase admin SDK to create and authenticate users.

**PAT authentication is immune**: Bearer auth uses only the Postgres DB. A Supabase Auth outage doesn't affect SDETs and AI agents already holding valid PATs. This is by design and is an important resilience property worth testing — confirm that `GET /api/v1/me` with a Bearer token still works when Auth is simulated-down.

**Critical timeouts and quirks**: Rate limiting at 429 must propagate as `rate_limited` — not masked as 200 (BR-001-3). The OTP email itself is managed entirely by Supabase's email provider — if Supabase's email delivery is delayed, users will wait for a link that may or may not arrive. There is no resend mechanism documented in the current routes.

**Acceptable degradation**: PAT-authenticated headless flows continue normally. Browser users are locked out.

---

### Supabase Postgres

**Which flows stop if it goes down**: Everything. Every data read and write depends on Postgres. There is no caching layer, no read replica, no fallback datastore. The health probe at `GET /api/v1/health` does not perform a DB connectivity check — it would still return 200 during a Postgres outage.

**Critical quirk**: The health endpoint being DB-blind means a load balancer or uptime monitor relying on it for availability signals will report the system as healthy even when every user request is failing. This needs to be fixed before the system goes to production.

**Acceptable degradation**: None. Full outage.

---

### Vercel

**Which flows stop**: Application is entirely unavailable. No failover documented.

**Environment variable risk**: Supabase keys (`NEXT_PUBLIC_SUPABASE_ANON_KEY` / `SUPABASE_PUBLISHABLE_KEY`) are configured in Vercel project settings. The documented env-var name mismatch between `middleware.ts` and `.env.example` means a fresh Vercel deployment from `.env.example` alone will have a broken middleware that fails silently — the session cookie won't be read correctly. This must be verified during any new environment setup.

**Vercel edge + If-Match header risk (BK-96)**: Sprint testing confirmed that Vercel's edge layer may intercept the `If-Match` request header and emit a 412 before or independently of the serverless function body, even when the function already committed the mutation. Any PATCH endpoint using `If-Match` must be verified for Vercel-layer interference specifically — this is not reproducible in local dev environments.

---

### DBHub (MCP — QA Tool)

**Which flows stop if it goes down**: QA tooling and automated fixture setup are affected. The application itself is unaffected.

**Key risk**: The `qa_inspector_rw` role has `BYPASSRLS` and write access. If DBHub credentials leak or are misconfigured to point at production, the role can read all non-secret tables and write arbitrary test data. The REVOKE on secret sibling tables (`access_token_secrets`, `workspace_invite_secrets`, `magic_link_token_secrets`) is the only protection — verify those REVOKEs are actually applied in the target environment.

---

### Jira Cloud (Reference Only)

No integration risk. The `external_id` and `external_url` fields on `user_stories` are plain text — no live API calls are made to Jira. A Jira outage has zero impact on Bunkai. A Jira key stored in `external_id` is not validated for existence or format — any string value is accepted.

---

## 6. Dependency Cascade Between Flows

```
Auth (cookie session)
  +---> Workspace Creation ---> workspace_members (active) ---> All RLS access
  |                              |                                |
  |                              +--> Invite Issuance --> Invite Accept
  |                                   [BK-60: email uniqueness gap — FIXED]
  |                                                         |
  |                                                         +--> workspace_members (active)
  |                                                              [BK-62: role overwrite — FIXED]
  |                                                              (same RLS access gate)
  |
  +---> Headless Signup/Signin ---> PAT Mint
                                      |
                                      +--> Bearer Auth ---> All PAT-gated endpoints
                                      |    [BK-84: bearer only /me+/workspaces — FIXED]
                                      |    [BK-117: workspace:admin scope leak — OPEN]
                                      |
                                      +--> /api/v1/me (identity resolution)

Project Tree (PostgREST via RSC)
  +---> Modules ---> User Stories ---> Acceptance Criteria
        [BK-57: PATCH rename+move not atomic]    |
                                                  +--> AC Description: 50,000 bytes max
                                                       [BK-143: decimal not binary KiB — OPEN]
                                                       |
                                                  +--> ATC Editor (bunkai_save_atc)
                                                            |
                                                            +--> atc_steps
                                                            +--> atc_assertions
                                                            +--> atc_acceptance_criteria (anchoring moat)
                                                            |
                                                       PATCH /api/v1/atcs/{id}
                                                       [BK-96: If-Match split-brain — CLOSED]
```

**Chain 1 — Auth → RLS → Everything**: If the workspace member record for a user is missing or suspended, that user has zero RLS access to any data in any table. Every test that requires data access must first verify the membership state is `active`. A test that bypasses this by inserting data directly via `qa_inspector_rw` will create data that the application user cannot see — tests will pass in isolation but fail in integration.

**Chain 2 — PAT → Bearer Auth → SDET Pipelines**: BK-84 empirically confirmed a staging-wide `requireAuth` middleware regression that silently blocked PATs on all member-only routes. Any SDET automation that doesn't verify the PAT is valid immediately after minting will spend hours debugging downstream `unauthorized` errors. Workspace endpoint Bearer gap (GET /workspaces/{id} is cookie-only) also lives in this chain — an agent that calls `/me` successfully but then fails on `/workspaces/{id}` will look like a permissions problem, not a known architectural limitation.

**Chain 3 — AC Existence → ATC Anchoring**: The `AnchoringPanel` in the ATC editor loads all ACs for all stories in the project. If no ACs exist (new project, no stories created yet), the panel is empty and the save button is functionally disabled by the application guard. But via direct RPC, an empty `ac_ids` array goes through. Test fixture setup order matters here — create workspace → project → module → story → AC BEFORE attempting ATC creation, or you'll be testing the wrong code path.

---

## 7. Edge Cases Developers Commonly Forget

---

**Concurrency**

- Two users simultaneously calling `bunkai_save_atc` on the same ATC: the RPC does a full DELETE+INSERT of steps, assertions, and bindings. Postgres will serialize these (transaction isolation), but the `atcs.version` counter is the optimistic-lock handle. If User B saves while User A's save is in flight, B's response arrives with a newer version. When A tries to save, they're submitting against a now-stale version — the UI doesn't yet have version-conflict detection, so A's save silently overwrites B's changes. This is a data-loss scenario with no warning. BK-96 confirmed the PATCH version-conflict path is already broken in a different way — fix that before adding UI-level conflict detection.
- Concurrent workspace creation with the same slug: the `bunkai_bootstrap_workspace` RPC wraps workspace + member inserts in a transaction, and the slug UNIQUE constraint will cause the second concurrent insert to fail. Verify the 409 response is clean, not a raw Postgres error leaking through the error handler.

---

**Permission Boundaries**

- A `viewer`-role workspace member attempting to call PostgREST directly to INSERT a user story or ATC — the RLS `bunkai_can_write_workspace` helper should reject this. Most tests use `member` or `admin` roles; explicitly test `viewer` role boundaries.
- A PAT with `atc:read` scope attempting to call `bunkai_save_atc` — confirm the scope check rejects this before reaching the DB, not after.
- A workspace `admin` attempting to grant `owner` role via invite — the `workspace_invites.role` CHECK constraint blocks this at the DB level. Verify the application error message is descriptive, not a raw constraint violation.
- A `member`-role user minting a PAT with `workspace:admin` scope (BK-117) — this should be 403 once the fix lands. Until the fix lands, treat any member-issued `workspace:admin` PAT as a security incident.

---

**Idempotency**

- Retry of a PAT mint (e.g., network timeout after the DB insert but before the response): the PAT is created in the DB but the caller never receives the token (shown only once). A retry creates a second PAT. The user now has a "ghost" PAT they can never use because they don't know its value. The idempotency infrastructure exists to prevent this but is wired to no routes — document this as a day-one defect for the PAT issuance endpoint.
- Retry of `POST /api/v1/workspaces` on network timeout: the `bunkai_bootstrap_workspace` RPC is atomic, but the slug uniqueness constraint means a retry with the same slug will return 409 even though the first attempt may have fully succeeded. Callers need to check for their workspace with `GET /api/v1/workspaces` before interpreting a 409 as a hard failure.

---

**Orphaned States**

- ATC status stuck in `running` with no execution engine: the state machine allows any status to return to `unrun`, but there is no API endpoint to do this. A future execution feature must include a "reset to unrun" endpoint or the `running` state will accumulate as technical debt in every workspace.
- `workspace_invite_secrets` rows added in migration 0011: workspaces created before migration 0011 may have invite records without corresponding secret rows. If token rotation (which writes to `workspace_invite_secrets`) is called on a pre-migration invite, verify the behavior is correct rather than silently creating an orphaned entry.
- ATCs with no `atc_steps` or `atc_assertions`: the schema allows zero steps and zero assertions (only `atc_acceptance_criteria` has the application-layer minimum). An ATC with no steps is a valid DB record that an executor wouldn't know how to run. Decide whether this is allowed or should be blocked by the same guard that enforces AC binding.

---

**Cookie / Session Behavior**

- The `bk_active_ws` cookie is set with 90-day maxAge. A user who is suspended from a workspace while that cookie is still valid will have a cookie pointing to a workspace they can no longer access. The `/me` endpoint has a fallback (falls back to oldest visible workspace) but the stale cookie lingers silently — the UI may show unexpected behavior between workspace refresh cycles.
- Cookie auth (`createClient()`) and admin client (`createAdminClient()`) use different Supabase client modes. A route that accidentally uses the wrong client — cookie path for an admin operation, or admin path for a user operation — either bypasses RLS inappropriately or rejects a legitimate request. Any new route should have its client mode verified in code review.

---

**Data Validation — Byte Cap / Unit Mismatch**

- AC description and user story description byte caps use **decimal kilobytes** (50,000 bytes = 50 × 1000), not binary kibibytes (51,200 bytes = 50 × 1024). Sprint testing BK-15 (BK-143) confirmed that 51,200-byte payloads are rejected with 422 even though the spec says "50 KB." This is a specification-vs-implementation drift — either the spec needs to say "50,000 bytes" or the code needs to use `50 * 1024`. Both client-side counter display and server-side validation must agree on which unit is being used (BK-99 confirmed the submit guard was also missing — both client and server gaps coexisted).
- Any future field with a size limit: explicitly specify and test BOTH the decimal and binary interpretations, and include a test at exactly the boundary value (e.g., 50,000 bytes, 50,001 bytes, 51,200 bytes, 51,201 bytes).

---

## 8. Pre-Release Checklist

Ordered CRITICAL first, then HIGH. Maximum 15 items.

1. **Verify `bunkai_save_atc` RPC rejects empty `ac_ids` array** — call the RPC directly with `p_ac_ids = '{}'::uuid[]` and confirm a DB-level error is returned, not a silent success.
2. **Verify cross-workspace RLS isolation for all PostgREST-only entities** — authenticated as Workspace A member, confirm zero Workspace B rows returned from `atcs`, `user_stories`, `acceptance_criteria`, `modules`, `projects`.
3. **Verify PAT authentication end-to-end, all member-owned routes** — mint a fresh PAT, use it on `GET /api/v1/me`, `GET /api/v1/tokens`, `POST /api/v1/workspaces/{id}/projects` — confirm 200 on each (regression for BK-84).
4. **Verify `workspace:admin` scope requires admin/owner role** — as a member-role user, `POST /api/v1/tokens` with `workspace:admin` scope must return 403, not 201 (regression for BK-117/134 — currently OPEN).
5. **Verify `PATCH /api/v1/atcs/{id}` with correct `If-Match` returns 200 and commits once** — confirm split-brain (412 + silent commit) is resolved (regression for BK-96).
6. **Verify invite acceptance preserves higher role on upsert** — owner accepts a member-role invite for own email; confirm owner role is NOT demoted (regression for BK-62).
7. **Verify invite issuance rejects duplicate email** — `POST /invites` with email of an active member returns 409, not 201 (regression for BK-60).
8. **Verify `magic_link_tokens.consumed_at` gap is documented as a defect** — confirm the column exists, confirm it is null after OTP exchange, file defect before production auth goes live.
9. **Verify invite acceptance email-match check with mixed-case email** — `User@Example.COM` invite, `user@example.com` acceptor, confirm acceptance succeeds; `other@example.com` acceptor, confirm 403.
10. **Verify reserved slug rejection on workspace creation** — test all 16 reserved slugs return validation error, not 409 conflict.
11. **Verify env var setup with only `.env.example` values** — deploy a fresh environment using exactly the keys in `.env.example`; confirm `middleware.ts` reads `NEXT_PUBLIC_SUPABASE_ANON_KEY` correctly (known mismatch with `SUPABASE_PUBLISHABLE_KEY`).
12. **Verify AC/story description byte cap boundary** — test at 49,999 bytes (accept), 50,000 bytes (accept — current behavior), 50,001 bytes (reject), 51,200 bytes (reject — per spec if decimal unit confirmed) (regression for BK-143).
13. **Verify ATC GIN search trigger post-save** — save an ATC with a unique title, immediately query `atcs` with `tsv @@ to_tsquery(unique_title)`, confirm the row is returned.
14. **Verify activity_log writes fire for key actions** — after workspace creation, invite issuance, and PAT mint, query `activity_log` via `qa_inspector_ro` and confirm rows exist.
15. **Verify health endpoint returns 200 but document DB-blind limitation** — confirm `GET /api/v1/health` returns 200 and is correctly documented as a liveness probe only (not a readiness probe); file a defect requesting DB connectivity check before production.

---

## 9. What Is NOT in This Plan

This document is a risk-ranked testing rationale. The following artifacts are downstream of it:

- **Flow-level diagrams and state-machine transition tables** → `.context/business/business-data-map.md`
- **Feature catalog, CRUD matrix, feature flags, per-feature test coverage ratings** → `.context/business/business-feature-map.md`
- **API endpoint inventory, contracts, schemas** → `bun run api:sync` and `/business-api-map` (when generated)
- **Detailed test case definitions, traceability links, ATP/ATR documents** → TMS via `/test-documentation`
- **Sprint-level execution order and test results** → `.context/PBI/` via `/sprint-testing`
- **Planned features not yet in schema** (Test Run History, ATC Search REST endpoint, Feature Flag enforcement, Idempotency wiring, Project Creation UI, OAuth login, Member Suspension UI, Transactional Email) → Sprint 2 planning; test strategy TBD when implementation lands
- **Performance and load testing** → out of scope for current sprint; no baseline established
- **Accessibility testing** → out of scope for current sprint; no `data-testid` strategy documented

---

## 10. Discovery Gaps

These are things this plan cannot fully ground in evidence from the available sources.

**Gap 1 — `auth/callback` route not directly read**
The OTP exchange logic at `/auth/callback` was inferred from the Supabase SSR pattern and `backend.md` description, but the actual file content was not read. The open-redirect guard on `next`, the `exchangeCodeForSession(code)` call, and the redirect behavior are assumed — not confirmed. Any test of the magic-link callback flow should start by reading this file.

**Gap 2 — ATC slug generation strategy unknown**
`atcs.slug` is UNIQUE per project but no slug-generation code was found in any API route, migration, or lib file. It's unknown whether slugs are user-provided, auto-generated on INSERT via a trigger, or set by a code path not yet discovered. Tests that create ATCs need to either provide a slug explicitly or discover the generation strategy first.

**Gap 3 — `bunkai_save_atc` DB-level empty `ac_ids` enforcement not confirmed from migration source**
The data map and feature map both document this as a confirmed gap based on migration file review, but the exact migration 0007 DDL was not reproduced here. Verify directly against the live DB by calling `\sf bunkai_save_atc` in psql or via DBHub before asserting defect vs. design.

**Gap 4 — No CI pipeline means no baseline regression data**
This plan cannot reference "historical breakage rates" from automated runs. Every risk rating is based on structural analysis and sprint-testing session findings. As the first test suite is built, update this plan with actual observed failure patterns. Sprint sessions BK-5 through BK-19 provide the closest thing to empirical failure history available today.

**Gap 5 — `qa_inspector_ro`/`qa_inspector_rw` DB roles are provisioned out-of-band**
These roles are REVOKED from secret sibling tables in migration 0011, but their `CREATE ROLE` DDL is absent from all migrations. A fresh DB setup from migrations alone will not have these roles — they must be created manually via Supabase Studio or a separate out-of-band script. Any QA environment setup doc needs to capture this step explicitly.

**Gap 6 — `workspace_invite_secrets` migration 0011 timing**
Invites created before migration 0011 (which introduced the secret sibling pattern) may not have corresponding `workspace_invite_secrets` rows. The data behavior of token rotation on pre-0011 invites is unknown — this gap affects any workspace that was bootstrapped before 0011 was applied.

**Gap 7 — BK-96 root cause not confirmed (Vercel edge vs. application layer)**
BK-96 (PATCH If-Match split-brain) is closed but the root cause between two hypotheses was not confirmed: (a) the `If-Match` precondition handler comparing the version against the post-increment value; (b) Vercel edge intercepting the `If-Match` header independently of the function body. Until the confirmed root cause is documented, any PATCH endpoint using `If-Match` should be treated as a potential regression target after Vercel-side changes or middleware updates.

**Gap 8 — `workspace:admin` scope enforcement fix not yet deployed**
BK-117/134/135 are open. As of 2026-06-19, member-role users can still self-issue `workspace:admin` PATs. Any test environment that has been used for sprint testing should be treated as having privilege-escalated tokens in its `access_tokens` table. A cleanup sweep (revoke all `workspace:admin` PATs owned by non-admin/non-owner users) is recommended before using staging as a clean baseline for security tests.

**Gap 9 — Feature map was available at generation time**
The feature map was present and fully read. No feature-map limitation applies to this plan. All 34 features (+ planned) were incorporated into risk scoring.

---

*Sources used: `.context/business/business-data-map.md` (991 lines), `.context/business/business-feature-map.md` (1094 lines), sprint-testing bug tracker (BK-5, BK-7/BK-8/BK-9, BK-15, BK-18/BK-19 sessions — 25+ confirmed defects incorporated)*
*Regenerate with: `/master-test-plan` after running `/business-data-map` or `/business-feature-map` when content is updated*
