# Placement algorithm (as implemented in `app.html`'s `schedule()`)

**Placement logic unchanged from Version 1** — same rules, same guardrails, same gap
selection, same task ordering. See V1's
`.claude/skills/weekly-plan/references/algorithm.md` for the full original writeup; it is
reproduced here (with two additions — §4 below, and the shortfall-counting fix in §2.5) so this
folder's docs are self-contained. The one behavioral difference from V1 is *diagnostic, not
placement*: how a Must task's shortfall is counted and reported when `repeatPerWeek > 1` (see
§2.5) — V1 has a known under-reporting gap here (its own `acceptance-criteria.md` §A11); V2
fixes the reporting only, in `app.html`/`app.en.html`, without touching V1.

Operates on one Monday–Sunday week, in minutes-of-day, per day, between `prefs.dayStart` and
`prefs.dayEnd`.

## 1. Immovable layer

In order, subtracted from each day's gap list, never revisited:

1. `fixedEvents` occurring on that weekday.
2. `prefs.protectedBlocks` occurring on that weekday.

## 2. Tasks, in priority order

Tasks are sorted `must` → `want` → `optional`, then by deadline ascending within a category
(no-deadline tasks sort last in their category).

Each task carries `repeatPerWeek` (default 1); each session is placed independently, preferring
a weekday not already used by an earlier session of the same task.

For each session:

1. **Eligible days** = on/before the task's deadline (if any) and matching `preferredDay` (if
   set).
2. **Deadline clipping**: on the exact calendar day the deadline falls on, only the portion of
   that day before the deadline's time-of-day is usable.
3. **Whole-gap search**: a single gap ≥ the session's duration, first requiring a
   `timePreference` match, then without.
4. **Split fallback**: if `splittable`, repeatedly take the largest remaining eligible gap (≥
   `minBlockMinutes`) until placed or gaps run out.
5. **Shortfall**: a Must session that doesn't fully fit, or a recurring task that can't get all
   its sessions in, is recorded and shown in the summary — never silently dropped or placed
   past its deadline. For a Must task with `repeatPerWeek > 1`, the per-task session loop
   `break`s on the first session that fails to fully place (unchanged from V1 — placement
   itself doesn't keep trying once capacity is exhausted), but the shortfall recorded at that
   point accounts for the *entire* remaining workload, not just that one session:
   `unscheduled = repeat - placedSessions` (every session from the failing one through the end
   of `repeat`, including ones the `break` meant were never even attempted) and
   `shortfallMinutes = remaining + sessionMinutes*(unscheduled-1)` (that session's own leftover
   — which may be a partial amount if it was splittable and got partway placed — plus one full
   session's worth for each later session that was never attempted). This is what fixes the
   known V1 gap where a `repeatPerWeek: 3` Must task missing 2 of its 3 sessions was reported
   as only a single session's shortfall.

## 3. Free time

Whatever remains in `day.gaps` after every task pass is the free time. V2 changes how those
leftover gaps are *presented* (see "Free-Time Suggestions" below) but not how they're computed
— the gap list itself is identical to V1's.

## 4. New in V2: `blockKey` assignment (still inside `schedule()`, still baseline-only)

Every call to `place()` that carries a `taskId` (i.e. every task block, never a fixed event or
protected block) is also assigned a `blockKey = taskId + "::" + sessionIndex`, via a per-task
counter reset at the start of each `schedule()` call. This is the only change to `schedule()`
itself in V2 — it does not affect what gets placed or where, only tags each placed block with a
stable handle that the manual-adjustment layer (`applyOverrides()`, see
`manual-adjustment.md`) uses to know which specific session a saved override refers to. Two
consequences worth knowing:

- Session numbering is positional, not identity-based: if a task's `repeatPerWeek` changes from
  3 to 2 between one render and the next, `::2`'s override (if any) becomes stale — it simply
  won't match any block in the new baseline and is ignored (see `manual-adjustment.md`'s
  "stale override" handling), not applied to the wrong session.
- `blockKey` is recomputed fresh on every `schedule()` call; it is never stored alongside the
  task itself.

## Free-Time Suggestions — simplified in V2

V1 listed every gap ≥ 30 minutes across the whole week, one row each — for a lightly-scheduled
week this could mean a dozen-plus rows, several of them near-duplicates of the same open
afternoon. V2 caps this at **one suggestion per day** (that day's single largest open block ≥
30 min), so a light week produces at most seven concise lines instead of an exhaustive gap
listing, matching the "concise, actionable" requirement from the original request. Two more
changes:

- A block ≥ 3 hours is described by day-part ("Saturday afternoon") rather than its exact clock
  range, since spelling out "08:00–22:00" for an entirely open day reads worse than naming the
  part of day. Anything shorter still shows the exact `HH:MM–HH:MM` range, since that precision
  is actually useful for a smaller window.
- The three-tier activity idea (`freetime.ideaShort`/`ideaMedium`/`ideaLarge`, bucketed by
  duration) is unchanged from V1.

Free time itself is still never auto-filled or written to the calendar — this is a suggestions
list only, computed from the same post-schedule `day.gaps`, same as V1.

## Location

Unchanged from V1: `locationOptions`/`location`/`people` remain display-only, not a placement
constraint.
