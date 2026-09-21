# Testcase JSON schema (Version 2)

**Unchanged from Version 1.** Per the original V2 request (§2: "V2 must remain compatible with
the exact same testcase JSON schema used for V1 — no schema changes — so that
`testcase.json → V1 → result` and `testcase.json → V2 → result` are directly comparable"),
`validateTestCase()` in this folder's `app.html` accepts exactly the same fields, types,
defaults, and validation rules as V1's. See V1's
`.claude/skills/weekly-plan/references/testcase-schema.md` for the complete field-by-field
reference (top-level structure, `preferences`, `protectedBlocks[]`, `fixedEvents[]`,
`tasks[]`, the field → HTML input → internal state mapping, and the round-trip guarantee) — it
applies to this file's Dev/Test Tools import/export unchanged. The same testcase files at the
repo root (`testcase.json`, `testcase.en.json`) import successfully into V2 exactly as they do
into V1.

## What actually changed in `validateTestCase()`

Only the **error message text**, not the accepted shape or the pass/fail logic. Every
`fail(...)` call that used to push a hardcoded Chinese string now calls `tr("v.xxx", {...})`
against a matching key in the `I18N` dictionary (`zh`/`en` versions of every validation
message), so an invalid import shows its error list in whichever language the UI is currently
in. A file that was valid/invalid under V1 is valid/invalid under V2 for the exact same reason.

## What the testcase schema does **not** cover (new in V2, same principle as V1 §6)

- The `overrides/<mondayISOdate>` collection (manual per-block placements — see
  `manual-adjustment.md`) is **not** part of the testcase JSON and is not touched by
  import/export. It's schedule-adjustment state, not planning input, the same way V1's
  `reflections/` was already out of scope for the same reason. Reset (both the full "Reset
  Planner Data" dev tool and import's own atomic-replace) clears the current week's
  `overrides` doc too, so a re-run starts from a clean, unadjusted baseline.
- The selected UI language (`localStorage`, not the db) is a per-browser display preference,
  not planning data — a testcase file doesn't specify or change which language is showing when
  it's imported.

## Comparing a V1 run to a V2 run

Import the same `testcase.json` into both `app.html` (V1) and this folder's `app.html` (V2)
with nothing dragged yet — the two calendars should be pixel-for-pixel the same layout (same
algorithm, same input, same schema). V2's calendar then additionally accepts manual drags on
top of that identical baseline; V1's doesn't. That's the intended point of comparison, not a
difference to treat as a bug.
