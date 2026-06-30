# TEST: BK-88: TC03: Validate DELETE /api/v1/tokens/{id} soft-revokes token returning 200 and setting revoked_at

**Jira Key:** [BK-122](https://jira.upexgalaxy.com/browse/BK-122)
**Status:** Borrador
**Components:** None

---

## Test Description

## TC03 — DELETE Happy Path: Soft Revoke, revoked_at Set

***Group******:*** C-DELETE

***Precondition******:***

- Authenticated user with valid cookie session
- At least one active PAT owned by the user (obtained via TC01 or pre-existing)
- Token ID available from GET /api/v1/tokens response

***Steps******:***

1. Record token ID from GET /api/v1/tokens
2. DELETE /api/v1/tokens/{id} with cookie session auth
3. Inspect response status code
4. Call GET /api/v1/tokens and inspect the revoked token's state
5. (Optional) Query DB via DBHub to verify revoked_at is set

***Expected******:***

- Response: 200 OK (soft revoke)
- GET after revoke: token may appear with revoked_at set (behavior TBD per PO question 1)
- DB: revoked_at timestamp set on the token row

***Auth******:*** Cookie session required (same constraint as POST)

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
