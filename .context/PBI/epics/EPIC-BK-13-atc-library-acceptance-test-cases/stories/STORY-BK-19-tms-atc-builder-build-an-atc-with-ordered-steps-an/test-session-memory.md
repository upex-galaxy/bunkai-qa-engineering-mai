# Test Session Memory — BK-19

## TMS Modality
jira-native (no Xray)

## Environment
active_env: staging
WEB_URL: https://staging-upexbunkai.vercel.app
API_URL: https://staging-upexbunkai.vercel.app/api
DB_MCP: staging-dbhub
API_MCP: staging-openapi

## Credentials
STAGING_USER_EMAIL: maibethvega.c@gmail.com
STAGING_USER_PASSWORD: (blank — magic-link auth)
Note: staging uses magic-link; STAGING_USER_PASSWORD is always blank.

## Shift-Left Status
Labels: shift-left-reviewed, shift-left-2026-06-18 (today — FRESH, <30 days)
ATP DRAFT: 43 outlines in customfield_10067 (acceptance-test-plan.md)
Stage 1 short-circuits Phases 1-3 → validates/finalizes the existing DRAFT from Phase 4 onward.

## Stage State
Session Start: completed
Stage 1 — Planning: completed
Stage 2 — Execution: completed (35 PASSED, 8 BLOCKED, 0 FAILED)
Stage 3 — Reporting: completed

## Stage 3 Outcomes
- BK-19 status: QA Approved
- Bugs filed: BK-144 (LOW), BK-145 (LOW), BK-146 (LOW)
- ATR uploaded to customfield_10147
- QA comment posted

## Key Risks (from shift-left)
1. CRITICAL — ATC Anchoring Moat: verify DB row has ac_ids after save (I-01, I-02, I-03)
2. HIGH — Cascading picker stale state (S-01, S-02)
3. HIGH — 422 error mapping — 4 codes via mapApiError (N-11, N-12, N-13)

## ATP Outlines Summary
Total: 43
- Positive: 8 (P-01 to P-08)
- Negative: 13 (N-01 to N-13)
- Boundary BVA: 9 (B-01 to B-09)
- State/Sequence: 8 (S-01 to S-08)
- Security/Integrity CRITICAL: 5 (I-01 to I-05)
