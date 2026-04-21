# CocoaBugs — Packard's Bugs Implementation Reference

This document describes the implementation of the "Packard's Bugs" model (an agent-based variant of the Bedau–Packard Bugs model) as realised in this Objective‑C / Cocoa codebase. Sources documented below live in `plugins/Packard Bugs/` (the Bugs plugin) and in `BugsFramework/` / `CocoaBugs/` (the host application and plugin protocol).

The document covers:

1. **Architecture** — how the pieces fit together.
2. **The world and food field.**
3. **The bug agent** — state, genome, and lifecycle.
4. **The LUT**: input construction, output space, gene count.
5. **The update loop** — order of operations and their semantics.
6. **All meta-parameters**, both API-exposed and hard-coded.
7. **Meaning of the scales**, especially the mutation rate.
8. **Activity tracking** — the only observable currently close to Bedau's evolutionary activity.
9. **Known quirks / edge cases** that future porters should be aware of.

## 1. Architecture

The host application (`CocoaBugs/`) loads ALife "plugins" that implement the `ALifeController` protocol (`BugsFramework/Source/Protocols/ALifeController.h`). Each plugin exposes:

- `configurationOptions` — list of exposed meta-parameters, declared in a plist.
- `initWithConfiguration:` — initializes simulation state from a parameter dict.
- `update`, `reset`, `view`, `statisticsCollector`, `alive`.
- `setCollectActivity:`, `exportActivity:` — optional gene-activity logging.

The "Packard Bugs" plugin is made of:

- `BugsController` — the `ALifeController` conformer; owns the `World` and wires API parameters.
- `World` — 2D grid, food field, population bookkeeping, and the per-tick update.
- `Cell` — one grid site: holds a `food` bool and (optionally) a `Bug *`.
- `Bug` — one agent: its 32 genes, its energy, age, position.
- `BugsStatistics` — KVO-driven aggregate statistics (population, births, deaths, ages).
- `WorldView`, `BugNeighborhoodView`, `GeneDistributionView`, `BugsColoringWindowController`, `HSVColorSquare` — rendering and the (optional) genome coloring panel.

The configuration schema for the API is `plugins/Packard Bugs/PackardBugs.plist`.

## 2. The world and food field

World state (`World.h` / `World.m`):

- Integer grid `width × height` of `Cell` objects in an `NSMutableArray` of rows.
- If a food image is provided, `width` and `height` are taken from the image pixel dimensions. Otherwise both default to **100**.
- The grid is a **torus** — `cellAtRow:andColumn:` wraps negative or out-of-range coordinates modulo height/width (`World.m:111–116`).
- Each `Cell` stores a single boolean `food` bit (no fractional food, no regeneration).
- Food is set once from a `NSImage` at configuration time: each grid cell samples the image; **pixel brightness `< 0.5` ⇒ food present** (`World.m:84–99`). `foodAmount` is the count of food cells.
- Food is **static** — there is no food regrowth or depletion of the food field over time. Eating a cell does *not* remove the food bit from the cell; a bug on a food tile keeps being fed every step it stays there.

Census sets maintained by `World`:

- `bugs` — all living bugs.
- `morgue` — bugs that died on the most recent tick.
- `maternity` — bugs born on the most recent tick.
- `activeGeneCounts` — an `NSCountedSet` intended for per-tick gene usage counts (declared but only partially populated in the current code).

Seed population: `seedBugsWithDensity:` places a fresh `Bug` on every cell with probability `density`.

## 3. The bug agent

`Bug` instance variables (`Bug.h`):

```
BugMovement genes[32];   // the LUT
int food;                // energy
int age;                 // ticks survived
int x, y;                // grid position
int lastUsedGeneNum;     // for coloring
```

where

```
typedef struct _BugMovement {
    int x;    // signed displacement
    int y;    // signed displacement
    int mag;  // 1..15
    int dir;  // 0..7  (quadrant<<1 | diag)
} BugMovement;
```

On default `init` all 32 genes are randomised with `-randomGene`. Initial food is **`INITIAL_FOOD = 10`** (`Bug.m:11`). Initial age is 0.

### Reproduction

`-doReproduceWithMutationRate:` halves the parent's food (`food / 2`), then returns a new `Bug` whose constructor `-initWithFood:andGenes:mutationRate:` copies the genome with independent per-gene mutation. **The parent keeps `food = food/2`, the child starts with that same halved value.** (Note: integer division; if parent had 21 food, each ends up with 10 and 1 food unit vanishes.)

### Eating / digesting

`-doEat:amount` adds `amount` to `food`.
`-doDigest:amount` subtracts `amount` from `food`.
Both are called from the world update (see §5).

## 4. The LUT: inputs and outputs

### 4.1 Input space — 5-bit von-Neumann-plus-self food mask

Implemented in `-[World calculateNeighborhoodForCellAtRow:andColumn:]` (`World.m:160–185`):

```
cell      = grid[i][j]       // self
upCell    = grid[i+1][j]     // row+1
leftCell  = grid[i][j-1]
rightCell = grid[i][j+1]
downCell  = grid[i-1][j]

bit 0 (1)  : upCell.food
bit 1 (2)  : leftCell.food
bit 2 (4)  : cell.food        // the site the bug stands on
bit 3 (8)  : rightCell.food
bit 4 (16) : downCell.food
```

So the LUT input is a **5-bit pattern**, giving **2⁵ = 32 possible inputs** — and `Bug.genes` is correspondingly an array of 32 `BugMovement`s. The neighborhood is the 4-cell von Neumann neighborhood **plus the bug's own cell**; diagonals are not sensed. The bug can only sense presence/absence of food — it cannot sense other bugs.

The coordinate convention is peculiar: "up" is `row+1`. In `World.m`'s grid, row indices increase from the bottom of the drawn image (the food-image sampling explicitly uses `height - i - 1` as the y coordinate). Whether "up" is visually up depends on the renderer; for modelling purposes it is irrelevant (it is just a label on one of four neighbors).

### 4.2 Output space — quantised 2D displacement

Each gene slot stores one `BugMovement` drawn by `-[Bug randomGene]` (`Bug.m:98–131`):

- `magnitude` uniform in **`{1, 2, …, 15}`** (`random() % 15 + 1`).
- `diag` ∈ `{0, 1}` — axial vs. diagonal.
- `quad` ∈ `{0, 1, 2, 3}` — which quadrant.

The eight resulting direction vectors (for magnitude `m`) are:

| quad | diag=0 (axial) | diag=1 (diagonal) |
|------|----------------|-------------------|
| 0    | (+m,  0)       | (+m, +m)          |
| 1    | ( 0, +m)       | (-m, +m)          |
| 2    | (-m,  0)       | (-m, -m)          |
| 3    | ( 0, -m)       | (+m, -m)          |

So the **full output alphabet is 8 directions × 15 magnitudes = 120 possible moves** per gene slot. `dir = (quad << 1) | diag` ∈ `{0..7}`.

Note that `magnitude = 0` (stay put) is **not representable** — the bug must always move at least one cell per tick.

### 4.3 Total genome size

One bug's genome is a function `{0..31} → (dir, mag) ∈ {0..7} × {1..15}`, i.e. `120^32 ≈ 8·10^66` distinct genomes (ignoring how the random distribution over these is not uniform — see §9).

The `World`-level activity table (when enabled) is sized `32 · 8 · 16 = 4096` (actually `32 · 128 = 4096`; see `GENOME_SIZE = 32*128` in `World.m:11`), with `mag` stored in 4 bits (mag=0 is unused) and `dir` in 3 bits. The hash/encoding used for the activity buffer is `(gene << 7) | (mag << 3) | dir` — see `GENE_INDEX` in `World.m:13`.

## 5. The update loop

`-[World update]` (`World.m:118–158`), per tick:

1. If activity collection is on, bump the activity step counter.
2. Clear `population`, `morgue`, `maternity`.
3. Iterate `[bugs shuffledArray]` (random permutation of all living bugs):
   1. If `bug.food ≤ 0`: remove bug from the cell, move to `morgue`, `lifespan += bug.age`, continue.
   2. If `bug.food > reproductionFood`: call `doReproduceWithMutationRate:` — parent's food is halved, child is placed at the parent's current `(row, col)` via `place:atRow:andCol:`. Child goes into `bugs` and `maternity`, `population++`.
   3. `population++` for the parent.
   4. Call `-updateBug:atRow:column:`:
      - If the cell has food: `doEat:eatAmount` (food is **not** consumed from the cell).
      - Else: `doDigest:-movementCost`. Since `movementCost` is constrained to `≤ 0`, `-movementCost ≥ 0` is subtracted from the bug's food — the bug loses `|movementCost|` per off-food step.
      - `age++`.
      - Compute the 5-bit gene index from the neighborhood.
      - Look up `BugMovement move = bug.genes[gene]`.
      - Call `place:atRow:(row+move.y) andCol:(col+move.x)` to move the bug.
      - If activity logging is on, increment `activity[GENE_INDEX(step, gene, mag, dir)]`.
4. `ticks++`.

Place-on-occupied behaviour (`-[World place:atRow:andCol:]`, `World.m:225–237`): if the destination cell is occupied, the bug is placed recursively at `(row + random()%3 - 1, col + random()%3 - 1)`, i.e. a random Moore neighbor of the intended destination — repeated until an empty cell is found. This can in principle be slow in dense populations, and deterministic reproducibility is hard to achieve because of it.

Note: the bug is asked to move **every** tick. "Stay put" is not in the action alphabet, and on-food bugs still move — but after eating. The only way a bug stays on a food cell long enough to refill its food is for its gene for that particular local configuration to encode a displacement that happens to loop back quickly, or for the next cell to also be food.

## 6. Meta-parameters

### 6.1 Exposed in the API (`PackardBugs.plist`)

| Name                | Type    | Min   | Max   | Default | Where it is stored | Meaning |
|---------------------|---------|-------|-------|---------|--------------------|---------|
| `world`             | Bitmap  | —     | —     | —       | `BugsController.foodImage` → `World.foodImage` | PNG/TIFF whose brightness defines the food field. World dimensions follow the image. |
| `populationDensity` | Float   | 0.01  | 0.99  | 0.2     | `BugsController.populationDensity`             | Probability that each cell is seeded with a bug at reset. |
| `mutationRate`      | Float   | 0     | 1     | 0.5     | `World.mutationRate`                           | **Per-gene, per-reproduction probability that the child's gene slot is replaced by a fresh random gene** (see §7). |
| `reproductionFood`  | Integer | 2     | 40    | 20      | `World.reproductionFood`                       | Food threshold above which a bug reproduces on its next turn. |
| `movementCost`      | Integer | -5    | 0     | -1      | `World.movementCost`                           | Food change per step when the bug is **off** a food cell. The value is *negative*; the bug loses `|movementCost|` each off-food tick. |
| `eatAmount`         | Integer | 0     | 5     | 2       | `World.eatAmount`                              | Food change per step when the bug is **on** a food cell. |

Default values shown here are those declared in the plist; a handful of fields have slightly different hard-coded defaults in `World` init (`mutationRate=0.5`, `reproductionFood=20`, `movementCost=-1`, `eatAmount=1`), which are overwritten from the configuration dict when the plugin is instantiated from the plist.

### 6.2 Hard-coded constants

Cannot be changed without code edits:

| Constant | Value | Location | Meaning |
|----------|-------|----------|---------|
| `INITIAL_FOOD` | 10 | `Bug.m:11` | Food on every freshly-constructed `Bug` (the `-init` path used for `seedBugsWithDensity:`). |
| Genome size | 32 genes | `Bug.h`, `Bug.m` | One `BugMovement` per possible 5-bit neighborhood. |
| Neighborhood | 5 cells (VN + self) | `World.m:160–185` | Four axial neighbors plus the bug's own cell. |
| Magnitude range | 1..15 | `Bug.m:103` | Zero stride is not in the alphabet. |
| Default world size | 100 × 100 | `World.m:38` | When no food image is provided. |
| Food threshold from image | brightness < 0.5 | `World.m:93` | Dark pixels become food. |
| `activityDelta` | 10 ticks | `World.m:267` | Width of each activity-bucket window when `collectActivity=YES`. |
| `GENOME_SIZE` for activity | 32·128 = 4096 | `World.m:11` | Per-step activity slots: `gene(5) · dir(3) · mag(4)`. |
| Reproduction split | `food / 2` | `Bug.m:67` | Integer division; one unit may be lost. |
| Collision bump | `±1` per axis | `World.m:231` | Random Moore-neighbor bump recursion to resolve occupancy. |

## 7. Meaning of the scales (especially mutation)

The mutation rate is **per gene slot**, not per bug.

From `-[Bug initWithFood:andGenes:mutationRate:]` (`Bug.m:34–52`):

```objc
for (i = 0; i < 32; i++) {
    if ((float)random() / INT_MAX < mutationRate) {
        genes[i] = [self randomGene];
    } else {
        genes[i] = myGenes[i];
    }
}
```

Each of the 32 gene slots is independently decided. Consequences:

- `mutationRate = 0`: child is an exact copy of the parent genome.
- `mutationRate = 1`: every gene is replaced by a fresh uniform random draw from the 120-element action alphabet; the child is essentially an unrelated random bug.
- The expected number of mutated genes per birth is `32 · mutationRate`.
- The probability a child has at least one mutation is `1 − (1 − mutationRate)^32`. At the default `mutationRate = 0.5` this is `1 − 2⁻³² ≈ 1` (virtually all children differ from their parent in many slots). Realistic evolutionary regimes will want much smaller values — a `mutationRate = 0.01` gives an expected `0.32` mutations per birth, which is closer to what the Bedau–Packard paper uses.
- Mutation is "hard reset" of a slot, not a perturbation: the old move is discarded. There is no notion of bit-flip or neighbourhood move.

Note also that "random" here is `random()` (the libc PRNG). There is no seeded RNG, and the plugin does not expose a seed parameter — runs are not reproducible.

Other scales:

- `populationDensity` is the per-cell Bernoulli probability at reset. Expected initial population = `width · height · populationDensity`.
- `reproductionFood` is an exclusive threshold: reproduction fires when `food > reproductionFood`.
- `movementCost` ≤ 0: the value is the **food delta** per off-food step, so -1 means "−1 food per step off food". UI labels it "ΔEnergy off food" accordingly.
- `eatAmount` ≥ 0: the food delta per on-food step ("ΔEnergy on food"). This is only added if the bug is on a food cell; food is not a stock, so this is essentially a per-tick subsidy while standing on food.

## 8. Activity tracking

Activity logging is optional (`-[BugsController setCollectActivity:]`), and is the nearest thing the codebase has to a Bedau–Packard "evolutionary activity" probe. When enabled:

- `activity` is a `long *` buffer of shape `[activitySize × GENOME_SIZE]` with `GENOME_SIZE = 4096`.
- Each simulation tick, for each bug's move, `activity[step * 4096 + ((gene<<7) | (mag<<3) | dir)] += 1`.
- `step = ticks / activityDelta` (default `activityDelta = 10`), so each "row" aggregates 10 ticks of usage.
- The buffer doubles when `step >= activitySize`, and the new bucket starts as a **copy** of the previous one (the counts are cumulative by bucket).
- `-exportActivity:` writes a CSV with one row per activity bucket, one column per `(gene, dir, mag)` triple (32·8·15 = 3840 non-empty columns; the `mag=0` slot is written but always 0).

This is **(input, output, time)** activity — it tells you how often each LUT entry maps to each output, cumulatively — not the Bedau–Packard genotype activity (which is per-genotype cumulative reproductive success). There is currently no genotype-level activity, no q_activity, no food_bug probe, and no online aggregate other than `BugsStatistics`' population / births / deaths / ages.

Other statistics currently collected (`BugsStatistics.m`):

- `population` = `|bugs|`
- `births` = `|maternity|` (for last tick only)
- `deaths` = `|morgue|` (for last tick only)
- `averageAge` = mean age over living bugs
- `mortalityAge` = mean age of bugs in the morgue (for last tick)

There is commented-out code for per-gene presence counting across the population, but it is disabled.

## 9. Known quirks and sharp edges

- The `randomGene` distribution is **not uniform over the 120 moves**. The axial moves (diag=0) are produced twice as often as they would be if drawn uniformly from the 8-direction circle, because for `diag=0` the quadrant still determines direction but the two "sides" of each axis collapse. Specifically: axial `(+m,0)` comes only from quad=0,diag=0, but the pair `(diag=0, quad=0)` has probability 1/8, same as any `(diag, quad)` pair. So axial and diagonal moves of a given magnitude each have probability 1/8 — *that* part is uniform across the 8 directions. The magnitude, by contrast, is uniform over 1..15. So the *gene distribution is uniform over 120 moves* after all — ignore this quirk, but read the code carefully before porting.
- Integer-truncated `food/2` on reproduction loses one unit when `food` is odd. Over many reproductions this is a small energy leak out of the ecosystem.
- Occupancy resolution in `place:atRow:andCol:` is a recursive random bump with no bounded depth and no "give up" path. Under extreme crowding this can recurse many times. No stack overflow has been observed, but if you port to a language without generous stack, convert to an iterative bump.
- The RNG (`random()`) is not seeded deterministically and is not exposed to the API — runs are not reproducible.
- Bugs cannot sense other bugs. The input space is only food; collisions only manifest as being bumped on placement.
- "Stay" is not an action — every bug moves each tick. A bug sitting on food still leaves the cell every tick unless its gene for the current neighborhood happens to return a vector that loops back soon.
- Food is a binary bit per cell and is never consumed or regrown. A single food tile feeds arbitrarily many bugs arbitrarily often.
- Population bookkeeping in `-update` re-counts `population` from scratch each tick by iterating living bugs; `births`/`deaths` exposed in `BugsStatistics` reflect only the most recent tick (they are not cumulative totals). `World` also exposes `lifespan` (cumulative sum of ages-at-death) but never exposes it via the statistics plist.
- `BugsStatistics.updateStatistics` is KVO-triggered on `world.ticks` — reads world state without a lock. Safe as long as updates run on the same thread (which they currently do).
