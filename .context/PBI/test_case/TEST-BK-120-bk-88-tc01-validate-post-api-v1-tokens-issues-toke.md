# TEST: BK-88: TC01: Validate POST /api/v1/tokens issues token and returns full secret exactly once in 201 response

**Jira Key:** [BK-120](https://jira.upexgalaxy.com/browse/BK-120)
**Status:** Borrador
**Components:** None

---

## Test Description

## TC01 — POST Happy Path: Token Issued, Secret Returned Once

***Group******:*** B-POST

***Precondition******:***

- Authenticated user with valid cookie session (admin or owner role)
- Cookie session obtained via browser login on staging

***Steps******:***

1. POST /api/v1/tokens with valid body: name (80 chars max), scopes (valid enum values)
2. Inspect response status code
3. Inspect response body for secret field
4. Call GET /api/v1/tokens to verify secret is NOT returned in list

***Expected******:***

- Response: 201 Created
- Body contains full token secret (one-time reveal)
- Subsequent GET /api/v1/tokens returns prefix only, no secret field

***Auth******:*** Cookie session (POST requires cookie — Bearer PAT returns 403 intentionally)

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
