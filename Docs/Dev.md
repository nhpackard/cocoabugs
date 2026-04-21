# Dev log — bugs-python branch

Chronological record of multi-step changes made to the Python port of Packard
Bugs on branch `bugs-python`. Each entry pairs a task with a short summary of
the resulting change in the tree. Newest at the bottom.

## Session — 2026-04-20

Original user request (five feature items + two add-ons):

1. Double the width of probe windows — the Python-backed sim is fast enough
   that the extra horizontal detail is cheap.
2. Give access to the original CocoaBugs food PNG templates.
3. Switch LUT neighborhood from Von Neumann (5-bit, 32 genes) to Moore
   (9-bit, 512 genes), accepting ~16× genome memory.
4. Add a "bug coloring" probe modeled after the CocoaBugs
   `BugsColoringWindowController`: user toggles a 3×3 neighborhood template
   that selects a LUT index; probe plots the histogram of per-bug move
   outputs at that index.
5. Add a scalar time-series probe window with multiple colored traces
   (population, total food-in-bugs).

Add-ons agreed during the session:

6. Fix the recipe format so the VN→Moore change can't silently load
   incompatible runs.
7. Update documentation for all of the above.

Implementation order (chosen to let each task smoke-test against the previous
one): 1 → 2 → 5 → 3 → 4 → 6 → 7.

### Task #6 — Double `PROBE_W` (512 → 1024)

- `Bugs/python/controls.py`: `PROBE_W = 1024`.
- `Bugs/python/sdl_worker.py`: `PROBE_W = 1024`; probe windows stack in the
  same column with the new width.
- All existing probes (`activity`, `q_activity`) re-verified at the new
  width.

### Task #7 — PNG food templates

- `Bugs/python/bugs_py.py`:
  - Added `_REPO_ROOT` and `_FOOD_TEMPLATES` mapping built-in names
    (`stripes`, `r-pentomino`, `big_box`, `3x3_boxes`, `empty_boxes`) to the
    `CocoaBugs/*.png` files shipped with the original distribution.
  - New `food_templates()` listing and `_load_food_png(path_or_name, N)`
    using Pillow (auto-resizes to `N×N`).
  - `state()` extended with `food_source='template'` + `template=` kwarg, and
    `brightness=` now accepts a string path (loaded as PNG).
- Requires `pip install pillow`.

### Task #8 — Scalar time-series probe

- `Bugs/python/controls.py`:
  - `_AVAILABLE_PROBES['ts']`.
  - Traces declared as `_TS_TRACES = ('population', 'food_bug')` with colors
    `_TS_COLORS = (0xFF44DD44, 0xFFFFAA22)`.
  - Shared-memory block `ts_shm = cursor:int32 + 2 × PROBE_W × float32`.
  - `_record_probes()` samples `get_population()` and `get_food_bug()` each
    tick and advances the ring cursor.
- `Bugs/python/sdl_worker.py`:
  - Argparse `--ts=<shm_name>`.
  - `_render_ts(dst, trace_bufs, cursor, global_max)` draws log-Y scrolling
    polylines with a shared Y scale.
- On-save handler writes `probe_ts.png`; `<| ts |>` buttons halve/double the
  shared Y scale.

### Task #9 — Moore neighborhood (LUT 32 → 512)

- `Bugs/C/bugs.h`: `N_GENES 32` → `N_GENES 512` (with `2^9` comment).
  Docstring rewritten with the 9-bit visual-reading-order bit table
  (NW=bit 0 … SE=bit 8), y increasing upward.
- `Bugs/C/bugs.c`: `neighborhood_gene()` expanded from 5 cells to 9 cells,
  bit masks 1, 2, 4, 8, 16, 32, 64, 128, 256 for NW, N, NE, W, C, E, SW, S,
  SE.
- `Bugs/python/bugs_py.py`: new module constants `NBHD = 'moore'`,
  `NBHD_BITS = 9`, `N_GENES = 1 << NBHD_BITS`. Docstring updated.
- Smoke-tested: grids of 64–256 run at plausible populations; genome memory
  ≈16× previous, still negligible at these N.

### Task #10 — Fix recipes for Moore change

- Recipe format bumped to `version: 2`, with new `nbhd` and `n_genes`
  fields.
- `import_run()` raises `ValueError` on nbhd mismatch ("genome structure
  differs") so v1 / VN recipes can't silently replay against the Moore
  runtime.
- Round-trip test + tampered-recipe test both pass.

### Task #11 — Bug-coloring probe with template

- `Bugs/C/bugs.h`, `bugs.c`:
  - Added `void bugs_bug_coloring_hist(int gene_idx, int32_t *hist_out);`.
  - Fills a 31×31 `int32` buffer with the per-bug `(dx, dy)` distribution
    from `genome.genes[gene_idx]` across the live population; out-of-range
    `gene_idx` yields an all-zero histogram.
- `Bugs/python/bugs_py.py`: ctypes binding + `Bugs.bug_coloring_hist(idx)`
  returning an `(31, 31) int32` numpy array.
- `Bugs/python/controls.py`:
  - `_AVAILABLE_PROBES['coloring']`, labels `NW … SE`, shared-memory block
    `coloring_shm = lut_idx:int32 + 31*31 × int32`.
  - 3×3 `GridBox` of `ToggleButton`s; toggling bits updates `coloring_idx[0]`
    (bit-packed in the same NW..SE order as the neighborhood table).
  - `_record_probes()` calls `bug_coloring_hist(coloring_idx[0])` each tick
    and writes into shm.
- `Bugs/python/sdl_worker.py`:
  - Argparse `--coloring=<shm_name>`.
  - `_render_coloring(dst, hist, lut_idx)` draws a log1p-scaled grayscale
    31×31 grid, a center crosshair at `(0, 0)`, and a miniature inset of the
    3×3 template so you can read back which bits are on.
- Invariant `hist.sum() == population` verified; sum over all 512 indices
  equals `pop × 512`.

### Task #12 — Documentation

- `Bugs/Bugs.md`:
  - Extensions list now covers Moore LUT, `ts` + `coloring` probes, PNG
    templates, and recipe v2.
  - Model section: 5-bit VN table replaced with the 9-bit Moore (NW..SE)
    table; step-list references `genes[9-bit-neighborhood]`.
  - `state(...)` signature adds `food_source='template'` + string
    `brightness=` paths.
  - New "Built-in templates" subsection lists the bundled PNG names.
  - Probes section adds `ts` (population, food-in-bugs) and `coloring`
    (3×3 template → 31×31 move histogram) and notes `PROBE_W = 1024`.
  - Recipes section documents v2 format and the `ValueError` guard.
- `Bugs/test.ipynb`:
  - §9 Moore / bug-coloring histogram with assertion
    `hist.sum() == pop` and cross-check
    `sum over all LUT indices == pop × N_GENES`.
  - §10 `food_templates()` + `run_with_controls` enabling all four probes
    on the `r-pentomino` template.
  - §11 recipe v2 round-trip + tampered-`nbhd` recipe raising `ValueError`.
- `Docs/Dev.md`: this file.
