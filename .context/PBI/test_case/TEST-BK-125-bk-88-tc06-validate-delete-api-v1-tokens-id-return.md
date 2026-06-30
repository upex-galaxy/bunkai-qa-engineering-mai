# TEST: BK-88: TC06: Validate DELETE /api/v1/tokens/{id} returns 401 for unauthenticated requests

**Jira Key:** [BK-125](https://jira.upexgalaxy.com/browse/BK-125)
**Status:** Borrador
**Components:** None

---

## Test Description

## TC06 — DELETE Unauthenticated: 401

***Group******:*** C-DELETE

***Precondition******:***

- No auth credentials (no cookie, no Bearer token)
- Any token ID (can be a known ID or fabricated UUID)

***Steps******:***

1. DELETE /api/v1/tokens/{id} without any Authorization header and without session cookie
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
