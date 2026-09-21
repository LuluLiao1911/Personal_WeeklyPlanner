# Weekly Planner — Version 1 Acceptance Criteria

This is the pass/fail rubric to apply to a run produced by `prompt.md` (import `testcase.json`
→ generate → export PDF). It extends your original list with items this specific
implementation needs — either because a bug already surfaced there once (§A9, §B6), because
a check is only meaningful once you know *how* V1 guarantees it (the "Why / how" column),
or because a whole feature area (Dev/Test Tools) had no criteria yet.

Each item is tagged with where to check it:

- **[code]** — guaranteed by `schedule()`'s structure; a failure here is a real regression,
  not a bad testcase. Spot-check on any run, but it shouldn't need re-deriving from scratch
  every time.
- **[visual]** — read off the calendar/PDF for the specific run.
- **[live]** — must be checked against the published artifact
  (https://claude.ai/artifact/9KtMmdz6JLccGjGs4GrJFE), not a local/offline copy — it depends
  on the `db`/`downloads` capabilities and the cdnjs-hosted PDF libraries, none of which a
  sandboxed test harness can exercise for real.

---

## A. Hard Scheduling Checks

- [ ] **A1.** All fixed events remain at their original day(s) and time. **[code]** — `schedule()`
      never rewrites a `fixedEvents` entry; the days it repeats on come straight from its
      `days[]` field. Verify visually that every declared occurrence appears (a fixed event with
      `days: ["mon","tue","wed","thu","fri"]` should show up 5 times).
- [ ] **A2.** No flexible task overlaps a fixed event. **[code]** — fixed events are subtracted
      from each day's gap list before any task placement runs; a task can only land in what's
      left.
- [ ] **A3.** No two scheduled activities overlap each other. **[code]** — every placement
      subtracts its own span from the gap list before the next placement is chosen.
- [ ] **A4.** All Must tasks receive their required total duration *or* the shortfall is
      visibly reported. **[code, conditional]** — V1 does not force-fit an infeasible Must
      task. If the full duration can't be placed, it reports the exact remaining minutes in the
      warning callout (⚠️) instead of silently under-scheduling it. Check the callout text, not
      just "did the task get a block."
- [ ] **A5.** No Must task's scheduled block(s) end after that task's own `deadline` — checked
      to the minute, not just the calendar date. **[code — regression-sensitive]** A same-day
      deadline (e.g. due 10:00) must not get a block later that same day (e.g. 14:00). This
      exact bug existed in an earlier build and was fixed by clipping each day's usable gaps to
      the deadline's time-of-day, not just its date — worth a dedicated check on any testcase
      with a same-day deadline.
- [ ] **A6.** Non-splittable tasks (`splittable: false`) are placed as one continuous block per
      occurrence. **[code]** — with `repeatPerWeek > 1`, each of the *n* sessions is still its
      own single continuous block; "non-splittable" constrains a session, not the weekly total.
- [ ] **A7.** Splittable tasks that do get split never produce a chunk shorter than
      `minBlockMinutes`. **[code]**
- [ ] **A8.** Nothing is scheduled outside `preferences.dayStart`–`dayEnd`. **[code, with a
      known exception — see A9]** — task and recurring-activity placements are structurally
      bounded, because they only ever land inside gaps derived from the `dayStart`–`dayEnd`
      window.
- [ ] **A9 (new — recommended).** A fixed event or protected block whose own `start`/`end`
      falls outside `dayStart`–`dayEnd` is not silently clipped from view. **[code — known
      gap]** Fixed events and protected blocks are drawn from their own `start`/`end`
      regardless of the day window (they don't go through the gap system at all), while the
      calendar column is CSS-clipped (`overflow: hidden`) to the `dayStart`–`dayEnd` height. A
      07:00 gym class with `dayStart: "07:30"` would currently render above the visible area
      and disappear. Worth testing explicitly with a testcase that has a fixed event or
      protected block right at (or outside) the day-window edges — this is the one item in
      section A that isn't actually guaranteed today.
- [ ] **A10 (new — recommended).** A task marked `completed: true` is excluded from the
      generated schedule (it still appears in the Master Task List, just with no calendar
      block). **[code]**

---

## B. Schedule Quality Checks

- [ ] **B1.** Tasks are not scattered unnecessarily across the week. **[visual, best-effort]**
      — the split-fallback always takes the *largest* remaining eligible gap first, which
      minimizes fragmentation as a heuristic, not a proven-optimal packing. Judge the specific
      run's calendar, don't assume it from the rule alone.
- [ ] **B2.** Short gaps are not used for tasks that need a longer focus block. **[code, as
      declared]** — enforced exactly to the `minBlockMinutes` *you* set per task; the algorithm
      has no independent notion of "this task needs focus" beyond that number, so this check is
      really "does the testcase's `minBlockMinutes` reflect the task honestly," not just "did
      the app respect it" (which A7 already covers).
- [ ] **B3.** Larger continuous free blocks are preserved where reasonably possible.
      **[visual, best-effort]** — same largest-gap-first heuristic as B1.
- [ ] **B4.** Free time is not filled merely because it's available. **[code]** — there is no
      filler pass; `schedule()` only ever places what a task or recurring activity actually
      asked for. An empty gap stays empty by construction, not by a check that could regress.
- [ ] **B5.** The resulting weekly plan appears realistically usable. **[visual, subjective]**
      — a human-judgment item; no mechanical substitute.
- [ ] **B6 (new — recommended).** A task's `timePreference` (morning/afternoon/evening)
      placement actually lands inside that window, not merely in some gap that happens to
      *overlap* it. **[code — regression-sensitive]** A gap spanning both afternoon and evening
      must not front-load an evening-preferred task into its afternoon portion. This was also a
      real bug in an earlier build (fixed via `prefStartWithin()`) — test with a wide gap that
      spans more than one of the three windows.
- [ ] **B7 (new — recommended).** A recurring task (`repeatPerWeek > 1`) has its sessions spread
      across distinct days rather than stacked on one day, whenever enough other eligible days
      exist. **[code]** — each task tracks which days it has already used and prefers an unused
      one first.

---

## C. Output Checks

- [ ] **C1.** The weekly calendar is easy to scan. **[visual, subjective]**
- [ ] **C2.** Fixed events are visually distinguishable from flexible tasks. **[code+visual]**
      — fixed = solid sage green; Must = solid rose; Want = solid mustard; Optional = mustard
      at reduced opacity; protected/free = dashed outline only. Confirm the legend and the
      actual blocks agree.
- [ ] **C3.** Free time can be identified quickly. **[visual]** — unclaimed space is just blank
      calendar; protected time is a dashed outline. Neither is filled or labeled busy.
- [ ] **C4.** Every task in the testcase appears in the Master Task List. **[code]** — the list
      renders every doc in the `tasks` collection unconditionally, whether or not it got a
      calendar slot.
- [ ] **C5.** The Weekly Reflection area is present. **[code]** — five fields, always rendered,
      empty on a fresh import.
- [ ] **C6.** The planner exports successfully to PDF. **[live]** — depends on `html2canvas`
      and `jsPDF` loading from cdnjs and the `downloads` capability; cannot be verified from an
      offline/sandboxed copy.
- [ ] **C7.** The PDF contains no clipped or missing content. **[live, visual]** — open the
      actual exported file; a rasterized (`html2canvas`) export can silently cut off content
      that overflows its source element's height, which is a different failure mode than a
      normal print and won't show up as a JS error.
- [ ] **C8 (new — recommended).** The Master Task List's "Scheduled at" times match exactly
      what's drawn on the calendar for that task. **[code]** — both are derived from the same
      `schedule()` result in the same render pass, so this should hold by construction; still
      worth a spot check since it's the kind of thing a future edit could quietly break.
- [ ] **C9 (new — recommended).** No entry in Free-Time Suggestions overlaps a block actually
      drawn on the calendar. **[code]** — suggestions are read directly from each day's
      post-scheduling leftover gaps, so this should also hold by construction; same spot-check
      rationale as C8.
- [ ] **C10 (new — recommended).** The page reads correctly in both light and dark theme, and
      at phone width (~400px). **[visual — untested]** Built against the Artifact contract's
      theming/responsive tokens but never actually screenshotted in dark mode or a narrow
      viewport with real data — flag this as unverified rather than assumed-working.

---

## D. Dev/Test Tooling Checks (new section — recommended)

Not covered by the original list at all, but this is the mechanism every other check in this
document runs through, so its own correctness is load-bearing:

- [ ] **D1.** Import validates the entire file before writing anything; a single invalid field
      aborts the whole import with no partial load. **[code — verified]**
- [ ] **D2.** A successful import fully replaces prior fixed events/tasks/preferences/
      protected blocks — nothing from a previous run leaks into the new one. **[code —
      verified]**
- [ ] **D3.** `weekStart` correctly pins the displayed week regardless of the real-world date
      the test is run on. **[code — verified]**
- [ ] **D4.** Reset clears fixed events, tasks, preferences, protected blocks, and the
      currently-displayed week's reflection notes, and returns to the live "this week."
      **[code — verified]**
- [ ] **D5 (new).** Export → re-import of the same exported file reproduces an identical
      schedule (round-trip fidelity). **[live]** — export depends on the `downloads`
      capability, so this one needs the live artifact even though import/validation alone can
      be checked offline.

---

## Known gaps this checklist already accounts for

- **A9**: fixed events/protected blocks outside `dayStart`–`dayEnd` can be clipped from view.
  Not yet fixed — call it out if a testcase run reveals it, rather than treating it as a
  surprise.
- `sacrificePriority` and every task's `locationOptions`/a fixed event's `location`/`people`
  are stored and round-trip correctly but don't influence placement yet — see
  `.claude/skills/weekly-plan/references/testcase-schema.md`, §7. Not an acceptance-criteria
  failure by itself; only flag it if a testcase specifically depends on one of those fields
  changing the output.
