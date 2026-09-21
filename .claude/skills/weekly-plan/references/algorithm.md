# Placement algorithm

Work on a single Monday–Sunday timeline, in minutes, per day. Assume a wake/sleep window if
`preferences.yaml` sets one (`day_start` / `day_end`); otherwise ask the user once and treat
it as a standing preference worth saving back to the file.

## 1. Immovable layer

Place, in this order, refusing any overlap:

1. `fixed_events.yaml` entries.
2. `preferences.yaml` protected blocks.

If two immovable items overlap each other (a data error, not a scheduling decision), stop and
tell the user — do not silently pick a winner.

## 2. Gap list

After step 1, compute the list of free gaps per day: `(day, start, end, duration)`.

## 3. Must tasks — deadline-first

Sort `must` tasks by deadline ascending, then by estimated duration descending (bigger tasks
get first pick of the roomier gaps while there's still room).

For each task:
- Look for a single gap ≥ estimated duration, on or before the deadline, matching
  `time_preference` if given. Prefer the earliest such gap (finishing early leaves slack for
  the next must task).
- If none exists and `splittable: true`, split across the largest available gaps (each chunk
  ≥ `min_block`) until the full duration is placed or gaps run out.
- If the task still doesn't fully fit before its deadline, place as much as fits and record
  the shortfall (task name, hours short, deadline) for the end-of-run warning. Do not steal
  a protected block or bump an already-placed earlier-deadline must task to make room.

Re-run the gap list after every placement.

## 4. Want tasks

Sort by `time_preference` match quality, then by deadline (if set) ascending. Same
single-gap-then-split logic as must tasks, but a want task that doesn't fit is simply pushed
to next week's backlog (no warning needed) — leave it unscheduled, don't force it in.

## 5. Personal-activity targets

From `preferences.yaml`, take each `weekly_targets` entry (e.g. exercise ×3 / 45 min /
mornings). Place into remaining gaps matching its preferred time-of-day, spread across
distinct days rather than clustered, before optional tasks get a chance to claim that space.
If a target can't hit its full count, place as many sessions as fit and note the shortfall
alongside must-task shortfalls (same visibility — this is exactly the thing the user is
trying not to lose).

## 6. Optional tasks

Same as want tasks, lowest priority. Fine to leave most of the backlog unscheduled — optional
tasks existing in the plan mostly as "if a gap happens to be free."

## 7. Fragment pass

For every remaining gap smaller than the smallest unplaced non-splittable task, check
splittable want/optional tasks whose `min_block` fits the gap. This is the "零碎時間" pass —
it should only ever consume gaps that nothing else could have used.

## 8. Free blocks

Everything still unclaimed after step 7 is a free block. Never fill it. A week with visible
free blocks is the point of this tool, not a failure of the algorithm.

## Location filtering

At every placement step, only consider a gap valid for a task if the gap's implied location
(inherited from the surrounding fixed events, or "anywhere" if unconstrained) is in the
task's `location_options`. If a task has no reachable gap because of location constraints
alone, say so specifically ("no gap at [允許地點] before deadline") rather than folding it
into a generic shortfall.
