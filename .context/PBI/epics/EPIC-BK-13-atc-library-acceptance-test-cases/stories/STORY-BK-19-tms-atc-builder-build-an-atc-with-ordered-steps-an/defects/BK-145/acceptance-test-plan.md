# Acceptance Test Plan — BK-145 Verification

**Defect:** BK-145 — ATC builder: mapApiError does not handle validation_failed + too_small
**Environment:** staging (https://staging-upexbunkai.vercel.app)
**Tester:** maibethvega
**Date:** 2026-07-06
**Type:** Defect Verification

---

## Veto + Triage

- Status Abierta — not fixed, not closed → proceed
- Severity: Low. No regression risk for existing data.
- Code analysis pre-stage: mapApiError absent; no /api/v1/atcs REST route; save path is Supabase RPC
- Suspected current behavior: 2-char title SAVES (no error at all) — worse than original defect

---

## Verification Goal

Determine current behavior of short-title submission in ATC editor on staging. Three possible outcomes:
1. **STILL BROKEN** — bug as described (generic toast, not field-level)
2. **REGRESSED** — saves successfully (no validation at all)
3. **FIXED** — field-level error shown at title input

---

## ATP Outlines

### V-01: DB constraint check — does `atcs.title` have a min-length constraint?

**Precondition:** staging-dbhub accessible.
**Steps:** Query `information_schema.check_constraints` or `pg_constraint` for `atcs` table constraints on `title` column.
**Expected:** Either a CHECK constraint enforcing `length(title) >= 3` exists (constraint present) or it doesn't (no server-side guard).
**Technique:** Direct DB inspection
**Priority:** HIGH — determines root cause scope

---

### V-02: UI — Save button enabled for 2-char title

**Precondition:** Authenticated on staging; existing ATC open in editor.
**Steps:** Clear title field, type 2 chars ("ab"), observe Save button state.
**Expected (defect still present):** Save button is ENABLED (canSave = `"ab".length > 0` = true).
**Technique:** EP boundary
**Priority:** CRITICAL

---

### V-03: UI — Submit ATC with 2-char title

**Precondition:** Same as V-02; Save button enabled with "ab" as title.
**Steps:** Click "Save ATC" with 2-char title.
**Expected (original defect):** Error shown — either field-level "Title must be at least 3 characters" or generic toast "Request body failed validation."
**Expected (regression):** ATC saves successfully — title updated to "ab" in DB.
**Technique:** EP negative + boundary (BVA: 2 chars = minimum - 1)
**Priority:** CRITICAL

---

### V-04: UI — Submit ATC with 3-char title (lower valid boundary)

**Precondition:** Authenticated on staging; existing ATC open in editor.
**Steps:** Set title to "abc" (3 chars), click "Save ATC".
**Expected:** ATC saves successfully (201 / success toast).
**Technique:** BVA lower boundary
**Priority:** HIGH

---

### V-05: UI — Error display type (generic vs. field-level)

**Precondition:** V-03 triggered an error (not a save).
**Steps:** Observe where error appears: title input field (field-level) or toast notification.
**Expected (fixed):** Error text "Title must be at least 3 characters" appears below/on the title input.
**Expected (original bug):** Generic toast "Request body failed validation." or similar.
**Technique:** UI error UX inspection
**Priority:** MEDIUM (depends on V-03 outcome)

---

## Evidence Plan

- Screenshot: Save button state with 2-char title
- Screenshot: Result after clicking Save (error or success)
- Screenshot: DB query result (V-01)
- If saves successfully: screenshot of saved ATC showing 2-char title

## Pass/Fail Criteria

| Result | Decision |
|---|---|
| V-03 = saves 2-char title successfully | Defect REGRESSED — scope is larger (no min-length anywhere), update BK-145 description |
| V-03 = error shown, V-05 = generic toast | Defect STILL PRESENT as described |
| V-03 = error shown, V-05 = field-level message | Defect FIXED → transition to Cerrada |
