# TEST: BK-88: TC08: Validate POST /api/v1/tokens returns 403 when member-role user issues workspace:admin scope

**Jira Key:** [BK-127](https://jira.upexgalaxy.com/browse/BK-127)
**Status:** Borrador
**Components:** None

---

## Test Description

## TC08 — POST Privilege Escalation: workspace:admin Scope by Member: 403

***Group******:*** B-POST

***Precondition******:***

- Authenticated user with valid cookie session
- User has member role in the workspace (NOT admin or owner)

***Steps******:***

1. POST /api/v1/tokens with body: { name: "test-token", scopes: ["workspace:admin"] }
2. Inspect response status code
3. Inspect response body for error message

***Expected******:***

- Response: 403 Forbidden
- Body indicates insufficient privileges to issue workspace:admin scope

> ***ERROR:**** ****Known defect BK-117 (HIGH severity)*** — This TC is expected to FAIL on current staging.
Current behavior: POST returns 201 (no role-gate enforcement).
Expected behavior: POST returns 403 Forbidden.
DB evidence: user 2742da39-... (member role) has 12 active workspace:admin PATs (confirmed 2026-06-11).
Do NOT mark as PASSED until BK-117 is resolved.

> ***INFO:*** TC08 is DISTINCT from TC07: TC07 tests an invalid enum string (e.g. "admin:all"), TC08 tests a valid scope ("workspace:admin") with insufficient role. These cover AC-4 (privilege escalation check), not AC input validation.

***Auth******:*** Cookie session (member-role user)

---

## Related Issues

- tests: [BK-88](https://jira.upexgalaxy.com/browse/BK-88) - Settings | Manage Personal Access Tokens

---

## Metadata

- **Created:** 6/12/2026
- **Updated:** 6/12/2026
- **Reporter:** Carlos Alberto Chiavassa
- **Assignee:** Carlos Alberto Chiavassa

---

_Synced from Jira by sync-jira-issues_
