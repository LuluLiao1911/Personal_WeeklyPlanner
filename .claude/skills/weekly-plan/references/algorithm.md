# Placement algorithm (as implemented in `app.html`'s `schedule()`)

Operates on one Monday–Sunday week (any week — `weekOffset` lets the page look ahead/back),
in minutes-of-day, per day, between `prefs.dayStart` and `prefs.dayEnd`.

## 1. Immovable layer

In order, subtracted from each day's gap list, never revisited:

1. `fixedEvents` occurring on that weekday.
2. `prefs.protectedBlocks` occurring on that weekday.

## 2. Tasks, in priority order

Tasks are sorted `must` → `want` → `optional`, then by deadline ascending within a category
(no-deadline tasks sort last in their category — they're placed opportunistically, not raced
against a clock).

Each task carries `repeatPerWeek` (default 1). A recurring personal activity (exercise,
reading…) is just a Want/Optional task with no deadline and `repeatPerWeek > 1` — there is no
separate "personal activity" data type. Each of the `repeatPerWeek` sessions is placed
independently, preferring a weekday not already used by an earlier session of the *same* task
so repeats spread across the week instead of stacking on one day.

For each session:

1. **Eligible days** = days on/before the task's deadline (if any) and matching
   `preferredDay` (if set).
2. **Deadline clipping**: on the exact calendar day the deadline falls on, only the portion of
   that day *before* the deadline's time-of-day is usable — not the whole day. (This is what
   stops a task due at 10:00 from being scheduled at 14:00 the same day.)
3. **Whole-gap search**: look for a single gap ≥ the session's duration. Two passes — first
   requiring the gap to overlap the task's `timePreference` window (morning `dayStart–12:00`,
   afternoon `12:00–18:00`, evening `18:00–dayEnd`), then without that requirement. When a
   preference match is found, the block is placed at the intersection of the gap and the
   preferred window (not just at the gap's start) — a gap spanning afternoon *and* evening
   still lands the block in the evening portion.
4. **Split fallback**: if no single gap fits and the task is `splittable`, repeatedly take the
   *largest* remaining eligible gap (≥ `minBlockMinutes`) until the full duration is placed or
   gaps run out. (The split fallback does not re-check `timePreference` — by the time a task
   needs splitting, fitting it at all takes priority over where.)
5. **Shortfall**: if a `must` task's session doesn't fully fit, or a recurring Want/Optional
   task can't get all its weekly sessions in, that's recorded and shown in the summary
   callout — never silently dropped, never silently placed past its own deadline.

## 3. Free time

Whatever remains in `day.gaps` after every task pass **is** the free time — nothing fills it
just because it's empty. Those leftover gaps (≥ 30 min) are also what populates the
Free-Time Suggestions list, bucketed by length into short/medium/long
activity ideas — suggestions only, never written to the calendar itself.

## Location

`locationOptions` and `people` are informational (shown in the task/event detail) rather than
a hard placement filter — the client-side engine does not model "this gap is at the gym."
Treat this as a known simplification if picking it back up: a stricter version would need each
gap to carry an inherited location from its surrounding fixed events.
