# Output style notes (as implemented in `app.html`)

A botanical planner look, not a generic SaaS calendar/dashboard:

- Warm oat/parchment paper background (`--bg`/`--paper`), sage-green header bands per day
  column, a small hand-drawn leaf sprig in the page header (inline SVG, no external asset).
- Type: "Caveat" (script) for the big title and day-column headers, "Noto Sans TC" for
  everything else — Caveat has no CJK coverage, so it's used only where the text is Latin or
  numeric.
- Four visually distinct classes, color **and** shape/border so it survives grayscale:
  - **Fixed events** — solid sage block.
  - **Must tasks** — solid rose/terracotta block.
  - **Want tasks** — solid mustard block.
  - **Optional tasks** — mustard block at reduced opacity (clearly related to Want, clearly
    lower-commitment).
  - **Protected/free blocks** — dashed outline only, no fill — the eye should rest there.
- The week is 7 side-by-side day columns (not 7 stacked rows) with a shared time axis on the
  left, matching a real paper weekly-planner grid; block height/position within a column is
  proportional to `(dayEnd-dayStart)`, so real empty time reads as real empty space.
- A summary callout above the grid is green/celebratory when nothing was dropped, and switches
  to a warm red warning listing every shortfall by name when something didn't fit — this is
  the single most important thing on the page and sits above the fold.
- Task master list, free-time suggestions, and the weekly reflection log are separate cards
  below the calendar, in that order — detail after the overview, editable notes last.
- The PDF export (`html2canvas` + `jsPDF`, both loaded from cdnjs at click time, saved via the
  `downloads` capability) rasterizes the on-screen cards rather than re-laying-out text in the
  PDF — this sidesteps CJK font embedding in jsPDF entirely, at the cost of the PDF being an
  image rather than selectable text. Revisit only if the user asks for selectable/searchable
  PDF text.
