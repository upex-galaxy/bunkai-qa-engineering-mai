# TEST: BK-88: TC09: Validate POST /api/v1/tokens accepts token name of exactly 80 characters

**Jira Key:** [BK-128](https://jira.upexgalaxy.com/browse/BK-128)
**Status:** Borrador
**Components:** None

---

## Test Description

## TC09 — POST Boundary: Name = 80 Characters (Accept)

***Group******:*** B-POST

***Precondition******:***

- Authenticated user with valid cookie session

***Steps******:***

1. Generate a token name of exactly 80 characters (e.g. "A" repeated 80 times)
2. POST /api/v1/tokens with body: { name: "<80-char-name>", scopes: ["read:tokens"] }
3. Inspect response status code

***Expected******:***

- Response: 201 Created
- Token created successfully at the boundary limit

***Note******:*** This is the upper boundary acceptance test. TC10 tests rejection at 81 characters.

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
