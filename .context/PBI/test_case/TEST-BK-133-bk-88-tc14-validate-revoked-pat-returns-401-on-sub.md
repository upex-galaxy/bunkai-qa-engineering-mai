# TEST: BK-88: TC14: Validate revoked PAT returns 401 on subsequent API call (revocation reflected immediately)

**Jira Key:** [BK-133](https://jira.upexgalaxy.com/browse/BK-133)
**Status:** Borrador
**Components:** None

---

## Test Description

## TC14 — Integration: Revoked PAT Rejected Immediately: 401

***Group******:*** D-Integration

***Precondition******:***

- User has an active PAT that has been confirmed to work for GET /api/v1/tokens
- Cookie session available to perform the DELETE revocation

***Steps******:***

1. Verify PAT works: GET /api/v1/tokens with Bearer PAT — 200
2. Revoke the PAT: DELETE /api/v1/tokens/{id} with cookie session — 200
3. Immediately retry: GET /api/v1/tokens with the now-revoked Bearer PAT
4. Inspect response status code

***Expected******:***

- Step 1: 200 OK (PAT valid)
- Step 2: 200 OK (revocation confirmed)
- Step 3: 401 Unauthorized (revocation reflected immediately, no cache lag)

***Note******:*** This integration test confirms that revocation is reflected in real-time on subsequent auth attempts, not eventually consistent.

***Auth******:*** Step 1 + 3: revoked Bearer PAT | Step 2: cookie session

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
