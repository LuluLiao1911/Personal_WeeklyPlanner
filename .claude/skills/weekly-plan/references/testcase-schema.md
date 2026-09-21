# Testcase JSON schema (Version 1)

This documents the JSON format accepted by the **Dev / Test Tools → Import Test Case JSON**
feature in `app.html` (the `validateTestCase()` function). It exists purely to make dev
testing reproducible — the normal user workflow is still the form UI. The schema is not a new
data model: every field here is the exact field `schedule()` and the form UI already read and
write. Import writes into the same `fixedEvents`/`tasks`/`prefs/main` db collections the form
writes into, and the calendar is produced by the same `schedule()` call either way — nothing
branches on where the data came from.

## 1. Top-level structure

```json
{
  "testCaseName": "Main Weekly Planner Test",
  "weekStart": "2026-09-28",
  "preferences": {
    "dayStart": "08:00",
    "dayEnd": "22:00",
    "sacrificePriority": ["optional tasks", "reading"]
  },
  "protectedBlocks": [
    { "label": "Friday night off", "days": ["fri"], "start": "19:00", "end": "23:00" }
  ],
  "fixedEvents": [
    { "name": "Operating Systems Class", "days": ["mon"], "start": "09:00", "end": "12:00", "location": "University", "people": "" }
  ],
  "tasks": [
    {
      "name": "OS Assignment", "category": "must", "deadline": "2026-10-01T23:59",
      "estimatedMinutes": 240, "repeatPerWeek": 1, "splittable": true, "minBlockMinutes": 60,
      "timePreference": "morning", "preferredDay": null,
      "locationOptions": ["Library", "Home"], "notes": "", "completed": false
    }
  ]
}
```

## 2–3. Field reference (type, required, allowed values, default)

### Top level

| Field | Type | Required | Notes / default |
|---|---|---|---|
| `testCaseName` | string | no | Label only, shown in the import confirmation. Default `""`. |
| `weekStart` | string `"YYYY-MM-DD"` | **yes** | The only truly required field. Doesn't need to fall on a Monday — the planner locks to the Monday–Sunday week *containing* this date (`getWeekDates()`), same as every other week calculation in the app. |
| `preferences` | object | no | Default `{dayStart:"07:30", dayEnd:"23:30", protectedBlocks:[], sacrificePriority:[]}`. |
| `protectedBlocks` | array | no | Default `[]`. |
| `fixedEvents` | array | no | Default `[]`. |
| `tasks` | array | no | Default `[]`. |

### `preferences`

| Field | Type | Required | Allowed values | Default |
|---|---|---|---|---|
| `dayStart` | string `"HH:MM"` | paired¹ | `00:00`–`23:59` | `"07:30"` |
| `dayEnd` | string `"HH:MM"` | paired¹ | `00:00`–`23:59`, must be > `dayStart` | `"23:30"` |
| `sacrificePriority` | string[] | no | any strings | `[]` |

¹ If you set either `dayStart` or `dayEnd` you must set both — this mirrors the "Daily Schedulable Time Range" form, which saves them together as one preference.

### `protectedBlocks[]`

| Field | Type | Required | Allowed values |
|---|---|---|---|
| `label` | string | **yes** (non-empty) | any |
| `days` | string[] | **yes** (non-empty) | `mon tue wed thu fri sat sun` |
| `start` | string `"HH:MM"` | **yes** | must be < `end` |
| `end` | string `"HH:MM"` | **yes** | must be > `start` |

### `fixedEvents[]`

| Field | Type | Required | Default |
|---|---|---|---|
| `name` | string | **yes** (non-empty) | — |
| `days` | string[] | **yes** (non-empty, from `mon..sun`) | — |
| `start` | string `"HH:MM"` | **yes**, < `end` | — |
| `end` | string `"HH:MM"` | **yes**, > `start` | — |
| `location` | string | no | `""` |
| `people` | string | no | `""` |

### `tasks[]`

| Field | Type | Required | Allowed values | Default |
|---|---|---|---|---|
| `name` | string | **yes** (non-empty) | any | — |
| `category` | string | **yes** | `must` \| `want` \| `optional` | — |
| `deadline` | string \| `null` | no | `null`, `"YYYY-MM-DD"`, or `"YYYY-MM-DDTHH:MM"` | `null` (no deadline) |
| `estimatedMinutes` | number | **yes** | > 0 | — |
| `repeatPerWeek` | integer | no | ≥ 1 | `1` |
| `splittable` | boolean | no | `true`/`false` | `false` |
| `minBlockMinutes` | number \| `null` | no | > 0 and ≤ `estimatedMinutes` | `null` (scheduler then uses the full `estimatedMinutes` as the effective minimum — same fallback the form's blank field produces) |
| `timePreference` | string | no | `any` \| `morning` \| `afternoon` \| `evening` | `"any"` |
| `preferredDay` | string \| `null` | no | `null` or one of `mon..sun` | `null` (no day restriction) |
| `locationOptions` | string[] | no | any strings | `[]` |
| `notes` | string | no | any | `""` |
| `completed` | boolean | no | `true`/`false` | `false` |

**Note on `locationOptions`:** the form stores this as one comma-joined string
(`"Home, Library"`) because it's a single text field; the JSON schema accepts an **array**
instead (`["Library","Home"]`) because that's a cleaner interchange shape, and the importer
joins it into the same comma-separated string internally — so a round-tripped task ends up
byte-identical to one you'd get from typing `Library, Home` into the form. Export converts it
back to an array the same way.

**Note on `minBlockMinutes` and effective behavior:** whether stored as `null` (JSON omitted
it) or as an explicit number equal to `estimatedMinutes`, `schedule()` reads it as
`+task.minBlockMinutes || sessionMinutes` — the two forms are functionally identical. Import
defaults to `null` (matching what the form produces when you leave that field blank); export
writes back the *effective* number, so an exported-then-reimported file still validates and
still behaves identically.

## 4. Complete working example

This is the actual testcase used to verify the feature (imported successfully end-to-end,
6 fixed events / 8 tasks / 1 protected block / no scheduling shortfalls):

```json
{
  "testCaseName": "Main Weekly Planner Test",
  "weekStart": "2026-09-28",
  "preferences": {
    "dayStart": "08:00",
    "dayEnd": "22:00",
    "sacrificePriority": ["optional tasks", "reading"]
  },
  "protectedBlocks": [
    { "label": "Friday night off", "days": ["fri"], "start": "19:00", "end": "23:00" }
  ],
  "fixedEvents": [
    { "name": "Operating Systems Class", "days": ["mon"], "start": "09:00", "end": "12:00", "location": "University", "people": "" },
    { "name": "Lab Meeting", "days": ["wed"], "start": "14:00", "end": "15:00", "location": "Lab", "people": "Lab members" },
    { "name": "Gym Class", "days": ["fri"], "start": "07:00", "end": "08:00", "location": "Gym", "people": "" },
    { "name": "Seminar", "days": ["thu"], "start": "10:00", "end": "11:00", "location": "Room 2", "people": "" },
    { "name": "Commute AM", "days": ["mon","tue","wed","thu","fri"], "start": "08:00", "end": "08:30", "location": "Transit", "people": "" },
    { "name": "Commute PM", "days": ["mon","tue","wed","thu","fri"], "start": "18:00", "end": "18:30", "location": "Transit", "people": "" }
  ],
  "tasks": [
    { "name": "OS Assignment", "category": "must", "deadline": "2026-10-01T23:59", "estimatedMinutes": 240, "repeatPerWeek": 1, "splittable": true, "minBlockMinutes": 60, "timePreference": "morning", "preferredDay": null, "locationOptions": ["Library", "Home"] },
    { "name": "Database Project", "category": "must", "deadline": "2026-10-02T23:59", "estimatedMinutes": 180, "repeatPerWeek": 1, "splittable": true, "minBlockMinutes": 45, "timePreference": "any", "preferredDay": null, "locationOptions": ["anywhere"] },
    { "name": "Exam Review", "category": "must", "deadline": "2026-10-03T18:00", "estimatedMinutes": 120, "repeatPerWeek": 1, "splittable": false, "minBlockMinutes": 120, "timePreference": "afternoon", "preferredDay": null, "locationOptions": ["Library"] },
    { "name": "Reply Emails", "category": "must", "deadline": "2026-09-30T18:00", "estimatedMinutes": 30, "repeatPerWeek": 1, "splittable": true, "minBlockMinutes": 10, "timePreference": "any", "preferredDay": null, "locationOptions": ["anywhere"] },
    { "name": "Running", "category": "want", "deadline": null, "estimatedMinutes": 60, "repeatPerWeek": 2, "splittable": false, "minBlockMinutes": 60, "timePreference": "any", "preferredDay": null, "locationOptions": ["Outdoor", "Gym"] },
    { "name": "Reading", "category": "want", "deadline": null, "estimatedMinutes": 30, "repeatPerWeek": 3, "splittable": true, "minBlockMinutes": 20, "timePreference": "evening", "preferredDay": null, "locationOptions": ["anywhere"] },
    { "name": "Movie", "category": "optional", "deadline": null, "estimatedMinutes": 150, "repeatPerWeek": 1, "splittable": false, "minBlockMinutes": 150, "timePreference": "evening", "preferredDay": null, "locationOptions": ["Home", "Cinema"] },
    { "name": "Organize Desk", "category": "optional", "deadline": "2026-10-10", "estimatedMinutes": 45, "repeatPerWeek": 1, "splittable": true, "minBlockMinutes": 15, "timePreference": "any", "preferredDay": null, "locationOptions": ["Home"] }
  ]
}
```

## 5. Field → HTML input → internal state mapping

| JSON path | Form field(s) (`app.html` id) | db location | Internal state |
|---|---|---|---|
| `weekStart` | *(no manual-UI equivalent — see §6)* | — | `state.pinnedWeekStart` (a `Date`), consumed by `getWeekDates()` in place of `new Date()` |
| `preferences.dayStart` / `.dayEnd` | `#pref-daystart`, `#pref-dayend` + `#pref-daytime-save` | `prefs/main` | `state.prefs.dayStart/dayEnd` → `schedule()`'s `dayStart`/`dayEnd` |
| `preferences.sacrificePriority` | `#priority-list` / `#priority-new` / `#priority-add` (↑↓✕ per row) | `prefs/main` | `state.prefs.sacrificePriority` — **stored and editable, not yet read by `schedule()`** (see §6) |
| `protectedBlocks[]` | `#protected-form` (`#protected-label`, `#protected-days`, `#protected-start`, `#protected-end`) | `prefs/main.protectedBlocks` | `state.prefs.protectedBlocks` → placed as immovable blocks in step 1 of `schedule()` |
| `fixedEvents[].name/days/start/end/location/people` | `#fixed-form` (`#fixed-name`, `#fixed-days`, `#fixed-start`, `#fixed-end`, `#fixed-location`, `#fixed-people`) | `fixedEvents/<id>` | `state.fixedEvents[]` → placed first in `schedule()`; `location`/`people` are display-only (see §6) |
| `tasks[].name` | `#task-name` | `tasks/<id>.name` | same |
| `tasks[].category` | `#task-category` | `.category` | drives sort order (`must`→`want`→`optional`) and the block's color/pill |
| `tasks[].deadline` | `#task-deadline` (datetime-local) | `.deadline` | clips the eligible placement window (`clippedGaps()`) |
| `tasks[].estimatedMinutes` | `#task-minutes` | `.estimatedMinutes` | the session duration to place |
| `tasks[].repeatPerWeek` | `#task-repeat` | `.repeatPerWeek` | number of independent sessions placed per week |
| `tasks[].splittable` | `#task-splittable` (checkbox) | `.splittable` | gates the split-fallback pass |
| `tasks[].minBlockMinutes` | `#task-minblock` | `.minBlockMinutes` | minimum chunk size when splitting |
| `tasks[].timePreference` | `#task-pref` | `.timePreference` | which of the two placement passes matches first (see `prefWindow()`) |
| `tasks[].preferredDay` | `#task-day` | `.preferredDay` | narrows `baseEligible` to one weekday |
| `tasks[].locationOptions` | `#task-location` (free text, comma-separated) | `.locationOptions` (string) | display-only (see §6) |
| `tasks[].notes` | `#task-notes` | `.notes` | shown under the task row in the master list |
| `tasks[].completed` | the ✓ checkbox on the master task list | `.completed` | excludes the task from scheduling once true (`activeTasks = tasks.filter(t => !t.completed)`) |

## 6. UI/data that the testcase JSON does **not** cover

- **The Weekly Reflection log** (`#reflect-wins`, `#reflect-unfinished`, `#reflect-longer`,
  `#reflect-change`, `#reflect-notes`, stored at `reflections/<mondayISOdate>`) is out of
  scope by design — it's the user's own journal about how a week *went*, not scheduling
  input, so it isn't part of a testcase. (Reset still clears the currently-displayed week's
  reflection doc, so a re-run starts clean; import does the same for the week it pins.)
- **`weekStart` has no manual-UI counterpart.** The normal UI only ever moves relative to
  today via the Prev Week / This Week / Next Week buttons (`state.weekOffset` against the
  real `new Date()`). Pinning
  an arbitrary week regardless of the real calendar date is a dev-tool-only capability
  (`state.pinnedWeekStart`) — that's the whole point of the field (§ "Planning Week" in the
  original request: V1 and V2 may run on different real-world dates but must render the same
  testcase week).
- Every other field a person can type into the form **is** represented in the JSON — see the
  id list in §5; nothing in the current UI was left out.

## 7. Fields the scheduler needs that the user never enters directly

- **Document ids.** Every `fixedEvents`/`tasks` doc gets a store-generated id (`db.collection(...).add()`
  mints one; the importer never sets one explicitly). The id becomes `taskId` on any calendar
  block the task produces, which is what lets the UI grey out a block (`.done`) when its task
  is checked off, and what the master list's "Scheduled at" column uses to look up where a
  task landed.
  Not meaningful to put in a testcase file — re-importing the same JSON twice will legitimately
  produce different ids, and that's fine.
- **`state.weekOffset`.** Navigation position (prev/this/next), not data — reset to `0` by
  both import and reset.
- **Two fields are captured today but not yet consumed by `schedule()`:**
  `preferences.sacrificePriority` and every task's `locationOptions` (and a fixed event's
  `location`/`people`). All four are stored, editable, exported and re-imported faithfully —
  they just don't change the computed schedule yet (category order already encodes "must
  outranks want outranks optional" without needing the priority list; location is shown in
  block tooltips and task rows but isn't used as a placement constraint — see
  `references/algorithm.md`, "Location"). Worth knowing before writing a testcase meant to
  probe location- or sacrifice-order-sensitive behavior: V1 will accept and echo those fields
  back on export, but won't act on them.

## Round-trip guarantee

Import → internal state → `schedule()` and manual-entry → internal state → `schedule()` are
the same code path from the moment data lands in `state`/db — `validateTestCase()` only
*produces* the same shape the form already writes (see §5's "internal state" column), it does
not introduce a parallel scheduling model. Export is the exact inverse of that normalization
(array ⇄ comma-string for `locationOptions`, `null` ⇄ "" for optional strings, etc.), so
export → import of the same file reproduces the same state.
