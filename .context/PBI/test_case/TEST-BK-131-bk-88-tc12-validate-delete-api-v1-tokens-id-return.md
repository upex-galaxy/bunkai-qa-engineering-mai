# TEST: BK-88: TC12: Validate DELETE /api/v1/tokens/{id} returns 404 when targeting another user's token (RLS)

**Jira Key:** [BK-131](https://jira.upexgalaxy.com/browse/BK-131)
**Status:** Borrador
**Components:** None

---

## Test Description

## TC12 — DELETE RLS: User B Cannot Revoke User A Token: 404

***Group******:*** C-DELETE

***Precondition******:***

- User A has at least one active PAT (token ID known)
- User B has valid cookie session
- User B is a different user than User A

***Steps******:***

1. Record User A token ID (via User A GET or DB query)
2. DELETE /api/v1/tokens/{user-a-token-id} with User B cookie session
3. Inspect response status code
4. Verify User A token is still active (not revoked)

***Expected******:***

- Response: 404 Not Found (NOT 403 — RLS returns 404 to avoid information disclosure)
- User A token remains unaffected

***Note******:*** RLS policy returns 404 (not 403) to prevent enumeration of other users' token IDs.

***Auth******:*** Cookie session (User B)

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
