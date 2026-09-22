# Brief

## Motivation

Without planning ahead, mandatory tasks fill every slot, and running, reading, socializing,
and rest keep getting pushed back. A standard calendar only lists what to do — it doesn't help
place tasks around fixed commitments or protect free time. V1 was built to solve this: read
fixed events and flexible tasks together, and auto-generate a weekly schedule by category,
deadline, and duration.

But after actually using V1, I discovered a problem the design hadn't anticipated: I didn't
like some of the placements the algorithm chose, and wanted to move a task — a run pushed to
evening instead of morning, a reading block moved to a freer afternoon. V1 had no way to do
this. The only path was editing the source task's constraints and regenerating the whole week,
which often reshuffled everything else too. The algorithm's output was being treated as final,
when in practice it was only ever a starting point.

## Design and Improvement

That discovery is what V2 was built around: V1's algorithm stays as the baseline, but its
output is now a recommendation, not a final answer. V2 adds direct manual adjustment —
dragging a task block to a new time slot on the calendar itself, validated live against
overlaps and deadlines, with undo and a reset back to the generated plan.

Building V2 also surfaced a separate, real bug in shortfall reporting: a Must task with
`repeatPerWeek > 1` that couldn't be fully placed only reported the one session that first
failed, not the rest. On this project's own test data, a 15×/week task that could realistically
fit only 1 session was misreported as a 30-minute shortfall; the fix now reports the true 420
minutes across 14 sessions.

## Results and Limits

Under the same `testcase.json`, both versions import identically and preserve all fixed
events. In V1, moving one task means editing its input and regenerating the whole schedule; in
V2 I drag just that block, and the rest of the week stays untouched — directly closing the gap
that prompted V2. Location still does not affect scheduling in either version.

**Model used** — Skill development: Claude Sonnet 5. Idea inspiration: ChatGPT 5.
