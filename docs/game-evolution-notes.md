# Owaré — Game Evolution Notes

A running backlog of ideas for where the game/app could go beyond the current
work. Captured from brainstorming; not yet specced. Items marked **[touches: …]**
connect to a feature currently being designed, so keep them in mind when shaping
that feature even if they aren't built yet.

> Current in-flight work: the **right-side resource rail** (Lesson-of-the-turn
> card + Papers / Reports / Overview / Bibliography overlays). See
> `docs/superpowers/specs/` for its spec.

---

## Learning & lessons

- **Saving / collecting terms and blocks** — let the user bookmark or "collect"
  Lesson cards (and individual terms/definitions) into a personal set they can
  revisit. **[touches: Lesson-of-the-turn card]** — needs the profiles/IndexedDB
  layer below to persist per user; design the Lesson card so a future save action
  fits without rework.
- **Context-triggered lessons** — let the *kind of move just played* surface a
  special lesson, instead of (or alongside) plain sequential rotation. Example
  sketch (approximate): sowing a large house that laps the board (multi-pass)
  could surface a block about recursive / multi-pass algorithms. **[touches:
  Lesson-of-the-turn card]** — v1 rotates sequentially through unlocked blocks;
  this is a later rotation strategy keyed off move characteristics.
- **Gain-points economy for learning** — let the player spend captured-seed
  "Gain" points to buy or unlock learning options / lesson sets. Couples
  progression to play in a second way beyond move-count unlocks.

## Profiles & multi-user (infrastructure)

- **Multiple profiles in IndexedDB** for single-device, multi-user learning
  modes. A simple **player registration**: unique usernames pushed into a list,
  each becoming the ID used to launch any dedicated learning mode. Foundation for
  per-user saved lessons, stats, and the relationship state below.

## Social / relationship state

- **Per-pairing debt/gain ledger** — for any two specific players, maintain an
  ongoing running debt/gain count across all their games (the relationship's
  cumulative balance). Flagged by the user as an important feature.

## Representation modes (how seed counts / numbers are shown)

- **Alternate numeral systems** for the count shown in a cup: Arabic (current),
  Binary (e.g. `0000 0010`), Hex, Tally, Roman.
- **Groups-of-four Unicode glyphs** — represent seeds using partially-divided
  circle characters: e.g. "circle divided by horizontal bar and top half divided
  by vertical bar" for 3, "circled vertical bar" for 2, "circle with plus sign"
  for 4 — so groups of four read as a single glyph, with remainders shown by the
  partially-divided circles, a single whole circle, or a dot. A base-4 / quartal
  tally view native to mancala counting.
- **Formula mode on the board** — use the houses/cups themselves to display
  formulas related to the game's mathematics (math formula overlays positioned on
  the board). Turns the board into a live diagram of the underlying math.

## Board theming / atmosphere

- **Swappable backgrounds** from a pre-loaded set — e.g. street scenes from
  places where Oware is popular (Ghana, the Caribbean). Extends the existing
  board-hue picker into full scene/atmosphere theming.

## Side games

- **Quick-glance row-sum guessing** — a side game: show a row, hide it after N
  seconds, player guesses the row's seed total; includes a randomize button to
  generate fresh rows. Trains subitizing / fast estimation. Candidate as a future
  "education tool" entry on the resource rail.
