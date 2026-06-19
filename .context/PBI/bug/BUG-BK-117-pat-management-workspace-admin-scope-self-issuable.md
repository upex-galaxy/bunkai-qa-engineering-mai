# BUG: PAT management: workspace:admin scope self-issuable by member-role users

**Jira Key:** [BK-117](https://jira.upexgalaxy.com/browse/BK-117)
**Priority:** High
**Status:** Abierta
**Components:** None

---

## Description

## Bug Report — Privilege Escalation: workspace:admin scope self-issuable by member-role users

***Story******:*** BK-109 (Settings | Manage Personal Access Tokens)
***AC******:*** AC-4 — workspace:admin scope requires admin/owner role
***Severity******:*** HIGH
***Found during******:*** Sprint testing — Stage 2 Execution (2026-06-11)

---

## Summary

`POST /api/v1/tokens` does not enforce role requirements for the `workspace:admin` scope. Any authenticated user — regardless of their role in the workspace — can self-issue a Personal Access Token with `workspace:admin` scope, bypassing role-based access control.

---

## Reproduction

### Expected behavior (per AC-4)

When a user with `member` or `viewer` role requests a PAT with `workspace:admin` scope:

- HTTP 403
- `{"error":{"code":"forbidden","message":"workspace:admin scope requires admin or owner role"}}`

### Actual behavior

- HTTP 201 — PAT created with `workspace:admin` scope regardless of user role

### Reproduction steps (API)

1. Obtain a cookie session for a user with `member` role in any workspace
2. `POST /api/v1/tokens` with body: `{"name":"priv-esc-test","scopes":["workspace:admin"],"workspace_id":"<any-workspace-id>"}`
3. Observe: HTTP 201 — token created successfully
4. The issued PAT now grants workspace-admin access via Bearer authentication

---

## DB Confirmation

Without needing a live API call, the bug is confirmed via DB inspection:

- ***User******:*** `2742da39-e0ff-4f0c-a0a1-88dae804e14f` (`member` role in "Bünkāï QA" and "Extra Test" workspaces)
- ***Tokens found******:*** 12 active PATs with `workspace:admin` in `scopes[]`
- ***workspace******_******id on tokens******:*** NULL (unscoped — applies to all workspaces)
- ***Date range******:*** 2026-06-06 to 2026-06-11 (ongoing, not isolated)
- ***Table******:*** `access_tokens`

This confirms the escalation path has been exercised repeatedly in staging.

---

## Root Cause

`app/api/v1/tokens/route.ts` — POST handler creates tokens for any authenticated user without:

1. Checking the user's role in `workspace_members`
2. Validating that `workspace:admin` scope is only grantable by admin/owner-role users

---

## Impact

- ***Access control bypass******:*** Member-role users gain workspace-admin API access via PAT
- ***Multi-tenant risk******:*** If PAT is used across workspaces (workspace_id = NULL), escalation is workspace-agnostic
- ***Severity******:*** HIGH — breaks the role isolation contract for workspace admin operations
- ***Affected users******:*** Any member/viewer-role user who issues a PAT with workspace:admin scope

---

## Suggested Fix

In `POST /api/v1/tokens` handler, before token creation:

```typescript
if (requestedScopes.includes('workspace:admin')) {
  const membership = await getWorkspaceMembership(userId, workspaceId);
  if (!membership || !['admin', 'owner'].includes(membership.role)) {
    return NextResponse.json(
      { error: { code: 'forbidden', message: 'workspace:admin scope requires admin or owner role' } },
      { status: 403 }
    );
  }
}
```

---

## Evidence

- DB query result: 12 active workspace:admin PATs for member-role user `2742da39-...`
- Code inspection: `app/api/v1/tokens/route.ts` — no role-gate present
- Evidence files: `.context/PBI/.../evidence/n-ac4-privilege-escalation-finding.txt`

---

## Metadata

- **Created:** 6/11/2026
- **Updated:** 6/11/2026
- **Reporter:** Carlos Alberto Chiavassa
- **Assignee:** Carlos Alberto Chiavassa
- **Labels:** Aurora, privilege-escalation, security

---

_Synced from Jira by sync-jira-issues_
