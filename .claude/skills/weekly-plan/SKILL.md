---
name: weekly-plan
description: Publish or update the user's personal weekly planner web app (fixed events + tasks + preferences input, an auto-computed Monday–Sunday schedule, free-time suggestions, a weekly reflection log, and PDF export). Use when the user asks to set up, open, update, or change their weekly planner/scheduler, or says something like "幫我排這週的計畫" / "打開我的週計畫" / "更新排程工具".
---

# Weekly Plan

## What this is

`app.html` in this repo is a complete, self-serve weekly planner: a single-page app with
two tabs.

1. **輸入資料 (Input)** — forms to add/edit/delete fixed events, tasks, and preferences.
   No YAML, no re-running a skill to see a new week — data is entered once and lives in the
   artifact's own database.
2. **本週計畫 (This week)** — a 手帳-style Monday–Sunday calendar computed **live in the
   browser** from that data, plus a task master list, free-time suggestions, a weekly
   reflection log, and a PDF export button. Recomputes instantly whenever data changes; no
   Claude round-trip needed to see next week or an updated plan.

The scheduling algorithm runs as JavaScript inside the published page (see `schedule()` in
`app.html`) — it is not something Claude re-derives per request. Claude's job is to
**publish or update this file as an Artifact**, and to modify the JS/HTML when the user wants
the tool itself changed (new field, different rule, different look).

## What it decides vs. what the user decides

The scheduler places things on the calendar: it sequences tasks against fixed events, splits
and packs tasks into gaps, and fills small fragments with short splittable tasks. It never
invents a task's Must/Want/Optional category, never decides which evenings stay protected,
never decides how many times a week to exercise, and never silently drops a Must task that
doesn't fit — it surfaces a shortfall instead. Those calls come only from what the user enters
in the Input tab.

## Dev/test tools (not the normal user flow)

A collapsed "開發 / 測試工具" panel at the bottom of the Input tab lets a developer import a
JSON testcase (fixed events + tasks + preferences + protected blocks + a pinned
`weekStart`), export the current input as one, or reset everything. It exists so the same
input can be replayed against different versions of this planner and produce comparable
output — it is strictly additive: it writes into the exact same `fixedEvents`/`tasks`/
`prefs/main` collections and fields the form UI does, and runs through the same `schedule()`
call, never a separate code path. See `references/testcase-schema.md` for the full field
reference (types, required/optional, defaults, exactly which HTML field and internal state
each JSON field maps to) and `references/testcase-example.json` for a ready-to-import example.
Import validates everything up front and loads nothing at all if any field is invalid.

## Data model (stored in the artifact's `db` capability)

- **`fixedEvents` collection** — immovable: `name, days[], start, end, location, people`.
- **`tasks` collection** — the whole backlog, must/want/optional alike, including recurring
  personal activities (exercise, reading…) modeled as a Want/Optional task with no deadline
  and `repeatPerWeek > 1`. Fields: `name, category, deadline (optional), estimatedMinutes,
  repeatPerWeek, splittable, minBlockMinutes, timePreference, preferredDay, locationOptions,
  notes, completed`.
- **`prefs/main` doc** — `dayStart, dayEnd, protectedBlocks[], sacrificePriority[]`.
- **`reflections/<mondayISOdate>` docs** — one per week: `wins, unfinished, longer, change,
  notes`, autosaved from the reflection textareas.

## Publishing / updating

- First-time setup: load `artifact-capabilities`, then `Artifact({file_path: "app.html",
  capabilities: {db: {}, downloads: {}}, icon: "calendar"})`. Seed the three collections with
  a small worked example via `ArtifactData` (batch `set`) so the page opens in a realistic
  working state instead of empty — see the app's own in-app forms for the exact field shapes.
- Any later change to `app.html` in this repo: republish to the **same URL** (pass `url`) so
  the link and the user's stored data keep working.
- If the user asks to change how scheduling works (e.g. a new task field, a different
  fragment-time rule, a different free-time suggestion heuristic), edit the `schedule()`
  function and the matching form fields directly in `app.html`, then republish. See
  `references/algorithm.md` for the placement rules the current implementation follows, and
  `references/style.md` for the visual language (a botanical planner aesthetic, not a generic
  dashboard).

## Guardrails already built into `schedule()`

- Fixed events and protected blocks are placed first and are never touched again — nothing
  else can be scheduled over them.
- A task's eligible time window is clipped to its exact deadline (date **and** time), not
  just the calendar date, so nothing is ever silently placed after its own deadline.
- A Must task (or a recurring Want/Optional target) that cannot fully fit produces a visible
  shortfall in the summary callout — it is never quietly dropped or overbooked.
- Preference-matched placement (`timePreference: morning/afternoon/evening`) lands inside the
  actual preferred window, not just anywhere in a gap that merely overlaps it.
- Free time is never auto-filled — leftover gaps stay blank on the calendar and are only
  listed, as suggestions, in the "自由時段建議" section.
