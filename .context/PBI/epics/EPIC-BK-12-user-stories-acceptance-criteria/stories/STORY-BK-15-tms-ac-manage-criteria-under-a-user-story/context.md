# Session Context — BK-15

**Sprint session started**: 2026-06-18
**QA Engineer**: Maibeth
**Env**: staging — https://staging-upexbunkai.vercel.app

---

## Open Questions Resolved (from Shift-Left)

| # | Question | Resolution |
|---|----------|------------|
| 1 | AC6 auto-revert vs block? | CONFIRMED AUTO-REVERT: "remove that criterion → the story drops back to draft" (Dev comment 6/5) |
| 2 | Drag vs up/down arrows? | CONFIRMED UP/DOWN ARROWS (Dev comment 6/5: "up/down arrows to reorder") |
| 3 | 403 vs 404 for non-member? | CLARIFIED: workspace outsider → 404; in-workspace read-only → 403 on writes (Dev comment 6/5) |
| 4 | Soft-archive field name? | `archived_at` (implementation plan: "WHERE archived_at IS NULL") |
| 5 | Reorder mechanism? | `bunkai_move_acceptance_criterion` RPC via PATCH position |

## Remaining Open Questions (needs PO/Dev confirmation during execution)

- Move-up on first AC / move-down on last AC: disabled or no-op?
- Edit AC (PATCH title) — does same validation apply?
- Post-archive visibility: hidden from all UI or restorable?
- ATC binding cascade on AC soft-archive (deferred — BK-18 blocked)

## Session Notes

_Add notes here during execution_
