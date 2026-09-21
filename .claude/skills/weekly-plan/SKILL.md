---
name: weekly-plan
description: Publish or update the user's personal weekly planner web app (fixed events + tasks + preferences input, an auto-computed Monday–Sunday schedule, free-time suggestions, a weekly reflection log, and PDF export). Use when the user asks to set up, open, update, or change their weekly planner/scheduler, or says something like "help me plan this week" / "open my weekly plan" / "update the scheduler".
---

# Weekly Plan

## What this is

`app.html` in this repo is a complete, self-serve weekly planner: a single-page app with
two tabs.

1. **Input tab** — forms to add/edit/delete fixed events, tasks, and preferences.
   No YAML, no re-running a skill to see a new week — data is entered once and lives in the
   artifact's own database.
2. **This Week tab** — a paper-planner-style Monday–Sunday calendar computed **live in the
   browser** from that data, plus a task master list, free-time suggestions, a weekly
   reflection log, and a PDF export button. Recomputes instantly whenever data changes; no
   Claude round-trip needed to see next week or an updated plan.

The scheduling algorithm runs as JavaScript inside the published page (see `schedule()` in
`app.html`) — it is not something Claude re-derives per request. Claude's job is to
**publish or update this file as an Artifact**, and to modify the JS/HTML when the user wants
the tool itself changed (new field, different rule, different look).

`app.en.html` is an English-UI twin of `app.html`, published as its own separate Artifact
with its own database (not a language toggle on a shared instance — the two never share
runtime state).

## Standing rule: every change ships to both language versions

**From this point on, no change to this app is done until both `app.html` and
`app.en.html` carry it.** This applies to every kind of change — a scheduling-logic fix, a
new field, a new section, a CSS/layout tweak, a bug fix found while testing one version —
not just user-visible features. The two files have no shared runtime, so nothing propagates
automatically; skipping one half silently reintroduces the exact bug or gap the other half
just fixed.

Workflow for any change:

1. Implement and verify the change in one file (either one — `app.html` is usually the
   reference copy since it's the original).
2. Port the identical change to the other file: same logic/markup/CSS edits, with only the
   already-established English translations applied to any new or changed user-facing
   string (extend the translation approach already used for the rest of the file — see the
   git history for `app.en.html`'s initial translation for the mapping style/tone to match).
3. Republish **both** Artifacts (pass each file's own `url` so both links and both
   databases keep working) before considering the change done.
4. If the change affects the data model (a new field, a renamed field, a new collection),
   update `references/testcase-schema.md`, `testcase.json`/`testcase-example.json`, and
   `acceptance-criteria.md` too — both language versions read the same JSON test-case shape,
   so the schema doc and fixtures are shared, not duplicated per language.

A change that only touches one file is not finished, even if that's the file the user
happened to be looking at when they asked for it.

## What it decides vs. what the user decides

The scheduler places things on the calendar: it sequences tasks against fixed events, splits
and packs tasks into gaps, and fills small fragments with short splittable tasks. It never
invents a task's Must/Want/Optional category, never decides which evenings stay protected,
never decides how many times a week to exercise, and never silently drops a Must task that
doesn't fit — it surfaces a shortfall instead. Those calls come only from what the user enters
in the Input tab.

## Dev/test tools (not the normal user flow)

A collapsed "Dev / Test Tools" panel at the bottom of the Input tab lets a developer import a
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
  listed, as suggestions, in the "Free-Time Suggestions" section.

## Implementation status

### Completed

- **Input tab**: full CRUD forms for fixed events, tasks (category, optional deadline,
  duration, weekly repeat count, splittable, minimum block, time-of-day preference,
  preferred day, location text, notes), and preferences (daily schedulable window, protected
  blocks, sacrifice-priority list).
- **This Week tab**: live in-browser calendar (7 day columns + shared time axis), task master
  list with computed "scheduled at" times and status, free-time suggestions, a per-week
  reflection log (autosaved), and week navigation (prev/this/next) independent of real-world
  "today."
- **Scheduling algorithm** (`schedule()`): Must → Want → Optional priority with deadline
  ordering; deadline clipping to the exact date *and* time; whole-gap placement with a
  time-of-day–aware pass that lands inside the actual preferred window; a split fallback that
  respects `minBlockMinutes`; recurring tasks (`repeatPerWeek > 1`) spread across distinct
  days; visible shortfall reporting instead of silent drops or overbooking.
- **PDF export**: `html2canvas` + `jsPDF`, loaded from cdnjs at click time, saved via the
  `downloads` capability.
- **Dev / Test Tools**: JSON import with full up-front validation and atomic replace (no
  partial loads), export, reset, and `weekStart` week-pinning for reproducible cross-version
  testing.
- **Testing artifacts** (repo root): `prompt.md` (fresh-run test procedure),
  `testcase.json` (its input), `acceptance-criteria.md` (the pass/fail rubric); schema
  reference at `references/testcase-schema.md`.

### Not yet implemented / known gaps

- `preferences.sacrificePriority` is stored, editable, and round-trips through import/export,
  but is **not read by `schedule()`** — it has no effect on placement yet.
- A task's `locationOptions` and a fixed event's `location`/`people` are display-only —
  location is never enforced as a placement constraint (no gap carries an inherited
  location).
- No manual drag/edit/override of a scheduled block — the calendar is fully recomputed from
  input every time; the only way to change the schedule is to change the input.
- A fixed event or protected block whose own `start`/`end` falls outside
  `preferences.dayStart`–`dayEnd` can render clipped out of view (it doesn't go through the
  gap system that the day-window bounding relies on). See `acceptance-criteria.md` §A9.
- A Must task with `repeatPerWeek > 1` only reports a shortfall for the first session that
  fails to fit, not for every session that never got attempted after it — understates how
  infeasible the testcase actually is. See `acceptance-criteria.md` §A11.
- Dark theme and phone-width (~400px) rendering follow the Artifact contract's tokens but
  haven't actually been screenshotted/verified with real data.
