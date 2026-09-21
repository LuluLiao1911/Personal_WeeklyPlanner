# Output style notes (as implemented in `app.html`)

Same botanical planner look as V1 — see V1's `references/style.md` for the full original
writeup (paper background, sage/rose/mustard block colors, Caveat/Noto Sans TC typography, the
7-day-column grid with a shared time axis, the celebratory-vs-warning summary callout, card
order below the calendar, and the html2canvas+jsPDF rasterized PDF export). All of that is
unchanged in V2. This file only covers what V2 adds.

## Language toggle

A small pill-shaped two-button group (`中文 | English`) in the header, to the right of the
title — same visual weight as a secondary control, not competing with the tab switcher for
attention. The active language is the solid sage-filled button (`aria-pressed="true"`); the
inactive one is a plain-text button in the same pill. It's the only chrome element that isn't
inside a card — like the tabs, it belongs to the page shell.

## Movable / locked / manually-adjusted / conflict block states

Four new visual states layered onto V1's existing four block colors (fixed/must/want/optional),
so a block's category color and its manual-adjustment state are always both visible at once:

- **Movable** (task blocks): `cursor: grab`, a subtle white outline on hover — inviting without
  being loud, since most blocks on a lightly-adjusted week are movable and shouldn't look
  "special."
- **Locked** (fixed events, protected blocks): `cursor: not-allowed`, a small 🔒 glyph in the
  block's top-right corner. This is the one state that needed an explicit always-visible marker
  (per the original request's requirement that locked blocks be visually distinguishable from
  movable ones), since a `cursor` style alone isn't visible in a screenshot or a PDF export.
- **Manually adjusted**: a thin white inset ring around the block plus a small "Adjusted" badge
  in the corner (`popover.originManual`'s short form, `adj.badge`) — a quiet indicator, not a
  color change, so a week with several manual tweaks doesn't look alarming or heavily
  edited-looking. The badge text is intentionally short (one word/label) to stay legible at the
  block's small size.
- **Conflict** (an override that no longer fits when re-applied — see
  `manual-adjustment.md`): a warning-orange outline, reusing the same warning color as V1's
  shortfall summary, so "something needs your attention" reads consistently across the whole
  app rather than introducing a new alarm color.
- **Dragging** (mid-drag, not persisted): the block being dragged gets a drop shadow and slight
  transparency so it visibly "lifts" off the calendar surface; a separate dashed **drop-ghost**
  element (sage outline while the candidate position is valid, warning-red outline the instant
  it isn't) shows where it would land, in the target day column, following the pointer.

## Block-detail popover

A centered modal card, not a slide-out panel or a tooltip — the original request asked for
enough information (name, category, scheduled time, duration, deadline, origin, and an
optional time editor) that a tooltip would be too cramped and a slide-out would fight with the
7-column calendar for horizontal space. Same paper/sage card language as the rest of the app,
Caveat script for the task name heading, small `label : value` rows for the rest. The
day-select + time-input pair only appears (enabled) for a movable block; a locked block's
popover shows everything except that editor and the Save button.

## Rejection toast

A single dark pill at the bottom-center of the viewport, matching the warning color used
elsewhere, auto-dismissing after ~2.5s. Deliberately transient and non-blocking — it explains
*why* a drop bounced back without requiring a dismiss click, since the block itself already
visibly snapped back to its previous position at the same moment.
