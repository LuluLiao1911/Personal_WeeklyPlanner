# Personal Weekly Planner

A Claude Code skill that builds a personalized Monday–Sunday schedule from your fixed
commitments, task backlog, and preferences — so that "一定要完成的事情" stops silently
crowding out exercise, reading, social time, and rest.

## The problem this solves

Without deliberate planning, most available time goes to must-do work by default, and
everything else (movement, reading, people, unstructured time) loses. This tool asks you to
externalize the inputs once — what's fixed, what's on your plate, what you value — and then
does the mechanical part: fitting things onto the calendar, using small gaps for short tasks,
and leaving real free blocks intact instead of letting them get eaten by "just one more task."

## What it decides vs. what you decide

| | |
|---|---|
| **The skill decides** | Where things go on the calendar: sequencing against fixed events, splitting/packing tasks into gaps, using fragments of time efficiently. |
| **You decide** | Whether a task is Must / Want / Optional. Which evenings stay protected. How many times a week you want to exercise. What's worth sacrificing when the week is overbooked. |

The skill never invents a priority or a sacrifice — it only acts on what you put in
`data/preferences.yaml` and `data/tasks.yaml`, and it tells you explicitly whenever something
didn't fit rather than quietly dropping it.

## Setup

1. Fill in your real data (the files already exist under `data/`, seeded from the templates —
   edit them directly, or copy fresh from `.claude/skills/weekly-plan/templates/` to start
   over):
   - `data/fixed_events.yaml` — immovable commitments (class, work, commute, recurring
     meetings).
   - `data/tasks.yaml` — your task backlog: deadline, estimated time, category
     (must/want/optional), splittable or not, minimum useful block, time-of-day preference,
     location options.
   - `data/preferences.yaml` — protected free blocks, weekly personal-activity targets
     (exercise/reading/social/...), and your own sacrifice-priority ordering.
2. In Claude Code, run the `weekly-plan` skill (e.g. "幫我排這週的計畫" / "update my weekly
   plan"). It reads the three files above and publishes/updates an interactive weekly
   planner.

## Output

A single interactive, 手帳-style HTML page with:

- **Weekly grid (Mon–Sun)** — every item classed as a Fixed event, Scheduled task
  (Must/Want/Optional), Personal activity, or Free block, each visually distinct.
- **Task master list** — every task with its deadline and a completion checkbox, so you can
  see at a glance what's done and what's still due. Checking things off persists across
  re-runs — regenerating next week's plan doesn't reset your progress.

The published link is saved to `data/artifact_url.txt` after the first run so later runs
update the same page instead of creating a new one each week.

## Example

A worked example, built from `.claude/skills/weekly-plan/templates/` sample data (a Monday
with two classes, a team meeting, a report due Saturday, and three weekly personal-activity
targets), is live at:

**https://claude.ai/artifact/9KtMmdz6JLccGjGs4GrJFE**

The static source behind that page is saved at `examples/sample-week.html` for reference —
opening it locally renders the same weekly grid and task list, minus the live checkbox
persistence (that needs the published artifact's database).

## Project layout

```
data/                                  your real inputs (start here)
.claude/skills/weekly-plan/
  SKILL.md                             what the skill does and its rules
  references/algorithm.md              the placement algorithm in detail
  references/style.md                  output styling notes
  templates/                           blank/example versions of the data files
examples/                              a sample week built from example data
```
