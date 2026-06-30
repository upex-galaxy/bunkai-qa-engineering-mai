# TEST: BK-18: TC02: should reject POST /atcs with 401 when auth is missing/invalid and 403 when the token lacks atc:write scope

**Jira Key:** [BK-150](https://jira.upexgalaxy.com/browse/BK-150)
**Status:** Candidate
**Components:** None

---

## Test Description

---

**Related Story****:** BK-18  |  **Epic****:** BK-13
**ATP****:** BK-94  |  **ATR****:** BK-95  |  **Pre-Condition****:** BK-161
**Test Set****:** BK-186  |  **Regression Plan****:** BK-65
**ROI verdict****:** Candidate
**AC covered****:** auth/scope gating (cross-cutting woven here, not a standalone security TC)
**Scenario****:** see the Cucumber definition (Gherkin) on this Test.

---

---

## Related Issues

- tests: [BK-18](https://jira.upexgalaxy.com/browse/BK-18) - TMS-ATC API | Create and edit ATCs with steps and assertions

---

## Metadata

- **Created:** 6/20/2026
- **Updated:** 6/29/2026
- **Reporter:** Ely
- **Assignee:** Ely
- **Labels:** api, automation-candidate, regression, regression-candidate

---

_Synced from Jira by sync-jira-issues_
