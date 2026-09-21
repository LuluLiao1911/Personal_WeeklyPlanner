# Personal Weekly Planner

A self-serve weekly planner app: enter fixed commitments, tasks, and preferences once through
a form, and get a Monday–Sunday schedule that protects exercise, reading, social time, and
real free blocks instead of letting "一定要完成的事情" quietly eat the whole week.

**Live app:** https://claude.ai/artifact/9KtMmdz6JLccGjGs4GrJFE

## How it works

`app.html` is a single-page app with two tabs, published as a Claude Artifact:

1. **輸入資料 (Input)** — add/edit/delete fixed events, tasks, and preferences through
   forms. No files to edit by hand.
2. **本週計畫 (This week)** — a 手帳-style weekly calendar, computed live in your browser the
   moment you open the tab or change any data. Includes:
   - the 7-day grid (fixed events / Must / Want / Optional tasks / protected & free time,
     each visually distinct),
   - a task master list (category, deadline, where it landed, status, completion checkbox),
   - a free-time suggestions list (meaningful leftover gaps, with activity ideas — suggestions
     only, never auto-booked),
   - a weekly reflection log (wins / unfinished / underestimated / what to change / notes,
     autosaved per week),
   - a "下載本週 PDF" button that exports the current week as a 2-page PDF.

Everything persists in the artifact's own database, so closing the tab or coming back next
week doesn't lose anything — completed tasks stay completed, reflections stay per-week.

## What it decides vs. what you decide

| | |
|---|---|
| **The app decides** | Where things go on the calendar: sequencing against fixed events, splitting/packing tasks into gaps, using small fragments of time efficiently, and telling you plainly when something won't fit. |
| **You decide** | Whether a task is Must / Want / Optional. Which time blocks stay protected. How many times a week you want to exercise (a recurring Want/Optional task with no deadline and a weekly repeat count). What's worth sacrificing when the week is overbooked. |

The scheduler never invents a priority and never silently drops a Must task or overbooks a
protected block — it surfaces a shortfall in the summary callout instead.

## Repo layout

```
app.html                                the whole app (single file, published as the Artifact above)
.claude/skills/weekly-plan/
  SKILL.md                              what the skill does and its rules
  references/algorithm.md               the placement algorithm as actually implemented
  references/style.md                   visual design notes
```

## Dev/test tools

The Input tab has a collapsed "開發 / 測試工具" panel at the bottom — not the normal way to
use the planner, but useful when changing the scheduling logic: import a JSON testcase (fixed
events, tasks, preferences, protected blocks, and a pinned week) to replace all current input
in one shot, export the current input back out as JSON, or reset everything to a blank state.
Import validates the whole file before loading anything — a bad field aborts with a specific
error, never a partial load. Full field reference:
`.claude/skills/weekly-plan/references/testcase-schema.md`; a ready-to-import example:
`.claude/skills/weekly-plan/references/testcase-example.json`.

## Making changes

Edit `app.html` directly (the scheduling logic is in the `schedule()` function; the input
forms and their fields are plain HTML above it) and ask Claude to republish it to the same
Artifact URL so your existing data and the link both keep working.
