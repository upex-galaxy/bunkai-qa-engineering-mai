# TEST: BK-88: TC11: Validate GET /api/v1/tokens returns only the authenticated user's tokens (RLS isolation)

**Jira Key:** [BK-130](https://jira.upexgalaxy.com/browse/BK-130)
**Status:** Borrador
**Components:** None

---

## Test Description

## TC11 — GET RLS: User B Cannot See User A Tokens

***Group******:*** A-GET

***Precondition******:***

- Two separate users (User A and User B) each with at least one active PAT
- User A Bearer PAT available
- User B Bearer PAT available

***Steps******:***

1. GET /api/v1/tokens with User A Bearer token — record token IDs
2. GET /api/v1/tokens with User B Bearer token — record token IDs
3. Verify no token ID from User A appears in User B's response
4. Verify no token ID from User B appears in User A's response

***Expected******:***

- Each user sees only their own tokens
- Cross-tenant isolation enforced at RLS level
- Neither response contains the other user's tokens

***Auth******:*** Bearer PAT (each user's own token)

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
