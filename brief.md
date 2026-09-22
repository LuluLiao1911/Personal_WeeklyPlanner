# Brief

## Motivation

Without planning a week in advance, mandatory tasks quickly fill every available slot, and
running, reading, socializing, and rest keep getting pushed back. A standard calendar app only
lists what needs to be done — it doesn't help place tasks around fixed commitments or protect
usable blocks of free time. This project's scheduler aims to maximize contiguous free time by
treating fixed events, tasks, deadlines, durations, and repeat frequency as inputs to an
automatic weekly placement, cutting the manual effort of re-planning every week. An agent skill
helps because it can read fixed events and flexible tasks together, place tasks by
category/deadline/duration, and regenerate the whole plan in seconds — while priority
(Must/Want/Optional) and what to protect stay entirely the user's calls, never the algorithm's.

## Design and Improvement

V1 takes fixed events, tasks (category, deadline, estimated duration, repeat frequency, minimum
block size, splittability), and preferences, and produces a weekly calendar, a master task
list, free-time suggestions, and a PDF export. Its checks: fixed events are never overlapped,
Must tasks are placed before their deadline (or reported as a shortfall), minimum block sizes
are respected, and repeated sessions are counted correctly — the last of which had a real bug I
found and fixed in V2: a Must task with `repeatPerWeek > 1` that couldn't be fully placed only
reported the one session that first failed, silently dropping the rest. On this project's own
test data, a 15×/week task that could realistically fit only 1 session was originally reported
as a 30-minute shortfall; the fix now correctly reports 420 minutes across 14 sessions.

V1's core limitation was rigidity: once placed, a block could only move by editing the source
task and regenerating the entire week — inflexible against how a week actually unfolds. My
durable judgment is that an AI-generated schedule should be a recommendation, not a final
decision, so the user needs a direct way to revise individual placements without regenerating
the whole plan. V2 keeps V1's algorithm as the baseline and layers manual drag-and-drop
adjustment on top, validated against overlaps and deadlines, with undo and reset.

## Results and Limits

Under the same `testcase.json`, both versions import identically and preserve all fixed events.
In V1, moving one task means editing its input and regenerating the whole schedule; in V2 I can
drag just that one block, and the rest of the week's placements stay untouched. Location still
does not affect scheduling in either version.

**Model used** — Skill development: Claude Sonnet 5. Idea inspiration: ChatGPT 5.
