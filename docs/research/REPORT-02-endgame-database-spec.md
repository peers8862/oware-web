# Spec: An Oware Endgame Database (Retrograde Analysis) for oware-web

Implementation spec for a perfect-play endgame database, derived from:
- **Stone, Sturtevant & Schaeffer (2024)**, *Set-Based Retrograde Analysis* (`papers/Set-Based-Retrograde-Analysis-2024.pdf`) — the algorithm and the indexing principle.
- **van den Herik, Uiterwijk & van Rijswijck (2002)**, *Games Solved* — why Oware's
  convergence makes this work (see `REPORT-01-foundations.md`).

It targets the existing engine in `index.html`, whose state is
`{ h:[12 ints], score:[s0,s1], turn:0|1, ncp }` — `h` = the 12 houses, `score` =
seeds already captured (off-board), `turn` = side to move.

---

## 1. The core idea, in one paragraph

Oware is **convergent**: seeds only ever *leave* the board (into a store on capture),
never appear. So the total seeds on the board, `boardSeeds = Σ h`, is **monotonically
non-increasing**. That lets us solve the game in **layers**, from few seeds up to many:
every capture jumps to a *strictly smaller* layer that is already solved. For each
position we store a single number — the perfect-play value — and we never store the
position itself. Instead a **ranking function** (a perfect hash) maps each position to a
unique array offset; *the offset is the position*. This is the key sentence from the
2024 paper:

> "In retrograde analysis, a ranking or perfect hash function is used to map every state
> to a unique number. States are not explicitly stored; the offset in memory uniquely
> identifies a state."

---

## 2. Two design decisions that shrink the database

### 2.1 Store-independent value encoding
Don't index on `score`. A position's future is unaffected by seeds already banked — only
by the seeds still on the board. So **define the stored value as**:

> `V(pos) = the number of the on-board seeds that the side-to-move will ultimately
> capture, under optimal play by both sides.`

`V` ranges over `0..boardSeeds`, so one byte per entry is plenty. To get a real game
result, recombine with the live stores:

```
seedsToMover      = V(h, turn)                      // from the DB
seedsToOpponent   = boardSeeds - seedsToMover
finalMover    = score[turn]    + seedsToMover
finalOpp      = score[1-turn]  + seedsToOpponent
result = sign(finalMover - finalOpp)               // win / draw / loss
```

This collapses the millions of positions that differ only in `score` into one entry —
the single biggest reduction available, and it's free.

### 2.2 Layer = `boardSeeds`; solve low → high
There are 12 houses, so the number of ways to place `s` indistinguishable seeds is the
**stars-and-bars** count `C(s+11, 11)`. Cumulatively (hockey-stick identity), all
positions with **≤ N** seeds number exactly `C(N+12, 12)`, doubled for `turn`:

| N (max seeds in DB) | positions `C(N+12,12)×2` | bytes @1B/entry |
|---|---|---|
| 10 | ~1.3 M | ~1.3 MB |
| 15 | ~35 M | ~35 MB |
| 16 | ~61 M | ~61 MB |
| 17 | ~104 M | ~104 MB |
| 20 | ~451 M | ~451 MB |

For a browser app, **N = 15 (~35 MB, gzips smaller)** is a sweet spot; the full game has
48 seeds, so this perfectly solves the last third of every game and is hit early in many
capture-heavy lines. (Romein & Bal went to all 48 with a 178 GB cluster DB — we are
deliberately doing the laptop-scale version.)

---

## 3. Indexing: rank / unrank (the perfect hash)

We need a bijection between a distribution `h = [h0..h11]` with `Σh = s` and an integer
`0 .. C(s+11,11)-1`. The standard tool is the **combinatorial number system** applied to
the "bars" positions of a stars-and-bars encoding. Pseudocode (JS, matches `index.html`):

```js
// Precompute Pascal's triangle once: C[n][k] for n up to N+11.
function rank(h, s) {            // h: 12 ints summing to s
  let r = 0, remaining = s, pitsLeft = 12;
  for (let i = 0; i < 11; i++) { // last pit is determined by the rest
    pitsLeft--;
    // skip all distributions where pit i holds fewer than h[i] seeds
    for (let k = 0; k < h[i]; k++) {
      r += C[remaining - k + pitsLeft - 1][pitsLeft - 1];
    }
    remaining -= h[i];
  }
  return r;                      // 0 .. C(s+11,11)-1
}

function unrank(r, s) {          // inverse: integer -> distribution
  const h = new Array(12).fill(0);
  let remaining = s, pitsLeft = 12;
  for (let i = 0; i < 11; i++) {
    pitsLeft--;
    let c;
    while ((c = C[remaining + pitsLeft - 1][pitsLeft - 1]) <= r) { r -= c; h[i]++; remaining--; }
    // NOTE: the loop condition above is a sketch — see §6 contribution point
  }
  h[11] = remaining;
  return h;
}
```

The DB for layer `s` is then `Uint8Array(C[s+11][11] * 2)`, indexed by
`rank(h,s) * 2 + turn`.

---

## 4. The retrograde algorithm (adapted from Alg. 1)

The paper's Algorithm 1 iterates depth `d`, computing each state's value from its
`d-1` successors. For Oware the "depth" dimension is **`s = boardSeeds`**, and there's a
twist the paper's clean DAG doesn't have: **within a layer, moves can cycle** (a sow that
captures nothing keeps `s` the same and can return to an earlier position). So a layer is
*not* a DAG; it needs a **fixed-point value iteration**, with boundary values flowing in
from the already-solved smaller layers (captures).

```
for s = 0 .. N:                         // small layers first
  allocate EDB[s]
  // 4a. seed terminal & capture edges
  for each (h, turn) with Σh = s:
      if no legal move for `turn`:       // terminal: opponent gathers remainder (rules)
          V = terminalValue(h, turn)
      else:
          V = undefined                  // to be resolved by iteration
  // 4b. fixed-point: negamax backup until nothing changes
  repeat until stable:
      for each unresolved (h, turn):
          best = -infinity
          for each legal move m:
              (h', turn', captured) = applyMove(h, turn, m)
              if captured > 0:           // jumps to a SMALLER, solved layer
                  child = captured + ( (boardSeeds(h') ... ) lookup EDB[s-captured] )
              else:                      // stays in THIS layer
                  child = lookup EDB[s] (may still be undefined this round)
              best = max(best, captured + (childSeedsForMover))
          V(h,turn) = best
  // unresolved-after-convergence  => cyclic draw convention (see §6)
```

Two practical notes:
- Iterate the in-layer backup to convergence (Bellman-style). Convergence is guaranteed
  because every *capture* edge strictly decreases `s`, so all infinite paths are pure
  no-capture cycles — handled by the §6 convention.
- `ncp` (no-capture plies) in the live engine enforces the cycle cutoff at play time; the
  DB encodes the *theoretical* optimal value under the chosen cycle convention.

---

## 5. Wiring it into the existing engine

In `index.html`, `negamax()` (line ~645) is the search. Add a DB probe at the top:

```js
function negamax(s, rules, depth, alpha, beta){
  if (boardSeeds(s) <= DB_MAX_SEEDS) {        // perfect: no search needed
    const v = dbProbe(s, rules);              // exact result from the database
    if (v !== null) return v;
  }
  // ... existing alpha-beta ...
}
```

`dbProbe` does `unrank`/`rank` + the store recombination from §2.1 and returns a value on
the *same scale* `negamax`/`evalState` already use. Effect: the engine plays **perfectly**
once a game has ≤ `DB_MAX_SEEDS` seeds on the board, and uses heuristic α-β above that —
the Lithidion stack from `REPORT-01`.

Build the DB **offline** (a Node script reusing `simulate`/`applyMove` from `index.html`),
emit a compact binary, and lazy-load it (or a gzip slice per layer) in the browser.

---

## 6. ★ Your turn — the one decision that defines correctness

I've scaffolded everything except the single piece where Oware's rules make a genuine
*modelling choice*: **what value to assign when optimal play never captures** (a cycle),
and the matching **terminal value** when a side has no move.

In the real rules, a game that can't progress ends by **dividing the remaining on-board
seeds** — and which seeds go to whom depends on the variant (standard Oware: each player
takes the seeds on their *own* side; some rulesets award all remainder to the last mover).
This convention is the boundary condition the whole fixed-point converges to, so it
*defines* every value in the database.

Please implement `terminalValue(h, turn, rules)` (and decide the cyclic-draw rule it
implies). ~5–10 lines. Consider:
- **No-move terminal:** when `turn` has no legal move, who gets the seeds still on the
  board? (Check the engine's `isOver`/`collectSides` at lines ~607–628 — match it exactly,
  or the DB will disagree with live play.)
- **Cycle/repetition:** if a position's value stays `undefined` after the fixed point, it's
  an infinite no-capture line — apply the same division rule.
- **Variant flag:** the engine already has a `'34'` vs `'23'` capture rule (`capturable`,
  line 537); your function and the build must be parameterised by it (one DB per ruleset).

Tell me which division convention `index.html` actually uses and I'll lock the indexing
math and generate the offline builder script around it.

---

## 7. Validation plan
- **Self-consistency:** for every position, `V(h,turn) + V'(opponent-to-move successor)`
  must reconcile (no value exceeds `boardSeeds`).
- **Round-trip:** `rank(unrank(r,s),s) === r` for random `r` in each layer.
- **Cross-check vs. search:** for low layers (s ≤ 8), brute-force negamax with no DB and
  assert it matches `dbProbe` on thousands of random positions.
- **Known result:** with N large enough, the 4-per-pit start should evaluate to a **draw**
  (the solved value of Awari — see `REPORT-01`).
```
