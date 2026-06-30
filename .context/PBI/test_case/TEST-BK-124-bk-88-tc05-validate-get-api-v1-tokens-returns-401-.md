# TEST: BK-88: TC05: Validate GET /api/v1/tokens returns 401 for unauthenticated requests

**Jira Key:** [BK-124](https://jira.upexgalaxy.com/browse/BK-124)
**Status:** Borrador
**Components:** None

---

## Test Description

## TC05 — GET Unauthenticated: 401

***Group******:*** A-GET

***Precondition******:***

- No auth credentials (no cookie, no Bearer token)

***Steps******:***

1. GET /api/v1/tokens without any Authorization header and without session cookie
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
