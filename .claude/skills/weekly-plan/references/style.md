# Output style notes

The output should read like a paper planner (手帳) weekly spread, not a generic SaaS
calendar. Concretely:

- Warm, slightly textured background rather than stark white; a page/notebook feel.
- Handwriting-adjacent or rounded display font for headings is fine; body text stays legible
  (a clean sans/serif), per the artifact-design type-pairing guidance.
- Four-category legend, always visible, each with a color **and** a shape/icon so it survives
  grayscale printing:
  - Fixed events — solid block, neutral/ink color, small pin icon
  - Scheduled tasks — colored block, icon varies by Must (!) / Want (heart) / Optional (dot)
  - Personal activities — a distinct accent color (e.g. green), leaf/activity icon
  - Free blocks — outline only, no fill, so the eye rests there instead of skipping past it
- Keep the grid honest: if a day has 3 free hours, show 3 hours of visible empty space, not a
  compressed sliver next to padded task blocks.
- Task master list: group by category or sort by deadline (pick one, state which, in a small
  toggle if easy — not required for v1). Overdue-but-incomplete tasks get a visible warning
  treatment (not just red text — color alone isn't enough).
- Must-task or weekly-target shortfalls from the algorithm go in a small callout at the top
  of the page, not buried — the user needs to see what didn't fit before the week starts.
- Mobile width (16px gutters, no horizontal scroll) — the user will likely check this on
  their phone mid-day.
