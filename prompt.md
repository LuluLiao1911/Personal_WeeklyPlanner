# Weekly Planner — Version 1 Test

## Purpose

This is the primary fresh-run test for Version 1 of the Weekly Planner (the `weekly-plan`
Claude Artifact app). Two language versions exist and should both be run: the Chinese UI,
published at https://claude.ai/artifact/9KtMmdz6JLccGjGs4GrJFE (source `app.html`), and the
English UI, at https://claude.ai/artifact/XRLUni3kdae8BHirFj62vs (source `app.en.html`).

The goal is to evaluate whether the planner can transform a fixed set of weekly events,
tasks, and scheduling constraints into a feasible and useful weekly schedule.

The planner should complete mandatory (Must) tasks before their deadlines while using
available time efficiently, and it should preserve meaningful continuous free time rather
than filling every gap just because it exists.

---

## Test Input

Use the supplied file matching the version under test:

`testcase.json` (Chinese content, for `app.html`) or `testcase.en.json` (the same testcase,
English content, for `app.en.html`) — both encode the identical scenario, just translated,
so results from either language are comparable.

Each JSON file contains:

- the fixed planning week (`weekStart`),
- all fixed events,
- all flexible tasks,
- task durations (`estimatedMinutes`, `repeatPerWeek`),
- task scheduling constraints (`category`, `deadline`, `splittable`, `minBlockMinutes`,
  `timePreference`, `preferredDay`, `locationOptions`),
- scheduling preferences (`dayStart`/`dayEnd`, `protectedBlocks`, `sacrificePriority`).

Do not modify, reinterpret, add, or remove any testcase values before generating the
schedule. Full field-by-field reference:
`.claude/skills/weekly-plan/references/testcase-schema.md`.

---

## Test Procedure

1. Open the Weekly Planner artifact in a fresh state (Input tab).
2. Open the **Dev / Test Tools** panel at the bottom of the Input tab and click
   **Reset Planner Data** to clear any existing fixed events, tasks, preferences, protected
   blocks, and the current week's reflection notes. Confirm the dialog.
3. Under **Import Test Case JSON**, choose `testcase.json` (or `testcase.en.json` on
   `app.en.html`).
4. Read the confirmation summary the panel shows (planning week, fixed event count, task
   count broken down by Must/Want/Optional, protected block count) and confirm it matches
   the testcase file exactly. If the panel instead shows a validation error, that error *is*
   part of the test result — record it and stop; do not hand-fix the JSON and re-import.
5. Switch to the **This Week** tab. The schedule is computed automatically in the browser the
   moment this tab is opened (or any input changes) — there is no separate "generate" step
   and no manual scheduling action to take.
6. Do not manually rearrange, delete, or re-time any scheduled block after generation.
   (Version 1 has no drag/edit affordance on the calendar itself, so this should already hold
   by construction — note it explicitly if a later version adds one.)
7. Preserve the generated schedule as the actual Version 1 output: full-page screenshot of
   the This Week tab (calendar + summary callout + master task list + free-time suggestions +
   reflection section).
8. Click **Download This Week's PDF** and keep the exported file as part of the test record.
9. Record any warnings, failures, unexpected behavior, or required human intervention —
   see the next section.

---

## Important Testing Rule

The purpose of this test is to observe the actual behavior of Version 1.

Do not manually repair an undesirable schedule before recording it.

If the result contains a scheduling problem, poor use of time, or an output issue, preserve
it as evidence for later evaluation and possible Version 2 improvement. This includes (but
isn't limited to):

- a Must task the summary callout flags as not fully scheduled before its deadline,
- a Want/Optional recurring task that couldn't get all of its weekly sessions placed,
- a task placed outside its stated `timePreference` window,
- any overlap between two blocks, or between a block and a fixed event,
- a task placed after its own `deadline`,
- `locationOptions` or `sacrificePriority` appearing to have no effect on placement — this is
  **known, expected Version 1 behavior** (see "Known Version 1 scope" below), not a bug to
  chase, but still worth confirming it didn't silently change either.

---

## Fresh-Test Requirement

This testcase should later be reused unchanged when testing Version 2.

Version 2 must begin from a reset state and import the same testcase file (matching
language version).

The comparison should therefore follow:

same starting input
→ Version 1 planner
→ Version 1 result

and

same starting input
→ Version 2 planner
→ Version 2 result

This ensures that observed differences are caused by changes to the planner rather than
changes to the testcase. `weekStart` in the testcase pins the planning week regardless of
the real-world date either run happens on, so a V1 run today and a V2 run next month still
render the same Mon–Sun week.

---

## Expected Output

The run should produce:

1. A complete Monday–Sunday weekly calendar (7 day columns, one shared time axis).
2. All fixed events shown at their original times, visually distinct from scheduled tasks
   (solid sage-green blocks).
3. Scheduled flexible tasks, colored/labeled by category (Must / Want / Optional).
4. Clearly identifiable free periods — unclaimed calendar space, plus any protected blocks
   shown as dashed outlines.
5. A master task overview (Master Task List): every task with category, deadline, where it
   landed in the week (or "Not scheduled"), and completion status.
6. Any free-time suggestions generated by the planner (Free-Time Suggestions): leftover gaps
   ≥30 min, labeled with generic activity ideas — these are suggestions only, never written
   back to the calendar.
7. A weekly reflection area (Weekly Reflection): five free-text fields, empty on a fresh
   import.
8. An exportable PDF version of the weekly planner (2 pages: calendar page, then task list /
   free-time suggestions / reflection page).

The actual result should be preserved even if some of these elements do not work correctly
in Version 1.

Score the preserved result against `acceptance-criteria.md` — that file is the pass/fail
rubric this test's output is judged by.

---

## Known Version 1 scope (context for evaluating results, not excuses to wave away a finding)

- `sacrificePriority` and every task's `locationOptions` (plus a fixed event's
  `location`/`people`) are stored, editable, and round-trip through import/export, but are
  **not yet read by the scheduling algorithm** — they don't influence placement. A testcase
  that varies only these fields is expected to produce the same schedule in V1.
- There is no manual drag/edit of a scheduled block — the calendar is fully computed from
  input each time; the only way to change the schedule is to change the input.
- Location is never enforced as a hard constraint (no gap carries an inherited location) —
  see `references/algorithm.md`.
- A Must task with `repeatPerWeek > 1` (both testcases include one) only reports a shortfall
  for the first session that fails to place, not for every session left unattempted after
  it — the summary can understate how infeasible the testcase actually is. See
  `acceptance-criteria.md` §A11.
