# Comments for BK-40

[View in Jira](https://jira.upexgalaxy.com/browse/BK-40)

---

### jesusgpythondev - 6/17/2026, 2:24:34 PM

## Refined Acceptance Criteria (Shift-Left QA pass - 2026-06-17)

> ***INFO:*** Refined and consolidated by QA during the pre-sprint Shift-Left review. These scenarios are the acceptance contract; reconciliation, risks, and ATP details live in the ATP / QA handoff comments.

```gherkin
Background:
  Given an authenticated project member has write access to the current project
    And a Test Run exists for that project

# ---- Happy path ----

Scenario: Open run-linked defect form from a failed step
  Given the user is executing a manual run
    And a run step has status "failed"
    And the run context includes module, executed steps, failing ATC, and captured evidence references
  When the user selects "Report defect"
  Then the defect form opens
    And module is prefilled from the run context
    And steps-to-reproduce are prefilled from executed steps
    And the failing ATC is linked or displayed
    And captured evidence references are listed
    And run, step, and ATC context cannot be reassigned manually

Scenario: Save a valid run-linked defect
  Given the defect form was opened from a failed run step
    And title length is between 5 and 200 characters
    And severity is P1, P2, P3, or P4
    And module belongs to the current project
    And evidence link count is 10 or fewer
  When the user saves the defect
  Then the defect is created in "open" state
    And it is visible in the defects list
    And it preserves the run, step, ATC, module, and evidence context

Scenario: Save a standalone defect
  Given the user opens the defects area
  When the user creates a defect with valid title, module, severity, and reproduction details
  Then the defect is created in "open" state
    And no run, step, or ATC link is required

# ---- Negative ----

Scenario: Hide run-linked defect action for non-failed steps
  Given a run step is not in "failed" state
  When the user views the step actions
  Then "Report defect" is not available for that step

Scenario: Reject invalid title length
  Given the user is filing a defect
  When the title has fewer than 5 characters or more than 200 characters
  Then saving is blocked
    And a clear title-length message is shown

Scenario: Reject missing or cross-project module
  Given the user is filing a defect
  When the module is empty or belongs to another project
  Then saving is blocked
    And no defect is created

Scenario: Reject invalid severity
  Given the user is filing a defect
  When severity is missing or not P1, P2, P3, or P4
  Then saving is blocked
    And a clear severity message is shown

# ---- Boundary ----

Scenario: Enforce evidence link limit
  Given the user is filing a defect
  When exactly 10 evidence links are attached
  Then saving remains allowed if all other fields are valid
  When an 11th evidence link is added
  Then the extra link is blocked
    And a clear evidence-limit message is shown

# ---- Integration ----

Scenario: Defect remains TMS-native without Jira sync
  Given a valid defect is filed from a failed run step
  When the defect is saved
  Then it is stored as a Bunkai TMS defect
    And it is linked to the relevant project, module, and run context when present
    And Jira issue creation or sync is not required for BK-40
```

***Copied from Refined AC by QA - Shift-Left pass 2026-06-17. PO ownership of this contract returns after Estimation grooming; any further AC edits must go through PO.***

---

### jesusgpythondev - 6/17/2026, 2:24:34 PM

## QA Handoff Mirror - BK-40 Shift-Left Review Update

> ***SUCCESS:*** BK-40 is now formatted using the same rich ADF pattern validated by BK-39 and the BK-91 formatter showcase: fenced `gherkin` for ACs, visual risk signals, and compact mirror content.

## Executive Summary

BK-40 defines how a QA Engineer files a TMS-native defect from a failed run step, while preserving run context and avoiding duplicate typing. The primary quality concern is context integrity: the saved defect must keep the correct run, step, ATC, module, and evidence references without allowing cross-project leakage.

## Refinement Delta

-  AC format corrected to a fenced `gherkin` block in the AC comment.
-  Expert-panel decisions retained: failed-step-only MVP; standalone filing in scope; Jira sync/file upload out of scope.
-  Custom-field REST update is still blocked by env/API mismatch, so AC/ATP remain fallback comments.
-  This comment is the QA mirror only; full AC and ATP content live in their dedicated fallback comments.

## ATP Draft Summary

| ID | Focus | Priority |
| --- | --- | --- |
| ATP-P1 | Open run-linked defect form | High |
| ATP-P2 | Save valid run-linked defect | High |
| ATP-P3 | Save standalone defect | Medium |
| ATP-N1 | Non-failed step has no report action | High |
| ATP-N2 | Invalid title length blocked | Medium |
| ATP-N3 | Missing/cross-project module blocked | High |
| ATP-N4 | Invalid severity blocked | Medium |
| ATP-B1 | Evidence limit enforced | High |
| ATP-I1 | No Jira sync required | High |

## High And Medium Risks

> ***ERROR:*** High: wrong run/step/ATC context, cross-project module injection, and evidence leakage. Covered by ATP-P1, ATP-P2, ATP-N3, and ATP-B1.

> ***WARNING:*** Medium: non-failed step filing, invalid title, invalid severity, and standalone missing context. Covered by ATP-N1, ATP-N2, ATP-N4, and ATP-P3.

## Open Confirmations

None blocking. Expert recommendations stand unless PO/Dev intentionally expand scope.

## Dependency Note

BK-40 depends on BK-35 failed-step state and BK-70 defect foundation. BK-41/BK-42/BK-43 remain downstream and out of scope.

## QA Story Points Recommendation

- Recommendation: 5 SP
- Confidence: 0.82
- Basis: effort=Med; complexity=Med; uncertainty=Low-Med; risk=Med
- Re-estimation triggers: Jira sync; file upload; blocked/skipped-step filing; expanded permissions; BK-35 contract change.
- Boundary: QA recommendation only; Jira estimate remains canonical unless explicitly updated.

## Out Of Scope

Jira sync/export, file upload evidence, defect lifecycle beyond initial `open`, marking a step failed, formal TC creation, automated test implementation.

## QA Publication Status

- Description updated: yes
- AC field updated: no - fallback comment updated because custom-field REST update returned 404 while `acli` mutations worked
- ATP field updated: no - fallback comment updated because custom-field REST update returned 404 while `acli` mutations worked
- QA mirror updated: yes
- Labels applied: `shift-left-reviewed`, `shift-left-2026-06-17` yes
- Read-back verification: required after this correction

---

### jesusgpythondev - 6/17/2026, 2:24:35 PM

## Acceptance Test Plan (ATP) DRAFT - BK-40

> ***INFO:*** Coverage estimate: Positive 3, Negative 4, Boundary 1, Integration 1, API 0, Total 9.

Rationale: BK-40 is a form-plus-context persistence story with one upstream run-state dependency, validation rules, boundary coverage, and security-relevant project scoping. The ATP stays outline-level because formal TC creation belongs to `/test-documentation` and automation belongs to `/test-automation`.

| ID | Outline | Priority | Precondition | Expected result | Automation hint |
| --- | --- | --- | --- | --- | --- |
| ATP-P1 | Open run-linked defect form | High | Failed run step exists | Form opens prefilled with module, executed steps, failing ATC, and evidence references | Candidate |
| ATP-P2 | Save valid run-linked defect | High | Valid defect fields and <=10 evidence links | Open defect is created, visible, and preserves run context | Candidate |
| ATP-P3 | Save standalone defect | Medium | Defects area is available | Open standalone defect is created without run/step/ATC link | Candidate |
| ATP-N1 | Non-failed step has no report action | High | Step is not failed | `Report defect` is unavailable | Candidate |
| ATP-N2 | Invalid title length blocked | Medium | Title length is 4 or 201 | Save is blocked with clear title message | Candidate |
| ATP-N3 | Missing/cross-project module blocked | High | Module is empty or outside current project | Save is blocked and no defect is created | Candidate |
| ATP-N4 | Invalid severity blocked | Medium | Severity is missing or outside P1-P4 | Save is blocked with clear severity message | Candidate |
| ATP-B1 | Evidence limit enforced | High | 10 evidence links are present | 10 are accepted; 11th is blocked with clear message | Candidate |
| ATP-I1 | No Jira sync required | High | Defect is saved from failed run step | TMS-native defect exists; Jira issue creation/sync is not required | API/DB candidate later |

## Edge Cases And Risks

> ***ERROR:*** High risks must stay visible during grooming because they affect data integrity, workspace isolation, or evidence access.

| Severity | Risk | Mitigation | Coverage |
| --- | --- | --- | --- |
| :red_circle: High | Defect filed against wrong run step | Make run/step/ATC context non-editable in run-linked flow | ATP-P1, ATP-P2 |
| :red_circle: High | Cross-project module injection | Enforce project-scope validation server-side | ATP-N3 |
| :red_circle: High | Evidence leakage across projects | Validate evidence ownership/access and max count | ATP-B1 |
| :large*orange*circle: Medium | User tries to file from non-failed step | Hide/block action | ATP-N1 |
| :large*orange*circle: Medium | Invalid severity pollutes reporting | Strict enum validation | ATP-N4 |
| :large*orange*circle: Medium | Title validation inconsistent at boundaries | Enforce 5-200 boundary | ATP-N2 |
| :large*orange*circle: Medium | Standalone defects lack run context | Require manual title/module/severity/repro fields | ATP-P3 |
| :green_circle: Low | Jira sync expectation confusion | Explicitly exclude Jira sync from BK-40 | ATP-I1 |

## QA Story Points Recommendation

- Recommendation: 5 SP
- Confidence: 0.82
- Basis: effort=Med; complexity=Med; uncertainty=Low-Med; risk=Med
- Rationale: 9 ATP outlines, one upstream state dependency, controlled validation scope, and no Jira sync/file upload keep it at 5 SP.
- Re-estimation triggers: Jira sync moves into BK-40; file upload enters scope; blocked/skipped step filing enters scope; permission model expands; BK-35 changes failed-step contract.
- Boundary: QA recommendation only; Jira Story Points / Epic / User Story fields remain canonical unless explicitly requested.

---


_Synced from Jira by sync-jira-issues_
