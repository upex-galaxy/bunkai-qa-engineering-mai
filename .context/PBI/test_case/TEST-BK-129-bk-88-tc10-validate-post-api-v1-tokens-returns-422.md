# TEST: BK-88: TC10: Validate POST /api/v1/tokens returns 422 when token name exceeds 80 characters

**Jira Key:** [BK-129](https://jira.upexgalaxy.com/browse/BK-129)
**Status:** Borrador
**Components:** None

---

## Test Description

## TC10 — POST Boundary: Name = 81 Characters (Reject)

***Group******:*** B-POST

***Precondition******:***

- Authenticated user with valid cookie session

***Steps******:***

1. Generate a token name of exactly 81 characters (e.g. "A" repeated 81 times)
2. POST /api/v1/tokens with body: { name: "<81-char-name>", scopes: ["read:tokens"] }
3. Inspect response status code
4. Inspect response body for validation error detail

***Expected******:***

- Response: 422 Unprocessable Entity
- Body contains validation error indicating name exceeds maximum length (80 chars)
- No token created

***Note******:*** This is the upper boundary rejection test. TC09 tests acceptance at exactly 80 characters.

***Auth******:*** Cookie session

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
