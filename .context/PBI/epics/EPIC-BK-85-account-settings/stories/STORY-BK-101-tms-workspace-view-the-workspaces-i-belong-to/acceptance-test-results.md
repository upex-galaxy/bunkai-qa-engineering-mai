# BK-101 — Acceptance Test Results (QA)

> Jira field: `customfield_10147` · [View in Jira](https://jira.upexgalaxy.com/browse/BK-101)

# Acceptance Test Results — BK-101

# TMS-Workspace | View the workspaces I belong to

***Story******:*** BK-101
***Epic******:*** BK-85 — Account & Settings
***Test Date******:*** 2026-06-11
***Tester******:*** QA Bot (chiavassa-bunkai-qa-engineering)
***Environment******:*** staging — https://staging-upexbunkai.vercel.app
***Modality******:*** jira-native
***Overall Verdict******:*** CONDITIONAL PASS

---

## Verdict Summary

> ***SUCCESS:**** ****CONDITIONAL PASS*** — Tested scope (API + DB) passes all executed outlines. 4 outlines blocked by test data / auth infrastructure gaps (not by product defects). Blocked scope does not invalidate the tested scope.

| Category | Count |
| --- | --- |
| PASS | 5 |
| BLOCKED-AUTH (no data contract issue) | 1 |
| BLOCKED-DATA (seed required) | 2 |
| CONDITIONAL PASS | 1 |
| FAIL | 0 |
| ***BUGS FILED**** | ****0*** |

---

## Executed Test Results

### P-02: Bearer token workspace list — PASS

- ***HTTP******:*** 200
- ***auth.source******:*** bearer ✅
- ***workspaces******[******]******:*** non-empty, all required fields present (`id`, `slug`, `name`, `owner*user*id`, `plan`, `created_at`) ✅
- ***active*************workspace*************id******:*** valid UUID, present in `workspaces[].id` ✅

### N-01: Unauthenticated request → 401 — PASS

- ***GET /api/v1/me (no auth)******:*** HTTP 401 — `{"error":{"code":"unauthorized"}}` ✅
- ***GET /api/v1/workspaces (no auth)******:*** HTTP 401 ✅

### N-02: Non-member workspace → 403 — PASS

- ***POST /api/v1/me/active-workspace (foreign workspace******_******id)******:*** HTTP 403 ✅
- ***Response******:*** `{"error":{"code":"forbidden","message":"You are not a member of that workspace."}}` ✅
- ***active*************workspace*************id******:*** unchanged on follow-up GET ✅

### B-01: Oldest workspace fallback — PASS (CONDITIONAL)

- ***active*************workspace*************id******:*** matches oldest workspace by `created_at` ✅
- ***Condition******:*** QA bot has 1 workspace — trivially correct; determinism not falsifiable with single workspace
- ***Unblock path******:*** Add 2nd workspace membership for QA bot in staging

### I-01: DB cross-validation — PASS

- ***API workspaces set == DB workspace******_******members (status='active') set******:*** exact match ✅
- ***active*************workspace*************id******:*** present in both API response and DB query ✅

---

## Blocked Outlines

> ***WARNING:*** These outlines require infrastructure setup before execution. No product defect was found in the tested scope.

| Outline | Block type | Reason | Unblock path |
| --- | --- | --- | --- |
| P-01 (cookie-session workspace list) | BLOCKED-AUTH | `/api/v1/auth/login` → 404; app uses magic-link only. No password-based auth endpoint for headless testing. Bearer equivalent covered by P-02. | Playwright magic-link flow via Resend + email receive, OR staging test-bypass endpoint |
| P-03 (switch active workspace) | BLOCKED-AUTH + BLOCKED-DATA | Cookie unavailable (magic-link) + QA bot has only 1 workspace | Magic-link auth + add 2nd workspace membership for QA bot |
| N-03 (suspended membership excluded) | BLOCKED-DATA | No suspended membership for QA bot in staging. Suspended user exists but PAT hash not readable. | Seed: `UPDATE workspace*members SET status='suspended' WHERE user*id=<qa.bot.id> AND workspace_id=<any>` OR issue fresh PAT for existing suspended user |
| N-04 (invited membership excluded) | BLOCKED-DATA | Zero `workspace*members.status='invited'` rows in staging. Invite state may live in `workspace*invites` table — needs dev clarification. | Dev clarification on invite state machine + seed row |

---

## API Gap — Open Item for Dev

> ***ERROR:**** ****Role field not returned by API.*** `GET /api/v1/me` and `GET /api/v1/workspaces` return `owner*user*id` but no `role` field per workspace. The `MemberRole` enum (`viewer/member/admin/owner`) is not exposed in any endpoint. AC-1 requires role labels ("Rol: Admin") — this AC cannot be fully validated until the API is extended.
***Recommendation******:*** Extend `GET /api/v1/me` workspaces query to join `workspace_members` and return `role: MemberRole` per entry.

---

## UI Tests — Deferred (BK-87 dependency)

All 9 UI-layer outlines (P-01 to P-05 visual, B-01 to B-03 UI, I-03 navigation) are deferred until BK-87 (Settings Hub) ships and deploys to staging. No `/settings` route exists in `app/(app)/` as of 2026-06-11.

---

## Next Steps

- ***QA bot infra******:*** Add 2nd workspace + magic-link auth setup for P-01, P-03, B-01 full coverage
- ***Dev******:*** Extend API to return `role` field → re-run AC-1 role-label assertion
- ***BK-87******:*** After Settings Hub deploys → resume all UI outlines
- ***Stage 5 → test-automation******:*** P-02, N-01, N-02 are automation-ready now; I-01 needs DB fixture

---
_Synced from Jira by sync-jira-issues_
