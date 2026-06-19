# Comments for BK-48

[View in Jira](https://jira.upexgalaxy.com/browse/BK-48)

---

### Benjamin Segovia - 6/16/2026, 1:21:55 PM

## Acceptance Test Plan (ATP) — Shift-Left DRAFT ready for review

The ATP DRAFT lives in the Acceptance Test Plan field.

Action Required: review ambiguities, answer critical questions, confirm edge-case behavior, validate parametrization.
Refined on: 2026-06-16 — QA Shift-Left batch session
Local working copy: .context/PBI/epics/EPIC-BK-44-coverage-traceability/stories/STORY-BK-48-tms-traceability-filter-the-chain-by-verdict-modul/shift-left-refinement.md

---

### Benjamin Segovia - 6/16/2026, 1:26:30 PM

# ATP DRAFT (continued) — truncated tail sections

> The ATP DRAFT custom field hit Jira's ~200KB ADF content-size limit. These sections did NOT fit in customfield_10067 and are posted here as a continuation. Read together with the field content for the full ATP DRAFT.

## Data feasibility flags

***DATA-FEASIBILITY-RISK******:****** confirmed and concrete.***

BK-48 filters "the chain" — the assembled evidence chain (User Story → Acceptance Criterion → ATC → Test → Run → Defect) that BK-45 ("Render full US to bug evidence chain in one read") is responsible for producing. BK-45 is currently in status ***Estimation**** (per the orchestrator's known facts) / ****Shift-Left QA*** (per the synced `story.md` read during this session — status may have moved between the two pre-sprint stages; either way it is NOT in development or done). Concretely, this means:

- ***Entity / fixture missing***: There is no queryable, filterable data structure to apply BK-48's filters against. BK-45's own refinement (`shift-left-refinement.md` §1.2–1.3) confirms zero implementation of `tests`, `test*runs`/`run*results`, or `defects`/`bugs` tables across all 20 reviewed migrations, and no `GET /api/v1/user-stories/{id}/traceability` endpoint exists. BK-48 has nothing to filter — not "limited data," literally no chain-assembly capability at all.
- ***API contract gap***: A filter capability implies query parameters (verdict, module_id, date range) on a chain-assembly endpoint. That endpoint does not exist; its contract (response shape, pagination, sort/latest-run semantics) is still under PO/Dev negotiation in BK-45's own open questions (BK-45 §6, items 1–4). BK-48 cannot define its filter contract until BK-45's base contract is settled — defining filter params against an undefined response shape risks rework.
- ***Required pre-work***: BK-45 must reach at least a stable, documented chain-assembly contract (ideally "Ready For Dev" or further) before BK-48 can be implemented or meaningfully estimated. Additionally, BK-48's AC1 (filter by verdict) depends on BK-30 (Manual Execution & Runs) defining the Run verdict model, and AC2's module-scoping question depends on no new work (the `modules` entity is Active in production) but still needs the chain-assembly join to expose `module_id` per chain row.

***Second dependency gap — BK-30***: BK-48 also implicitly depends on BK-30 (Manual Execution & Runs, status Planificación per the Story's traceability section) for the Run verdict and Run `executed_at` timestamp that AC1 (verdict filter) and AC2 (date-range filter) both filter on. This is the same upstream dependency BK-45 already flagged as a CRITICAL blocker (BK-45 §1.2, §7 Main Risk Areas) — BK-48 inherits it without adding a new blocker, but it reinforces that BK-48 cannot be implemented in isolation from BK-30 either.

***Sequencing risk***: BK-48 should NOT enter sprint planning or receive an SP estimate ahead of BK-45. Recommend the PO treat BK-48 as sequenced strictly after BK-45 reaches "Ready For Dev" with a documented response contract, and after BK-30 has at least a defined Run verdict + timestamp schema. Estimating BK-48 today would commit sprint capacity against a moving target.

---

## Recommended testing strategy

### Pre-implementation

- Do not write parametrized test-data or numbered test steps yet — defer to in-sprint planning once BK-45's chain contract and BK-30's Run schema are stable (per `acceptance-test-planning.md` §0.3 Data feasibility check).
- Track BK-45 and BK-30's status; re-run this refinement's Phase 0.3-equivalent validation once either status changes, since the refined ACs here assume a specific (currently undefined) chain row shape.
- Resolve the 3 Critical PO Questions and the verdict/timestamp Dev questions before any SP estimation session.

### During implementation

- Verify filter predicates are pushed server-side (Tech Q1) early — this is the highest-leverage architectural decision affecting every later performance and boundary test.
- Validate tenant-isolation (RLS) is enforced BEFORE filter predicates apply, not after — inherits BK-45's CRITICAL cross-workspace concern; a filter that "helpfully" widens scope to bypass RLS would be a severe security regression.

### Post-implementation (in-sprint by /sprint-testing)

- Expand the 13 DRAFT outlines into full parametrized test cases with concrete module names, dates, and verdict values once BK-45's actual response shape is known.
- Add the deferred edge cases (Phase 5) as formal ACs or test-only cases per PO's confirmation.
- Re-validate AND-combination semantics and module tree-scoping against the real implementation, since both were inferred (NEEDS PO/DEV CONFIRMATION) here.

---

## Risks & mitigation

| # | Risk | Likelihood | Impact | Mitigated by which outlines |
| --- | --- | --- | --- | --- |
| 1 | BK-48 estimated/scheduled before BK-45 + BK-30 ship, committing sprint capacity to an unbuildable Story | High | High | N/A — mitigated by sequencing recommendation in `## Data feasibility flags`, not by a test outline |
| 2 | Filter implemented as tree-pruning when PO intended row-only filtering (or vice versa), causing rework | Medium | Medium | Outline "Should filter chain to failed-only evidence..." + Scenario 1.2 |
| 3 | Cross-workspace data exposure via crafted filter query params (module_id / date range referencing another workspace) | Low | Critical | Outline "Should not leak cross-workspace evidence through filter params" |
| 4 | Two distinct empty states (BK-45 "no coverage" vs BK-48 "filter matched nothing") rendered identically, confusing users into thinking a feature is broken when it's just over-filtered | Medium | Medium | Outline "Should handle filter applied when the underlying chain itself has zero evidence" |
| 5 | Module filter implemented as exact-match when users expect tree-scoping (or vice versa), producing false-negative empty results | Medium | Medium | Outline "Should filter chain to a single module and date range combination" + Critical Question 2 |

---

## Next steps

- [ ] PO answers Critical Questions before sprint planning
- [ ] Dev answers Technical Questions before estimation
- [ ] Story enters sprint at status Ready For Dev once estimated
- [ ] When Story reaches Ready For QA, `/sprint-testing` will short-circuit refinement (label `shift-left-reviewed` detected)
- [ ] ***BLOCKER***: Do not estimate or schedule BK-48 ahead of BK-45 reaching a stable chain-assembly contract and BK-30 defining its Run verdict/timestamp schema

---


_Synced from Jira by sync-jira-issues_
