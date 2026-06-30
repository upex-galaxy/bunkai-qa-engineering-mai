# TEST: BK-17: TC02: Validate idempotent re-import updates without duplicating stories

**Jira Key:** [BK-170](https://jira.upexgalaxy.com/browse/BK-170)
**Status:** Candidate
**Components:** None

---

## Test Description

## BK-17: TC02: Validate idempotent re-import updates without duplicating stories

***Related Story******:*** BK-17
***Priority******:*** Critical
***ROI******:*** 16.7 (Candidate)
***Type******:*** API + DB
***Tags******:*** @critical @regression @automation-candidate @BK-17

### Test Design

```gherkin
@critical @regression @automation-candidate @BK-17
Scenario: Validate idempotent re-import updates without duplicating stories
  """
  Related Story: BK-17
  Covers: AC2, TC-POS-02
  """

  # === PRECONDITIONS ===
  Given a completed import exists for project {project_id} with jql {jql}
  And {expected*count} user*stories rows exist with external_id matching the JQL

  # === ACTION ===
  When the user sends the same POST /api/v1/imports with identical payload

  # === VALIDATIONS ===
  Then the response status is 202 (NOT 409 — prior job is completed, not active)
  And polling shows status "completed"
  And created_count = 0
  And updated*count = {expected*count}
  And DB query SELECT count(*) FROM user*stories WHERE project*id AND external*id IN (...) returns exactly {expected*count} (zero duplicates)
  And module_id and status on existing rows are unchanged after re-import
```

### Preconditions

- TC-01 must have completed first (chained dependency)
- Same project, same JQL, same user

### Expected Results

- Re-run is safe: updates only, no creates, no duplicates
- Module placement and status survive re-import untouched

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
- **Labels:** automation-candidate, critical, e2e, regression

---

_Synced from Jira by sync-jira-issues_
