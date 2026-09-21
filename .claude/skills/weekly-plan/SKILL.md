---
name: weekly-plan
description: Generate a personalized Monday–Sunday weekly schedule from the user's fixed events, tasks, deadlines, estimated durations, and time/location preferences. Produces an interactive "手帳" (planner-style) HTML weekly grid plus a task master list, so must-do work stops crowding out exercise, reading, social time, and free blocks. Use when the user asks to plan their week, build/update their weekly schedule, or says something like "幫我排這週的計畫" / "更新我的週計畫".
---

# Weekly Plan

## Purpose

The user's default failure mode: without deliberate planning, "must-do" work eats the whole
week and exercise / reading / social / rest get silently sacrificed. This skill turns
structured inputs (fixed commitments + tasks with metadata + personal preferences) into a
week that protects the things that lose by default, while still hitting deadlines.

**What this skill decides:** where things go on the calendar — sequencing tasks against fixed
events, fitting short/splittable tasks into small gaps, and packing work efficiently.

**What this skill never decides:** the user's priorities. It never invents a task's
Must/Want/Optional category, never decides which evenings stay free, never decides how many
times a week the user should exercise, and never chooses what gets cut when time is tight.
Those calls come only from the input files below. If a task or preference is missing a
required field, ask the user for it — do not guess a priority value.

## Inputs

Read three YAML files from the user's `data/` directory (create them from the templates in
`templates/` on first run if they don't exist yet):

1. **`data/fixed_events.yaml`** — immovable commitments: time, day(s), location, people.
2. **`data/tasks.yaml`** — the work backlog. Each task carries: name, deadline, estimated
   duration, category (`must` / `want` / `optional`), whether it's splittable, its minimum
   useful time block (the shortest chunk worth opening the task for), preferred time-of-day,
   and acceptable locations.
3. **`data/preferences.yaml`** — the user's own standing decisions: protected free blocks
   (e.g. Friday evening, Sunday morning), weekly personal-activity targets (e.g. "exercise ×3,
   45 min, mornings"), and a sacrifice-priority ordering for when the week is overbooked.

Full field definitions and examples are in `templates/fixed_events.yaml`,
`templates/tasks.yaml`, and `templates/preferences.yaml`. If any file is missing fields the
algorithm needs (e.g. a task with no category or no estimated time), stop and ask — don't
default it silently.

## Algorithm

See `references/algorithm.md` for the full placement algorithm (gap-finding, splitting rules,
tie-breaking, overflow handling). Summary:

1. Lay fixed events on the Mon–Sun grid first — they never move.
2. Lay down `preferences.yaml` protected blocks next — they are treated like fixed events and
   nothing else may be scheduled over them.
3. Place `must` tasks by deadline urgency (earliest deadline first), preferring blocks that
   meet the task's full estimated duration; only split a task across multiple blocks if
   `splittable: true`, and never into a block smaller than `min_block`.
4. Place `want` tasks into remaining slots that match their time-of-day preference, then
   `optional` tasks into whatever is left.
5. Use small leftover gaps (below any unsplit task's `min_block`) for splittable short tasks
   that fit — this is the "零碎時間" pass.
6. Fit weekly personal-activity targets from `preferences.yaml` (exercise, reading, etc.)
   into remaining slots that match their preferred time-of-day, before optional tasks claim
   that space.
7. Whatever remains unscheduled and unclaimed stays a visible **free block** — do not fill it
   just because it's empty.
8. If a `must` task cannot fit before its deadline given everything above, do not silently
   drop it or silently cannibalize a protected block: flag it in the output and tell the user
   directly, listing the shortfall in hours.

## Output

Produce two things in a single interactive HTML artifact (load `artifact-design` and
`artifact-capabilities` skills before building it — this page needs the `db` capability so
completion checkboxes persist across the week instead of resetting every time the plan is
regenerated):

1. **Weekly grid (Mon–Sun), 手帳-style** — a planner-page layout, not a generic calendar UI.
   Time down the side, days across the top (or a day-block/agenda layout if that reads
   cleaner at phone width — see `references/style.md`). Every placed item is visually
   classed into exactly one of four categories, distinguished by color/texture (not color
   alone — add an icon or label so it still reads in grayscale):
   - **Fixed events** (immovable)
   - **Scheduled tasks** (from `tasks.yaml`, sub-tagged Must/Want/Optional)
   - **Personal activities** (from `preferences.yaml` targets — exercise, reading, social...)
   - **Free blocks** (intentionally empty, protected or leftover)
2. **Task master list** — every task from `tasks.yaml` in one table/list: name, category,
   deadline, estimated time, and a completion checkbox. Checking a task off writes to the
   artifact's shared `db` so progress survives regeneration; re-running this skill next week
   must not wipe prior completion state for tasks that still exist. Sort or group by deadline
   so the user can see at a glance what's coming up and what's already done.

If the user already has a published weekly-plan artifact (check for a saved URL, e.g. in
`data/artifact_url.txt`), update that artifact in place with `Artifact({url: ...})` rather
than publishing a new one each week, so the link and the completion history stay stable.
Save the URL back to `data/artifact_url.txt` after the first publish.

## Style detail

See `references/style.md` for the color/category legend and layout notes so the page reads
as a planner, not a dashboard.
