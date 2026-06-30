# TEST: BK-17: TC04: Validate chunked pagination across >100 issues with accurate final count

**Jira Key:** [BK-172](https://jira.upexgalaxy.com/browse/BK-172)
**Status:** Candidate
**Components:** None

---

## Test Description

## BK-17: TC04: Validate chunked pagination across >100 issues with accurate final count

***Related Story******:*** BK-17
***Priority******:*** High
***ROI******:*** 5.3 (Candidate)
***Type******:*** Integration + API + DB
***Tags******:*** @high @regression @automation-candidate @BK-17

### Test Design

```gherkin
@high @regression @automation-candidate @BK-17
Scenario: Validate chunked pagination across >100 issues
  """
  Related Story: BK-17
  Covers: AC5, TC-BND-02, TC-INT-01
  """

  # === PRECONDITIONS ===
  Given a JQL {broad_jql} returns more than 100 issues from the connected Jira instance

  # === ACTION ===
  When the user sends POST /api/v1/imports with the broad JQL

  # === VALIDATIONS ===
  Then the job completes with status "completed"
  And imported_count equals the total issue count from the JQL (e.g. 150)
  And created*count + updated*count = imported_count
  And errors is empty
  And during polling, imported_count increases monotonically (observable progress)
```

### Variables

| Variable | How to obtain |
| --- | --- |
| {broad_jql} | `project = BK ORDER BY key ASC` (returns ~150 issues as of 2026-06-21) |

### Preconditions

- Jira project must have >100 issues reachable by the JQL
- Target Bunkai project can accommodate N new stories

### Expected Results

- Worker pages through in ≤100-issue chunks per Jira Cloud limit
- Final imported_count matches the source JQL total exactly

---

## Related Issues

- tests: [BK-17](https://jira.upexgalaxy.com/browse/BK-17) - Jira Import | Pull Jira issues by JQL
- relates to: [BK-70](https://jira.upexgalaxy.com/browse/BK-70) - QA Test Repository

---

## Metadata

- **Created:** 6/21/2026
- **Updated:** 6/21/2026
- **Reporter:** Andrés Daniel Cumare Morales
- **Assignee:** Andrés Daniel Cumare Morales
- **Labels:** automation-candidate, e2e, high, regression

---

_Synced from Jira by sync-jira-issues_
