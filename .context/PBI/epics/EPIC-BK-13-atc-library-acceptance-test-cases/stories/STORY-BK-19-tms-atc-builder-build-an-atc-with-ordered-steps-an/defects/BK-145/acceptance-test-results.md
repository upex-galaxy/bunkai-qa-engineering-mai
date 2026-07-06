# Acceptance Test Results — BK-145

**Defect:** BK-145 — ATC builder: mapApiError does not handle validation_failed + too_small
**Tester:** maibethvega
**Environment:** Code analysis (staging-dbhub blocked — magic-link auth required)
**Date:** 2026-07-06
**Overall Result:** REGRESSED — behavior is WORSE than originally described

---

## Summary

| TC | Description | Result | Notes |
|---|---|---|---|
| V-01 | DB: min-length constraint on `atcs.title` | FAILED | No CHECK constraint — `title text not null` only |
| V-02 | UI: Save button enabled for 2-char title | CONFIRMED BUG | `canSave = title.trim().length > 0` passes for "ab" |
| V-03 | UI: Submit ATC with 2-char title | REGRESSED | `saveAtcAction` + `bunkai_save_atc` accept 2-char title silently |
| V-04 | UI: Submit ATC with 3-char title | N/A | Both 2 and 3 chars save — no lower boundary enforced |
| V-05 | Error display type | N/A | No error produced at all |

---

## Root Cause Analysis

Original defect described: 422 `validation_failed + too_small` → generic toast (wrong error UI)

**Code analysis reveals broader issue — validation absent at ALL layers:**

| Layer | Expected | Actual |
|---|---|---|
| `atcs` table | `CHECK (length(title) >= 3)` | `title text not null` only |
| `bunkai_save_atc` RPC | `RAISE EXCEPTION` for `length(p_title) < 3` | No validation |
| `saveAtcAction` | `if (input.title.trim().length < 3)` | Only checks `=== 0` |
| `AtcEditor.tsx` canSave | `title.trim().length >= 3` | `title.trim().length > 0` |
| `mapApiError` utility | Handle `validation_failed + too_small` | Does not exist in codebase |

**The referenced `POST /api/v1/atcs` endpoint does not exist.** ATC save path is Supabase RPC. The `mapApiError` utility referenced in the original defect also does not exist.

---

## Scope Update

Original scope: "wrong error message when server rejects short title"
Actual scope: "no validation of title minimum length at any layer — 2-char titles save successfully"

---

## Recommendation

Update BK-145 description to reflect actual scope. Fix requires:
1. DB: Add `CHECK (length(title) >= 3)` to `atcs.title` OR enforce in RPC
2. `saveAtcAction`: Add `if (input.title.trim().length < 3)` guard
3. `AtcEditor.tsx`: Update `canSave` to `title.trim().length >= 3`

---

_Verified by code analysis — 2026-07-06_
