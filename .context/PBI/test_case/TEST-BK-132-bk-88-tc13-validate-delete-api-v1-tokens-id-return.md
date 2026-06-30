# TEST: BK-88: TC13: Validate DELETE /api/v1/tokens/{id} returns 404 when revoking an already-revoked token

**Jira Key:** [BK-132](https://jira.upexgalaxy.com/browse/BK-132)
**Status:** Borrador
**Components:** None

---

## Test Description

## TC13 — DELETE Idempotency: Already-Revoked Token: 404

***Group******:*** C-DELETE

***Precondition******:***

- User has a previously revoked PAT (token ID known)
- Cookie session active for the token owner

***Steps******:***

1. Obtain a token ID that is already revoked (via TC03 or pre-existing revoked token)
2. DELETE /api/v1/tokens/{id} for the already-revoked token
3. Inspect response status code

***Expected******:***

- Response: 404 Not Found
- API does not return 200 for an already-revoked token (idempotency handled via 404)

***Auth******:*** Cookie session (token owner)

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
