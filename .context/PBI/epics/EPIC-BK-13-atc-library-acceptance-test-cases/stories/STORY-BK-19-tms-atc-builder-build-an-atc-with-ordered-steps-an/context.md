# Context — BK-19: TMS-ATC Builder

## Session Notes
- Sprint-testing started: 2026-06-18
- Shift-left refresh done same day (2026-06-18) — 43 outlines ready
- STAGING_USER_PASSWORD blank by design (magic-link flow)
- BK-18 (API contract) merged and deployed — no blockers

## Open Questions
None — all 10 shift-left questions resolved via PO analysis on 2026-06-18.

## Technical Notes
- Route: app/(workspace)/modules/[moduleId]/atcs/new/page.tsx
- Submit: POST /api/v1/atcs → 201 → redirect to /atcs/{slug}
- Error mapping: mapApiError covers 4 codes (ac_outside_user_story, module_outside_project_subtree, steps_position_invalid, title_too_short)
- AC picker clears on US change (cascading dropdowns)
- Tags: component state cap at 10 + Zod validation

## Anchoring Moat — CRITICAL verification
After any successful ATC create, MUST verify via:
  GET /api/v1/atcs/{id} → response body must include non-empty ac_ids array
  OR via DB: SELECT * FROM atc_acceptance_criteria WHERE atc_id = '{id}'
