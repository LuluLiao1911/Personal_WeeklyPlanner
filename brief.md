# Brief

## Motivation

Without planning ahead, mandatory tasks tend to fill every available slot, while running,
reading, socializing, and rest keep getting pushed back. A standard calendar can record what I
need to do, but it does not help me actively place flexible tasks around fixed commitments,
deadlines, and available time, or preserve larger blocks of free time. I therefore built V1 to
combine fixed events and flexible tasks in one system and automatically generate a weekly
schedule based on category, deadline, and duration.

## Design and Improvement

V1 was designed to reduce the effort of planning from scratch each week. It takes structured
inputs such as fixed events, task priorities, deadlines, estimated durations, and minimum work
blocks, then generates a complete weekly schedule. However, after actually using V1, I realized
that a technically feasible schedule was not always the schedule I personally preferred. I
sometimes wanted to move a run from evening to morning or shift a reading block to a freer
afternoon, but V1 required me to change the original task constraints and regenerate the entire
week, which could also reshuffle unrelated tasks.

This experience directly shaped V2. Instead of treating the algorithm's output as the final
answer, V2 treats it as an initial recommendation. Users can directly drag individual task
blocks to new time slots while keeping the rest of the schedule intact. Each move is checked
against conflicts and deadlines, and users can undo changes or reset to the original generated
plan. I also simplified the free-time section so that it highlights useful opportunities rather
than listing every empty period. The main design shift was therefore from full automation
toward a collaborative planning workflow in which the system handles scheduling complexity
while the user keeps final control.

## Results and Limits

Under the same `testcase.json`, both versions import identically and preserve all fixed
events. In V1, moving one task means editing its input and regenerating the whole schedule; in
V2 I drag just that block, and the rest of the week stays untouched — directly closing the gap
that prompted V2. Location still does not affect scheduling in either version.

**Model used** — Skill development: Claude Sonnet 5. Idea inspiration: ChatGPT 5.
