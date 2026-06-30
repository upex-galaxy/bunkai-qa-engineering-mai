# TEST: BK-17: TC06: Validate RLS non-disclosure on non-member and inaccessible resources

**Jira Key:** [BK-174](https://jira.upexgalaxy.com/browse/BK-174)
**Status:** MANUAL
**Components:** None

---

## Test Description

## BK-17: TC06: Validate RLS non-disclosure on non-member and inaccessible resources

***Related Story******:*** BK-17
***Priority******:*** High
***ROI******:*** 8.9 (Manual — requires 2nd user account setup)
***Type******:*** Negative + Security
***Tags******:*** @high @regression @manual-only @BK-17

### Test Design

```gherkin
@high @regression @manual-only @BK-17
Scenario Outline: Validate RLS non-disclosure returns 404
  """
  Related Story: BK-17
  Covers: TC-NEG-01, TC-NEG-04
  """

  # === PRECONDITIONS ===
  Given user <user_type> is authenticated

  # === ACTION ===
  When the user sends <method> <endpoint> targeting <resource_type>

  # === VALIDATIONS ===
  Then the response status is 404
  And the response does not leak whether the resource exists

  Examples:
    | user*type   | method | endpoint                  | resource*type              |
    | non-member  | POST   | /api/v1/imports           | project in another workspace |
    | member      | GET    | /api/v1/imports/{other_id}| job from another workspace   |
    | member      | GET    | /api/v1/imports/{nil_uuid}| non-existent job             |
```

### Preconditions

- Non-member test requires a second Jira/staging user account NOT in the target workspace
- Inaccessible job test uses job_id `b4b8e74c-...` (belongs to "Smoke Checkout", different workspace)

### Expected Results

- RLS makes invisible resources return 404 (not 403) — by design (WAD)
- No information leakage between "exists but hidden" and "truly absent"

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
- **Labels:** e2e, high, manual-only, regression

---

_Synced from Jira by sync-jira-issues_
