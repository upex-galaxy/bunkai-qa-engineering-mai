# TEST: BK-88: TC02: Validate GET /api/v1/tokens lists tokens with prefix only, no secret field, wrapped in {tokens:[...]}

**Jira Key:** [BK-121](https://jira.upexgalaxy.com/browse/BK-121)
**Status:** Borrador
**Components:** None

---

## Test Description

## TC02 — GET Happy Path: Prefix Only, Wrapped Shape

***Group******:*** A-GET

***Precondition******:***

- Authenticated user with at least one active PAT
- Bearer PAT token available (GET accepts Bearer auth)

***Steps******:***

1. GET /api/v1/tokens with Authorization: Bearer token
2. Inspect response status code
3. Inspect response body structure
4. Verify no secret field present on any token object
5. Verify response is wrapped: {tokens:[...]} (not a bare array)

***Expected******:***

- Response: 200 OK
- Body: { tokens: [ { id, name, prefix, created_at, scopes, ... } ] }
- No secret field on any token object
- prefix field present (partial token identifier only)

***Auth******:*** Bearer PAT (confirmed working in BK-109 session 2026-06-11)

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
