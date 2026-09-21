# Manual schedule adjustment (new in V2)

This is the main V2 feature: the algorithm's output becomes an initial recommendation, and the
user can move individual blocks by hand without the whole plan being recomputed. Everything
below lives in `app.html`; there is no server-side component.

## Data model

`overrides/<mondayISOdate>` — one db doc per week, same key format as `reflections/`
(`weekDocKey()`):

```json
{ "moves": { "auto11::0": { "day": "mon", "start": 1185 } } }
```

- The key (`"auto11::0"`) is a **blockKey**: `taskId + "::" + sessionIndex`, assigned inside
  `schedule()`'s `place()` helper via a per-task counter (`sessionCounters`). A task with
  `repeatPerWeek: 2` produces two distinct blockKeys (`::0`, `::1`) across the week — each
  session can be moved independently. Only blocks with a `taskId` (i.e. task blocks, not fixed
  events or protected blocks) get a blockKey and are therefore ever overridable.
- The value is `{day, start}` only — **never a duration**. A block's length is always derived
  from the task (`estimatedMinutes`, or however much of it that session represents), never
  stored in the override, so a moved block can't accidentally change size through the override
  path.
- This is deliberately a *separate* collection from `tasks`/`fixedEvents`/`prefs/main`. Moving
  a block writes only to `overrides`; it never touches the task's own stored fields. This is
  what keeps "what the user needs to do" (task input) and "when the user currently plans to do
  it" (schedule override) distinct, per the design goal in the original request.

## The two-pass merge: `applyOverrides(baseline, overridesMap)`

`schedule()` still computes the exact same baseline it always did — full recompute, no
knowledge of overrides. `applyOverrides()` runs after it, purely at render time:

1. **Pass 1 — reserve everything not being moved.** For every block in the baseline result
   whose `blockKey` is *not* a key in `overridesMap`, place it at its original day/time exactly
   as the algorithm chose. This rebuilds each day's free-gap list from scratch (fixed events +
   protected blocks + every non-overridden task block), the same way `schedule()`'s own
   `place()`/`subtractRange()` machinery works.
2. **Pass 2 — place each overridden block.** For every `blockKey` in `overridesMap`, look up
   its original block (for its label, category, duration, and `taskId`) and try to place it at
   the override's `{day, start}`:
   - Reject (fall back to step 3) if the new `[start, start+duration)` range falls outside
     `[dayStart, dayEnd]`.
   - Reject if it overlaps anything already reserved on that day — a fixed event, a protected
     block, or another task block (including another override that already succeeded earlier
     in this same pass, so two overrides can never silently land on top of each other).
   - Reject if the block's task is `category: "must"` and has a deadline, and the new
     placement's day+end-time would land on or after that deadline.
   - If none of those trip, reserve it at the new position and mark it `manual: true`.
3. **Fallback on rejection.** A rejected override is placed at its **original baseline
   position** instead of being dropped — it's still marked `manual: true` (the user did try to
   move it) but also `overrideFailed: true`. If even that original slot is no longer free
   (only possible if a *different* override was placed into it in pass 2 — the one
   multi-override edge case this design doesn't fully resolve), it's additionally marked
   `conflict: true` and rendered with a warning outline rather than silently overlapping with
   no visual indication. In normal single-move usage this fallback path is never hit — it only
   matters if overrides go stale (e.g. the referenced task's duration changed after the move
   was saved) or in an unlikely multi-move collision.

This design only works because manual placement in this app is intentionally narrow: an
override is never allowed to *displace* another block (§7 of the original request: a move must
not overlap another scheduled task), so pass 2 never needs to cascade or re-solve anything —
it either fits in the gaps pass 1 left behind, or it doesn't.

## Live drag validation (`validateMove`)

Rejecting a bad *override on the next render* (above) is a safety net, not the primary UX. The
primary check happens live, during the drag itself, in `validateMove(blockKey, targetDayKey,
start)`, which runs against `currentSchedule` (the already-on-screen, override-applied result)
so the user sees a rejection *before* releasing the mouse, with a specific reason:

| Situation | i18n key | Example message |
|---|---|---|
| Overlaps a fixed event | `drag.overlapFixed` | "That overlaps the fixed event "X" — can't drop there." |
| Overlaps a protected block | `drag.overlapProtected` | "That's the protected block "X" — can't drop there." |
| Overlaps another task block | `drag.overlapTask` | ""X" is already scheduled there — can't drop there." |
| Outside `[dayStart, dayEnd]` | `drag.outOfBounds` | "That's outside the day's schedulable time range." |
| Must task would miss its deadline | `drag.deadlineViolation` | "…moving it here would push it past its deadline ({deadline}). Move cancelled." |
| Same day/time as before | `drag.noChange` | (no toast shown — treated as a no-op, not an error) |

A live drag never silently accepts an invalid drop: the drop-ghost preview turns from the
normal sage outline to a warning-red outline (`.drop-ghost.invalid`) the moment the candidate
position fails validation, and on release the block animates back to its last valid position
(nothing is committed) with a toast naming the reason.

## Snapping

Every candidate position — from a drag or from the popover's time input — is snapped to the
nearest 15-minute increment (`snapMinutes()`, `Math.round(m/15)*15`) before validation ever
runs, so a block can only ever land on `:00/:15/:30/:45`.

## Click vs. drag

`wireBlockInteraction()` distinguishes a click from a drag by total pointer movement: fewer
than ~4px of movement from `pointerdown` to `pointerup` counts as a click and opens
`openBlockPopover()`; more than that starts the drag-ghost/validation flow instead. Only
`movable` blocks (task blocks) get this handler at all — fixed events and protected blocks are
rendered with a `.locked` class (and a small lock glyph) and never get pointer handlers, so
they're inert to both click and drag.

## The block-detail popover

Shows name, category, current scheduled day/time, duration, deadline (if any), and whether the
current placement is system-scheduled or manually adjusted (`popover.originAuto` /
`popover.originManual`). For a movable block it also offers a day-select + time-input pair as
an alternative to dragging; saving from there calls the exact same `commitMove()` →
`validateMove()` path a drag does, so the two input methods can never disagree about what's a
valid placement. The popover's editor is hidden (not just disabled) for a locked block, since
those can't be moved either way.

## Undo / reset

- **Undo Last Move** (`undoStack`, in-memory, this browser tab only — not persisted): every
  successful `commitMove()` pushes `{weekKey, blockKey, prevValue}` (the override's value
  *before* this move, or `undefined` if the block had no prior override). Undo pops the stack
  and either deletes the override key or restores the previous value, then re-saves and
  re-renders. This is a plain stack, not a full history browser — "undo" only ever means "undo
  the most recent move."
- **Reset to Generated Schedule** clears `state.overrides` and the week's `overrides/<key>` db
  doc entirely, then re-renders — every block returns to the algorithm's baseline position.
  This is deliberately a different, narrower action than the existing "Reset Planner Data" dev
  tool: it never touches `fixedEvents`/`tasks`/`prefs/main`, only the schedule placement.
  Verified (via a Playwright test against a mock db) that a task's own data survives a schedule
  reset untouched.
- Both buttons are disabled when there's nothing to undo/reset (`updateManualButtons()`,
  called after every load/commit/undo/reset).

## What does *not* trigger a full recompute

Per §10 of the original request, only a change to `fixedEvents`/`tasks`/`prefs/main` reruns
`schedule()` (via the same `onSnapshot`-driven `renderOutput()` V1 already had). A drag, a
popover edit, an undo, or a reset all go through `applyOverrides()` against the *same* baseline
— they never call `schedule()` again. Switching weeks does call `schedule()` again (a different
week is a different baseline by definition), and loads that week's own `overrides` doc
separately.
