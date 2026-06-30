# TEST: BK-17: TC03: Validate concurrent import returns 409 import_in_progress

**Jira Key:** [BK-171](https://jira.upexgalaxy.com/browse/BK-171)
**Status:** Candidate
**Components:** None

---

## Test Description

## BK-17: TC03: Validate concurrent import returns 409 import*in*progress

***Related Story******:*** BK-17
***Priority******:*** High
***ROI******:*** 8.9 (Candidate)
***Type******:*** API + Integration
***Tags******:*** @high @regression @automation-candidate @BK-17

### Test Design

```gherkin
@high @regression @automation-candidate @BK-17
Scenario: Validate concurrent import returns 409 import*in*progress
  """
  Related Story: BK-17
  Covers: Cross-cutting A, TC-NEG-02
  """

  # === PRECONDITIONS ===
  Given an import job is currently queued or running for project {project_id}

  # === ACTION ===
  When the user sends a second POST /api/v1/imports for the same project_id

  # === VALIDATIONS ===
  Then the response status is 409
  And the response body contains {"error": {"code": "conflict", "details": {"reason": "import*in*progress"}}}
  And DB has exactly ONE active (queued/running) import*jobs row for this project*id
```

### Preconditions

- A broad JQL import (e.g. `project = BK`) is actively running on the target project
- Must send the second POST before the first completes (~6s window for 150 issues)

### Expected Results

- Every concurrent attempt returns 409 with the exact envelope shape
- DB partial UNIQUE index prevents double-active rows

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
