# TEST: BK-17: TC05: Validate input validation rejects invalid JQL and project_id

**Jira Key:** [BK-173](https://jira.upexgalaxy.com/browse/BK-173)
**Status:** Candidate
**Components:** None

---

## Test Description

## BK-17: TC05: Validate input validation rejects invalid JQL and project_id

***Related Story******:*** BK-17
***Priority******:*** Medium
***ROI******:*** 60 (Candidate — trivial to automate)
***Type******:*** API (Boundary + Negative)
***Tags******:*** @medium @regression @automation-candidate @BK-17

### Test Design

```gherkin
@medium @regression @automation-candidate @BK-17
Scenario Outline: Validate input validation on POST /api/v1/imports
  """
  Related Story: BK-17
  Covers: TC-BND-01, TC-API-03, TC-API-04, TC-NEG-03
  """

  # === PRECONDITIONS ===
  Given the user is authenticated with a valid PAT

  # === ACTION ===
  When the user sends POST /api/v1/imports with {"project*id": "<project*id>", "jql": "<jql>"}

  # === VALIDATIONS ===
  Then the response status is <expected_status>
  And the response body contains a structured error with code "<error_code>"

  # === EQUIVALENT PARTITIONS ===
  Examples: JQL length boundaries
    | project*id                             | jql              | expected*status | error_code         |
    | ae10a3bd-574f-4caf-8076-f19a8e80f5a6   | (empty)          | 422             | validation_failed  |
    | ae10a3bd-574f-4caf-8076-f19a8e80f5a6   | k                | 202             | (none)             |
    | ae10a3bd-574f-4caf-8076-f19a8e80f5a6   | (2000 chars)     | 202             | (none)             |
    | ae10a3bd-574f-4caf-8076-f19a8e80f5a6   | (2001 chars)     | 422             | validation_failed  |

  Examples: Invalid project_id
    | project*id  | jql              | expected*status | error_code         |
    | not-a-uuid  | key in (BK-9)    | 422             | validation_failed  |

  Examples: Invalid job_id on GET
    | id          | expected*status | error*code  |
    | not-a-uuid  | 400             | bad_request |
    | 12345       | 400             | bad_request |
```

### Expected Results

- Zod validation rejects out-of-range JQL lengths with 422
- Non-UUID project_id returns 422 (not a raw DB error)
- Malformed job_id on GET returns 400

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
- **Labels:** automation-candidate, e2e, medium, regression

---

_Synced from Jira by sync-jira-issues_
