# TEST: BK-17: TC01: Validate fresh Jira import completes with accurate counts and correct API envelope

**Jira Key:** [BK-169](https://jira.upexgalaxy.com/browse/BK-169)
**Status:** Candidate
**Components:** None

---

## Test Description

## BK-17: TC01: Validate fresh Jira import completes with accurate counts and correct API envelope

***Related Story******:*** BK-17
***Priority******:*** Critical
***ROI******:*** 16.7 (Candidate)
***Type******:*** API + DB
***Tags******:*** @critical @regression @automation-candidate @BK-17

### Test Design

```gherkin
@critical @regression @automation-candidate @BK-17
Scenario: Validate fresh Jira import completes with accurate counts
  """
  Related Story: BK-17
  Covers: AC1, TC-POS-01, TC-API-01, TC-API-02, TC-INT-03
  """

  # === PRECONDITIONS ===
  Given a member of workspace {workspace_id} is authenticated with a valid PAT
  And project {project_id} exists in the workspace

  # === ACTION ===
  When the user sends POST /api/v1/imports with {"project*id": "{project*id}", "jql": "{jql}"}

  # === VALIDATIONS ===
  Then the response status is 202
  And the response body contains {"import*job*id": "{uuid}", "status": "queued"}
  And polling GET /api/v1/imports/{import*job*id} returns status "completed"
  And the completed job shows imported*count = {expected*count}
  And created*count = {expected*count}
  And updated_count = 0
  And skipped_count = 0
  And errors is an empty array
  And started*at and completed*at are populated with completed*at > started*at
  And the response envelope contains all 13 documented fields (id, workspace*id, project*id, jql, status, imported*count, created*count, updated*count, skipped*count, errors, started*at, completed*at, created_at)
  And DB user*stories table has {expected*count} new rows with matching external_id values
```

### Variables

| Variable | How to obtain |
| --- | --- |
| {workspace_id} | From test fixture or GET /api/v1/me |
| {project_id} | Pre-created test project in workspace |
| {jql} | `key in (BK-8, BK-9)` or equivalent known-good JQL |
| {expected_count} | Number of issues matched by the JQL |
| {import*job*id} | From the 202 POST response |

### Preconditions

- Authenticated user is a member of the target workspace
- Target project exists and has no active (queued/running) import jobs
- Jira credentials configured on staging (ATLASSIAN*URL/EMAIL/API*TOKEN)

### Expected Results

- 202 on POST with correct envelope shape
- Job transitions to completed with accurate counts
- DB rows match imported issues 1:1

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
