# TEST: BK-88: TC04: Validate POST /api/v1/tokens returns 401 for unauthenticated requests

**Jira Key:** [BK-123](https://jira.upexgalaxy.com/browse/BK-123)
**Status:** Borrador
**Components:** None

---

## Test Description

## TC04 — POST Unauthenticated: 401

***Group******:*** B-POST

***Precondition******:***

- No auth credentials (no cookie, no Bearer token)

***Steps******:***

1. POST /api/v1/tokens without any Authorization header and without session cookie
2. Inspect response status code
3. Inspect response body for error message

***Expected******:***

- Response: 401 Unauthorized
- Body contains error indicating authentication required

***Auth******:*** None (testing unauthenticated path)

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
