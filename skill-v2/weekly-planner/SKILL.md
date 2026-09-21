---
name: weekly-planner
description: Publish or update the user's personal weekly planner web app, Version 2 — a planner (Traditional Chinese and English UI, as two separate artifacts) with fixed events + tasks + preferences input, an auto-computed Monday–Sunday schedule the user can then drag/edit by hand, simplified free-time suggestions, a weekly reflection log, and PDF export that reflects the manually-adjusted plan. Use when the user asks to set up, open, update, or change their weekly planner/scheduler, or says something like "help me plan this week" / "open my weekly plan" / "update the scheduler". This is a new version of the same skill as `weekly-plan` (Version 1) — not a separate tool.
---

# Weekly Planner — Version 2

## Relationship to Version 1

**Version 1 (`../.claude/skills/weekly-plan/`, `app.html`/`app.en.html` at the repo root) is
frozen.** Nothing under this skill ever touches V1's files, database, or docs. V2 lives
entirely under `skill-v2/weekly-planner/` as its own pair of Artifacts with their own
databases — never V1's.

V1 and V2 are treated as two versions of **one skill**, not two skills: same purpose (publish
the user's weekly planner), same testcase JSON schema (see
`references/testcase-schema.md` — unchanged from V1), same core scheduling algorithm, same
two-separate-language-files architecture. What V2 adds is layered on top, not a rewrite:
manual drag/click adjustment of the generated schedule, and a simplified free-time list.

## What this is

Two files in this folder, each a complete, self-serve weekly planner with the same two tabs:

- **`app.html`** — Traditional Chinese UI, published as its own Artifact with its own
  database.
- **`app.en.html`** — English UI, published as its own separate Artifact with its own
  database. Not a language toggle on a shared instance — same relationship as V1's
  `app.html`/`app.en.html` twins, the two never share runtime state.

Both files share the same `I18N`/`tr()` lookup dictionary and `data-i18n` markup internally
(so every UI string lives in one place per language and both files stay easy to keep in sync),
but each file locks `currentLang` to one language permanently (`"zh"` in `app.html`, `"en"` in
`app.en.html`) — there is no runtime switch, and the switching machinery from an earlier draft
of this file (a `中文 | English` toggle button) was removed in favor of this two-file model,
matching V1's own architecture and the standing "every change ships to both language versions"
rule V1 established.

Each file's two tabs:

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

## Two-language architecture

Each file carries its own `I18N = {zh:{...}, en:{...}}` dictionary and `tr(key, vars)` lookup
function (kept as shared internal machinery between the two files even though each only ever
renders one language — it's what lets a new UI string be added to both `zh`/`en` entries in one
place per file and stay easy to keep the two files in sync, the same discipline V1's standing
"every change ships to both language versions" rule already established). `data-i18n`/
`data-i18n-ph` attributes mark up the static HTML; everything generated dynamically calls
`tr()` directly. `currentLang` is a fixed constant per file (`"zh"` in `app.html`, `"en"` in
`app.en.html`), set once at the top of the script — there is no runtime switch and no
`localStorage` dependency.

- User-entered data (task/event names, notes, locations, people) is never touched by any of
  this — only `esc()` ever inserts that text, `tr()` never wraps it.
- Each native `<input type="time">`/`type="datetime-local">` element (including the popover's
  time input) gets its `lang` attribute set from the same fixed `currentLang` in `applyI18n()`
  — this is what fixes the exact bug V1 originally hit (`app.en.html`'s time inputs showing
  Chinese AM/PM text because Chromium reads a native time input's placeholder text from that
  per-element attribute, not the viewer's own browser locale, and not `<html lang>` either).

## Standing rule inherited from V1: nothing user-visible is ever auto-translated

Task names, fixed-event names, locations, people, and notes are exactly what the user typed,
in whichever language they typed it, regardless of which file/language they're viewed in. The
example in the original request — "微算機 Lab 作業" staying exactly as entered even in the
English file — is enforced structurally: `tr()` only ever looks up i18n dictionary keys, and
every call site that renders user data calls `esc()` on the raw string, never `tr()`.

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

- First-time setup: load `artifact-capabilities`, then publish **both** files as separate
  Artifacts — `Artifact({file_path: "app.html", capabilities: {db: {}, downloads: true},
  icon: "calendar"})` and the same for `app.en.html` — from this folder, each getting its own
  URL and its own database, neither shared with V1's.
- Any later change to either file: republish to **that file's own URL** (pass `url`) — same
  standing rule V1 established: a change isn't done until it ships to both `app.html` and
  `app.en.html` and both are republished (see "Standing rule" above, and V1's own SKILL.md for
  the original workflow this mirrors).
- **A page open without a live `db`/`downloads` connection (the raw file opened directly,
  outside the Artifact platform, or an Artifact published without `capabilities`) will look
  like it's silently doing nothing**: `if(!db) return;` guards every write (Add/Edit/Delete,
  dev-tools import/export/reset), so every one of those actions no-ops with no error shown.
  This is the same guard V1 already had — it isn't new to V2 — but it's worth knowing before
  debugging a report like "the Add button doesn't do anything": check first whether the page
  being tested is actually a published Artifact with `db`/`downloads` declared, not a bug in
  the submit handler itself.
- A change to the *scheduling logic itself* should stay identical to V1's unless the user asks
  otherwise (§1 of the original V2 request: preserve the V1 foundation). A change to *manual
  adjustment or free-time suggestions* is V2's own territory — edit freely, but update
  `references/manual-adjustment.md` if the override data model or validation rules change.

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
- **Two-file bilingual UI** (Chinese `app.html` / English `app.en.html`, each its own Artifact
  and database — matching V1's architecture): verified both files render correctly for static
  labels, dynamic status/summary text, validation messages, and the JSON schema hint; verified
  native time-input locale text is correct in each file (the exact bug V1's `app.en.html`
  needed a separate file to fix in the first place); verified user-entered task/event data is
  never translated in either file.
- PDF export DOM wiring: `#pdf-page1`/`#pdf-page2` rasterize the live `#calendar`/`#task-list`
  elements (same nodes the on-screen view uses, not a separate copy), so a manually-adjusted
  block's new position and the "Adjusted" badge are present in the DOM at export time —
  confirmed by inspecting that DOM after a drag. **Not independently confirmed**: an actual
  end-to-end PDF render, because `html2canvas`/`jsPDF` load from `cdnjs.cloudflare.com` at
  click time and that host was unreachable from the sandboxed test environment used to build
  this (network policy, not app code). Worth a real click-through before relying on it.

### Bugs found and fixed during V2 development (not present in V1's behavior, or inherited
unmodified — see below)

- The block-detail popover backdrop had `hidden` set on load but a class rule
  (`.block-popover-backdrop{display:flex}`) that unconditionally overrode the browser's
  default `[hidden]{display:none}`, so the invisible-but-still-flex backdrop covered the whole
  page and silently ate every click, anywhere, from first render. Fixed with an explicit
  `.block-popover-backdrop[hidden]{display:none}` rule (and the same fix for the popover's
  `.kv` detail rows, which had the identical bug). This is a new-in-V2 element, not a V1 bug.
- An earlier draft of this file used a single bilingual artifact with a `中文 | English`
  toggle; that toggle's buttons were rendered and styled but had no click handler wired up at
  all — clicking them did nothing. This was moot once the toggle itself was removed in favor
  of the current two-file architecture, but is recorded here since it was a real bug at the
  time.
- Reported after first publishing: a page opened **without** a live `db`/`downloads`
  connection makes every Add/Edit/Delete button (and dev-tools import/export/reset) look
  broken — they silently no-op (`if(!db) return;`) with no error shown, so "Import failed" and
  "the Add button doesn't do anything" were the same underlying cause, not two bugs. This is
  V1's own existing guard, not something V2 introduced — the fix was publishing both files as
  real Artifacts with `capabilities: {db:{}, downloads:true}` declared (see "Publishing /
  updating" above), not a code change.

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
  screenshotted/verified with the new V2 elements (popover, drag ghost, lock icon, adjusted
  badge) in place.
