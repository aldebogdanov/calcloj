# Tech debt

## Scroll model — replace the giant spacer div

Current viewport uses a "fake-scroll" spacer: one `<div id="space">` sized to the
logical sheet (`cols*CW × rows*RH`) so the native scrollbar reflects the sheet
size, with only the visible window of cells rendered on top.

**Problem:** the spacer height is bounded by the browser's max element pixel size
(~33.5M px Chrome/Safari, ~17.9M px Firefox). At 26px/row that caps usable rows
at ~1.28M (Chrome) / ~688k (Firefox). `MAX-ROWS` is pinned to 600000 to stay
under the Firefox limit. Scrollbar precision is also terrible at that scale
(~1 px ≈ thousands of rows).

**Want:** a scroll model that needs **no huge div** — compute everything from a
logical position instead of a physical one. Options:
- custom/synthetic scrollbar whose thumb maps to a row range (non-linear ok),
  decoupled from any sized element;
- "logical scroll": intercept wheel/drag, keep a virtual `r0/c0` offset, render
  the window at fixed screen coords (translate, not scroll), draw our own
  scrollbar. Address-box jump already proves far-navigation without scrolling.

This removes the row cap entirely and makes the cap purely a coordinate clamp.

## Other

- Concurrent edits can race into a transient `#ERR` / stale toast (per-sheet
  lock serializes server side, but simultaneous async posts arrive unordered).
- No config system yet — grid bounds, geometry are `def` constants. Fold into
  per-sheet settings once persistence lands.

## Sessions — crash/sleep cleanup backstop

DONE: sessions now register on load (`/session/start`) and release on unload via
`navigator.sendBeacon('/session/end')`; sheets are ref-counted and unloaded
(execution context closed, saved) when the last session leaves; viewport is
per-session.

REMAINING: `sendBeacon` only fires on *graceful* unload (tab close, navigate).
A browser crash, killed process, or laptop sleep leaves an orphan session that
pins its sheet forever. Add a backstop:
- stamp each session with last-activity time (touch on /cell, /view);
- a periodic sweep ends sessions idle past a TTL (and unloads their sheets).

NOTE: we dropped the persistent-SSE approach — http-kit does not fire an
async-channel close on idle client disconnect without a write, so SSE-based
lifecycle needed heartbeats. Beacon is simpler and reliable for the common case.

The `/debug` endpoint (session/sheet counts) is dev-only — gate or remove before
any real deployment.
