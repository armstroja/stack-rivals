# Making an engine compatible with `arena.html`

**Audience: an AI assistant (or developer) asked to adapt a stacker/Tetris bot engine so it runs inside `arena.html`.**
Read this whole file, then follow the checklist at the bottom.

`arena.html` is a single HTML file that runs two bots against each other. It embeds one JavaScript function, `ENGINE()`, which is the only part you replace. Everything else (rendering, garbage, scoring, match flow, UI) stays unchanged. Your job is to make your engine expose the exact interface below.

---

## 1. Hard constraints

1. **`ENGINE` must be a top-level `function ENGINE(){ ... return {...}; }` declaration.**
   The page calls `ENGINE()` on the main thread and also does `ENGINE.toString()` to build a Web Worker from the source text. Therefore:
   - It must be **fully self-contained**: no references to variables, imports, or helpers defined outside the function body.
   - No `import`/`require`, no DOM access, no `window`, no `localStorage`.
   - Each call must return a fresh, independent engine (the worker calls it again).
2. **Plain JavaScript (ES2018-ish), no build step.**
3. **Planning runs inside a worker** and must return structured-cloneable data only (numbers, arrays, plain objects, no functions or typed-array subclasses with extra props).

---

## 2. Board and piece model (what the arena assumes)

| Concept | Contract |
|---|---|
| Width `W` | `10` columns |
| Height `H` | `40` rows total (rows `0..19` are hidden buffer, rows `20..39` are visible) |
| `TOP` | `20`, the first visible row (`H - 20`) |
| Board | `Int32Array(H)`; each element is a row bitmask, **bit `x` set = filled cell in column `x`** (column 0 is the left side). Row 0 is the top, row `H-1` is the bottom. `y` increases downward. |
| `FULL` | `(1<<W)-1`, a complete row |
| Piece ids | `1=I 2=J 3=L 4=O 5=S 6=T 7=Z` (0 means "none") |
| Rotation `r` | `0..3`; `0` is spawn orientation; rotation direction `d`: `1`=CW, `3`=CCW, `2`=180 |
| `SH` | `SH[id][r]` is an `Int8Array(8)`: `[x0,y0,x1,y1,x2,y2,x3,y3]`, the four cell offsets from the piece origin `(x,y)` (origin = top-left of the piece's bounding box) |
| `SPX` | `SPX[id]` = spawn column of the origin |
| `SPY` | spawn row of the origin (the reference engine uses `TOP-2` = 18) |

If your engine uses a different representation (2D arrays, y-up, 20-row boards, etc.), **write an adapter inside `ENGINE()`** that converts at the boundary. The arena only touches the interface below.

---

## 3. Required exports

`ENGINE()` must `return` an object containing **all** of these:

```js
return { W, H, TOP, FULL, SPY, SH, SPX,
         collide, spinOf, attackOf, AO,
         Bag, rng32, planPath,
         KEYS, DEF };
```

(The reference engine also exports `rotate, RR, clearLines, addGarbage, search, getPath, plan, trainGen, playSim, gauss`. **`arena.html` does not use these**, so you may omit them. They are only needed by the full training game, `index.html`.)

### 3.1 Constants
`W, H, TOP, FULL, SPY` are numbers. `SH` is `[null, rots...]` indexed by piece id (see table above). `SPX` is an array indexed by piece id (`SPX[0]` unused).

### 3.2 `collide(board, id, r, x, y) -> boolean`
Returns `true` if piece `id` in rotation `r` with origin `(x,y)` overlaps a wall (`X<0 || X>=W`), the floor (`Y>=H`), or a filled cell. Cells with `Y<0` count as empty (above the top).

### 3.3 `spinOf(board, id, r, x, y, rotated, fifthKick, allMini) -> 0 | 1 | 2`
Classifies the piece **at its final locked position, before the cells are written to the board**.
- Returns `0` = no spin, `1` = mini spin, `2` = full spin.
- `rotated`: the last successful move was a rotation (arena passes `f.v > 0`).
- `fifthKick`: the last rotation used the 5th kick (arena passes `f.v === 2`). Used to promote T-spin minis to full.
- `allMini`: when `true`, non-T spins count as minis instead of full spins.
- Must return `0` when `rotated` is false, and `0` for the O piece.

### 3.4 `attackOf(lines, spin, b2b, combo, perfectClear) -> AO`
Computes garbage sent for one lock **that cleared `lines > 0`**. Must **write into the shared array `AO` and return it** (the arena reads `E.AO[0..2]` immediately):
- `AO[0]` = attack (garbage lines sent, before cancelling)
- `AO[1]` = new back-to-back counter (`-1` means chain broken/none, `0` means chain started, `>=1` active bonus)
- `AO[2]` = new combo counter (`combo+1`; the arena resets combo to `-1` itself on a no-clear lock)

Input conventions: `b2b` and `combo` come in as the previous values, with `-1` meaning "none". `spin` is the `0|1|2` from `spinOf`.
`AO` is a module-level `[0,0,0]` created once inside `ENGINE()`. You may change attack tables freely; the arena has no table of its own.

### 3.5 `Bag(seed) -> () => pieceId`
Returns a generator of piece ids `1..7` (7-bag recommended). The arena creates **one bag per side with the same seed**, so both bots see the identical sequence. Must be deterministic per seed.

### 3.6 `rng32(seed) -> () => number in [0,1)`
Deterministic PRNG. Used for garbage hole placement.

### 3.7 `KEYS` and `DEF`
- `KEYS`: array of strings naming each brain weight.
- `DEF`: numeric array, same length as `KEYS`, the default ("starter") weights.
- **Brain files are only accepted if `mean.length === KEYS.length`.** Changing `KEYS` order or length invalidates every brain trained with the old engine.

### 3.8 `planPath(st, w, o) -> plan | null`  (the important one)
Called in the worker once per piece. Chooses where the current piece goes.

**Input `st`** (plain object):
```js
{
  board:   number[H],   // row bitmasks (a plain array; convert with Int32Array.from)
  cur:     number,      // id of the piece to place (already removed from queue)
  hold:    number,      // id in hold slot, or 0
  queue:   number[],    // upcoming piece ids; queue[0] is the next piece (>= 6 entries)
  b2b:     number,      // -1 = none
  combo:   number,      // -1 = none
  canHold: boolean
}
```
**Input `w`**: `number[]`, the brain weights (`KEYS.length` entries).
**Input `o`** (options chosen by the arena from the brain's metadata):
```js
{ tucks: true, useHold: true,
  look: 6|7|8,      // lookahead breadth (how many top candidates get a second-ply search)
  noise: number,    // gaussian noise added to candidate scores (weaker bots = more noise)
  mistake: number,  // probability of picking a random top-6 candidate
  allMini: boolean, // pass through to spinOf
  appOnly: boolean  // true = reward only attack (see section 5)
}
```
You may ignore any option your engine does not support, but `look`, `noise` and `mistake` are what make a young brain weaker than a trained one, so honoring them keeps matches meaningful.

**Return value**: `null` if no legal placement exists (the arena treats this as a top-out), otherwise:
```js
{
  useHold: boolean,   // true = swap with hold first
  id:      number,    // id of the piece that will actually be placed (after hold swap)
  f:  { x, y, r, v }, // final origin, rotation, and v = how it got there:
                      //   0 = hard drop / slide only, 1 = last move was a rotation,
                      //   2 = last move was a rotation using the 5th kick
  path: number[]      // optional; the arena ignores it (the full game uses it for animation)
}
```
**Rules the arena enforces on your result** (a violation causes a silent re-plan, so bugs can look like stalls):
- After the optional hold swap, the current piece id must equal `plan.id`.
- `collide(board, id, f.r, f.x, f.y)` must be `false` at the returned position.
- The position must be a **resting** position (the piece cannot move down further), or the stack will have floating pieces.
- When the hold slot is empty and `useHold` is true, the placed piece is `queue[0]`, so `id` must equal `queue[0]`.

**Pure function requirement:** do not read or mutate anything outside `st`/`w`/`o`; copy `st.board` with `Int32Array.from(st.board)` before use.

---

## 4. What the arena does itself (do NOT reimplement in the engine)

- Writing the locked piece into the board, tracking colors, clearing lines, detecting perfect clears.
- Top-out rules: any cell above row 0 (`Y<0`), a lock entirely inside the hidden rows (`Y<TOP` for all cells), spawn collision, or garbage pushing filled rows out of the buffer.
- Calling `spinOf` / `attackOf` and updating `b2b` and `combo`.
- Garbage: cancelling incoming garbage with outgoing attack, a 500 ms delay before garbage becomes "ready", delivery only on a lock that cleared nothing, a cap of 8 rows per delivery, one shared hole per delivery, and a 30% chance the hole column moves.
- Pressure: after 60 s both sides receive growing garbage so a match always ends.
- Pacing: the bot's PPS comes from the brain's `gen` and `appEma` (or the Speed slider); the arena waits `1000/PPS` ms, then snaps the piece to `plan.f`.
- Rendering, scoring, UI, brain file loading.

---

## 5. The `appOnly` option

When `o.appOnly` is true, the reward for a placement should be **only the attack it produces** (attack per piece), scaled to be comparable with your board-evaluation terms. The reference engine does:

```js
const ATK_R = 2.5;
function reward(res, id, w, prevB2b, appOnly){
  if(res.dead) return -1e9;
  if(appOnly) return res.atk * ATK_R;   // attack already includes spin/B2B/combo/PC bonuses
  /* ...shaped rewards using the learned weights... */
}
```
Board-shape terms (holes, bumpiness, wells) remain active so the bot survives. If your engine has no such mode, ignore the flag.

---

## 6. Brain file format (what "Load brain" accepts)

Arena parses JSON of either shape, `{ "bot": {...} }` or the bare object. Required and used fields:

```json
{
  "format": "stack-rival-brain",
  "bot": {
    "name": "PIP",
    "mean": [ /* KEYS.length finite numbers */ ],
    "gen": 120,
    "appEma": 1.35
  }
}
```
- `mean` is **required** and must have exactly `KEYS.length` finite numbers, otherwise the file is rejected.
- `name` is uppercased and stripped to `A-Z 0-9 _ -`, max 10 chars. `gen` defaults to 0. `appEma` is `null` or a number.
- Other fields (`xp`, `sigma`, `wins`, ...) are ignored by the arena.

`gen` and `appEma` drive the bot's strength settings (noise, mistake rate, lookahead breadth, default PPS), so a brain with higher `gen` plays cleaner and faster. If your engine has its own weight vector, map it into `mean` and keep `KEYS` consistent.

---

## 7. Integration steps

1. Open `arena.html`. Near the top of the `<script>` block there is the reference `function ENGINE(){ ... }` followed by `const E=ENGINE();`.
2. **Replace only the `function ENGINE(){ ... }` block** with your adapted engine. Leave `const E=ENGINE();` and everything below it untouched.
3. Make sure the returned object has every export in section 3.
4. Make sure `KEYS.length === DEF.length`, and that any brain you want to load has `mean.length === KEYS.length`.

---

## 8. Self-test (run in Node before opening the page)

```js
const E = ENGINE();                       // paste your function above this
const bag = E.Bag(123);
const board = new Int32Array(E.H);
for (let i = 0; i < 100; i++) {
  const cur = bag(), queue = Array.from({length:6}, bag);
  const plan = E.planPath(
    { board: Array.from(board), cur, hold: 0, queue, b2b: -1, combo: -1, canHold: true },
    E.DEF,
    { tucks:true, useHold:true, look:6, noise:0.1, mistake:0.001, allMini:true, appOnly:false }
  );
  if (!plan) { console.log('no placement at piece', i); break; }
  const id = plan.id, f = plan.f;
  console.assert(!E.collide(board, id, f.r, f.x, f.y), 'plan collides');
  console.assert(E.collide(board, id, f.r, f.x, f.y + 1), 'plan is not resting');
  const k = E.SH[id][f.r];
  for (let c = 0; c < 8; c += 2) board[f.y + k[c+1]] |= 1 << (f.x + k[c]);
  let w = E.H - 1;                        // simple line clear
  for (let r = E.H - 1; r >= 0; r--) if (board[r] !== E.FULL) board[w--] = board[r];
  while (w >= 0) board[w--] = 0;
}
console.log('ok');
```
Also check: `E.attackOf(2, 0, -1, -1, false)` returns an array of length 3 and `E.AO` is the same array object; `E.spinOf(board, 6, 0, 3, 18, false, false, true) === 0`.

---

## 9. Common failures

| Symptom | Likely cause |
|---|---|
| Page loads but the bots never move | `planPath` throws in the worker (open DevTools, check the worker error), or returns an object with the wrong shape |
| Bots freeze after a hold | `plan.id` does not match the piece after the swap (check the empty-hold case: `id` must equal `queue[0]`) |
| Pieces float above the stack | `f.y` is not a resting position |
| "Not a Stack Rival brain" alert | `mean.length !== KEYS.length` |
| Worker fails to start | `ENGINE` references outside variables, so `ENGINE.toString()` is not self-contained |
| Everyone tops out instantly | Board/row orientation mismatch (row 0 must be the top, `y` increases downward, hidden rows are `0..19`) |
| Attack looks wrong | `attackOf` did not write into the shared `AO`, or `b2b`/`combo` sentinel is not `-1` |
| Scores are the same every time | `Bag`/`rng32` ignoring the seed |

---

## 10. Prompt to give an AI along with this file

> Here is `arena.html` and the guide `ARENA_ENGINE_GUIDE.md`. Replace the `function ENGINE(){...}` block in `arena.html` with an adapter around my existing engine (pasted below) so that it satisfies the guide's interface exactly. Keep the `const E=ENGINE();` line and the rest of the file unchanged. Keep the function self-contained. When done, run the self-test from section 8 and report the results.
