# TEST: BK-88: TC07: Validate POST /api/v1/tokens returns 422 when scopes array contains invalid enum value

**Jira Key:** [BK-126](https://jira.upexgalaxy.com/browse/BK-126)
**Status:** Borrador
**Components:** None

---

## Test Description

## TC07 — POST Invalid Scope Enum Value: 422

***Group******:*** B-POST

***Precondition******:***

- Authenticated user with valid cookie session

***Steps******:***

1. POST /api/v1/tokens with body: { name: "test-token", scopes: ["admin:all"] }
2. Inspect response status code
3. Inspect response body for validation error detail

***Expected******:***

- Response: 422 Unprocessable Entity
- Body contains validation error indicating invalid scope value
- No token created

> ***INFO:*** TC07 is DISTINCT from TC08. TC07 tests a scope string that does not exist in the AccessTokenScope enum at all (e.g. "admin:all"). TC08 tests a scope string that IS valid ("workspace:admin") but requires a privilege level the user does not have.

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
