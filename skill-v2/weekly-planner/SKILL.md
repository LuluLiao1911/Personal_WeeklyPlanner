---
name: weekly-planner
description: Publish or update the user's personal weekly planner web app, Version 2 — bilingual (Traditional Chinese / English) single-page planner with fixed events + tasks + preferences input, an auto-computed Monday–Sunday schedule the user can then drag/edit by hand, simplified free-time suggestions, a weekly reflection log, and PDF export that reflects the manually-adjusted plan. Use when the user asks to set up, open, update, or change their weekly planner/scheduler, or says something like "help me plan this week" / "open my weekly plan" / "update the scheduler". This is a new version of the same skill as `weekly-plan` (Version 1) — not a separate tool.
---

# Weekly Planner — Version 2

## Relationship to Version 1

**Version 1 (`../.claude/skills/weekly-plan/`, `app.html`/`app.en.html` at the repo root) is
frozen.** Nothing under this skill ever touches V1's files, database, or docs. V2 lives
entirely under `skill-v2/weekly-planner/` as its own Artifact with its own database — the two
never share runtime state, the same way V1's Chinese and English twins never did.

V1 and V2 are treated as two versions of **one skill**, not two skills: same purpose (publish
the user's weekly planner), same testcase JSON schema (see
`references/testcase-schema.md` — unchanged from V1), same core scheduling algorithm. What V2
adds is layered on top, not a rewrite: manual drag/click adjustment of the generated schedule,
a simplified free-time list, and one bilingual interface instead of two separate language
files.

## What this is

`app.html` in this folder is a complete, self-serve weekly planner: a single-page app with two
tabs, in **one** HTML file that serves both languages (a "中文 | English" toggle in the header
— not two separate Artifacts).

1. **Input tab** — forms to add/edit/delete fixed events, tasks, and preferences. Identical
   fields and behavior to V1.
2. **This Week tab** — a paper-planner-style Monday–Sunday calendar computed live in the
   browser from that data. In V2 the calendar is a starting point, not the final answer: the
   user can drag a task block to a new time, or click it to edit its time through a small
   popover, and the plan remembers that choice until they undo it or regenerate.

The scheduling algorithm (`schedule()`) and the manual-adjustment layer on top of it
(`applyOverrides()`) both run as JavaScript inside the published page — not something Claude
re-derives per request. Claude's job is to publish or update this file as an Artifact, and to
modify the JS/HTML when the user wants the tool itself changed.

## The core V2 idea

Version 1 treated the generated schedule as the final answer. Version 2 treats the generated
schedule as a **recommendation** and gives the user direct control to revise individual
placements without rebuilding the entire plan. Concretely:

- `schedule()` still computes a full baseline placement exactly as V1 did — nothing about the
  algorithm itself changed.
- A separate `overrides` db collection (one doc per week) records only the placements the user
  moved by hand: `{ blockKey: {day, start} }`.
- `applyOverrides(baseline, overrides)` merges the two at render time: every block the user
  didn't touch stays exactly where the algorithm put it; every block they did touch is
  re-validated (no overlap, stays in bounds, doesn't blow a Must task's deadline) and placed at
  its new spot, or falls back to its original spot with a `conflict` flag if the override no
  longer fits (e.g. the underlying task data changed since the move was made).
- Dragging one block never reruns the algorithm and never moves any other block. A full
  recompute only happens when the underlying input (fixed events/tasks/preferences) changes —
  the same `onSnapshot` wiring V1 already had for that.

See `references/manual-adjustment.md` for the full design: data model, the two-pass merge, drag
validation rules, deadline handling, and the undo/reset model.

## Bilingual UI

One HTML file, one artifact, one database. A `I18N = {zh:{...}, en:{...}}` dictionary plus a
`tr(key, vars)` lookup function drives all UI chrome (headings, buttons, labels, validation
messages, status text, free-time suggestions, popover labels). `data-i18n`/`data-i18n-ph`
attributes mark up the static HTML; everything generated dynamically calls `tr()` directly.
Switching language:

- Never touches user-entered data (task/event names, notes, locations, people) — only `esc()`
  ever inserts that text, `tr()` never wraps it.
- Re-applies the `lang` attribute on every native `<input type="time">`/`type="datetime-local">`
  element (including the popover's time input), not just `<html>` — Chromium renders a native
  time input's placeholder/AM-PM text from that per-element attribute regardless of the
  viewer's own browser locale, which is what V1's `app.en.html` had to work around by being a
  separate file with `lang="en-US"` baked in. V2 does the same thing dynamically in
  `applyI18n()` since there's only one file now.
- Persists the chosen language to `localStorage` (`weeklyPlannerV2Lang`) per browser, not per
  account — a fresh browser/profile defaults to Chinese.

## Standing rule inherited from V1: nothing user-visible is ever auto-translated

Task names, fixed-event names, locations, people, and notes are exactly what the user typed,
in whichever language they typed it, regardless of which UI language is selected. The example
in the original request — "微算機 Lab 作業" staying exactly as entered even with the English UI
active — is enforced structurally: `tr()` only ever looks up i18n dictionary keys, and every
call site that renders user data calls `esc()` on the raw string, never `tr()`.

## Dev/test tools (not the normal user flow)

Same panel and same purpose as V1: import/export a JSON testcase, reset all planner data,
pin a `weekStart` for reproducible comparison. **Unchanged schema** — see
`references/testcase-schema.md`. The dev-tools status messages, the JSON-schema hint text, and
the reset confirmation are all now language-aware (`tr("dev.msg*")`,
`buildDevSchemaText()`), but the accepted JSON shape and the validation rules themselves are
byte-for-byte the same as V1's `validateTestCase()` — only the error message text was made
translatable, not the logic.

## Data model (stored in the artifact's `db` capability)

Same as V1, plus one new collection:

- **`fixedEvents`**, **`tasks`**, **`prefs/main`**, **`reflections/<mondayISOdate>`** — unchanged
  from V1 (see V1's `references/testcase-schema.md` §5 for the full field mapping; V2 reads and
  writes these identically).
- **`overrides/<mondayISOdate>` docs (new)** — `{ moves: { "<taskId>::<sessionIndex>": {day,
  start} } }`. One doc per week, same key format as `reflections/`. Never contains task/event
  data itself, only a placement override keyed by a stable `blockKey` the scheduler assigns to
  each placed session (`taskId + "::" + sessionIndexWithinThatTask`). See
  `references/manual-adjustment.md`.

## Publishing / updating

- First-time setup: load `artifact-capabilities`, then `Artifact({file_path: "app.html",
  capabilities: {db: {}, downloads: {}}, icon: "calendar"})` from this folder — a separate
  Artifact and database from V1's.
- Any later change to this `app.html`: republish to the **same URL** (pass `url`).
- A change to the *scheduling logic itself* should stay identical to V1's unless the user asks
  otherwise (§1 of the original V2 request: preserve the V1 foundation). A change to *manual
  adjustment, bilingual UI, or free-time suggestions* is V2's own territory — edit freely, but
  update `references/manual-adjustment.md` if the override data model or validation rules
  change.

## Guardrails already built into `schedule()` (unchanged from V1)

- Fixed events and protected blocks are placed first and never touched again.
- A task's eligible window is clipped to its exact deadline (date **and** time).
- A Must task (or a recurring Want/Optional target) that can't fully fit produces a visible
  shortfall in the summary — never silently dropped or overbooked.
- Preference-matched placement lands inside the actual preferred window.
- Free time is never auto-filled.

## Guardrails specific to manual adjustment (new in V2)

- A drop that overlaps a fixed event, a protected block, or another scheduled block is
  rejected outright — the block snaps back to where it was, with a short on-screen explanation
  naming what it collided with.
- A drop that would push a Must task past its own deadline is rejected the same way, never
  silently accepted with a warning.
- Every move snaps to a 15-minute increment.
- A block's duration never changes through a drag or a popover edit — only its day/start time
  does.
- Fixed events and protected blocks are never draggable (shown with a small lock icon); only
  task blocks (must/want/optional) are.

## Implementation status

### Completed (implemented and exercised with Playwright + a mock `db`, per the original
request's instruction not to claim untested features work)

- Everything V1 had: full CRUD input forms, the live calendar, master task list, weekly
  reflection, PDF export, dev/test tools with the same JSON schema.
- **Simplified Free-Time Suggestions**: at most one suggestion per day (that day's single
  largest open block), phrased as a day-part ("Saturday afternoon") once the block is ≥ 3
  hours rather than always spelling out exact clock times, replacing V1's exhaustive
  per-gap listing.
- **Manual schedule adjustment**: pointer-based drag-and-drop with 15-minute snapping, a live
  valid/invalid drop-ghost preview, and a rejection toast naming the specific conflict;
  click-to-open detail popover with an alternate day/time editor that commits through the same
  validated path as dragging; a distinct "Adjusted" badge on manually-placed blocks; a
  `conflict` visual state for the rare case an override no longer fits when re-applied.
- **Undo Last Move** (in-memory stack, this browser session only) and **Reset to Generated
  Schedule** (clears only the current week's `overrides` doc — verified to leave the
  underlying task/fixed-event data untouched, distinct from the existing full "Reset Planner
  Data" dev tool).
- **Bilingual single-file UI**: verified both languages render correctly for static labels,
  dynamic status/summary text, validation messages, and the JSON schema hint; verified native
  time-input locale text switches correctly (the exact bug V1 hit with `app.en.html`); verified
  user-entered task/event data is never translated; verified language choice persists across
  reloads via `localStorage`.
- PDF export DOM wiring: `#pdf-page1`/`#pdf-page2` rasterize the live `#calendar`/`#task-list`
  elements (same nodes the on-screen view uses, not a separate copy), so a manually-adjusted
  block's new position and the "Adjusted" badge are present in the DOM at export time —
  confirmed by inspecting that DOM after a drag. **Not independently confirmed**: an actual
  end-to-end PDF render, because `html2canvas`/`jsPDF` load from `cdnjs.cloudflare.com` at
  click time and that host was unreachable from the sandboxed test environment used to build
  this (network policy, not app code). Worth a real click-through before relying on it.

### Two known bugs found and fixed during V2 development (not present in V1's behavior, or
inherited unmodified — see below)

- The block-detail popover backdrop had `hidden` set on load but a class rule
  (`.block-popover-backdrop{display:flex}`) that unconditionally overrode the browser's
  default `[hidden]{display:none}`, so the invisible-but-still-flex backdrop covered the whole
  page and silently ate every click, anywhere, from first render. Fixed with an explicit
  `.block-popover-backdrop[hidden]{display:none}` rule (and the same fix for the popover's
  `.kv` detail rows, which had the identical bug). This is a new-in-V2 element, not a V1 bug.
- The `中文 | English` toggle buttons were rendered and styled but had no click handler wired
  up at all — clicking them did nothing. Fixed by adding the two `addEventListener` calls and
  an initial `applyI18n()` call at page load.

### Known limitation inherited unmodified from V1 (not a V2 regression — confirmed present in
V1's own `app.html` too, so out of scope to fix here per the instruction to preserve V1's
foundation and not redesign unrelated parts of the app)

- The "+ Add Fixed Event" / "+ Add Task" / "+ Add Protected Block" forms are always laid out
  and visible (`form.entry-form{display:grid}` similarly overrides `[hidden]`), regardless of
  the `hidden` attribute the toggle button sets — confirmed identically present in V1's
  `app.html`, not something introduced while building V2. Left as-is since fixing it isn't part
  of the requested V2 change set; flagging here so it isn't mistaken for a new gap.

### Not yet implemented / known gaps (see the original request's §17 self-check list — these
are the parts of it not yet exercised)

- `preferences.sacrificePriority` and `locationOptions`/`location`/`people` are still
  display-only, same as V1 — not read by `schedule()`.
- A Must task with `repeatPerWeek > 1` still only reports a shortfall for the first session
  that fails to fit, same known V1 gap (see V1's `acceptance-criteria.md` §A11).
- Touch-device dragging (as opposed to mouse/pointer-emulated) hasn't been tested on an actual
  touch screen — the interaction is built on Pointer Events with `touch-action:none`, which
  should cover it, but this wasn't verified on real hardware.
- Dark theme and phone-width rendering follow the same tokens as V1 but haven't been
  screenshotted/verified with the new V2 elements (lang toggle, popover, drag ghost) in place.
