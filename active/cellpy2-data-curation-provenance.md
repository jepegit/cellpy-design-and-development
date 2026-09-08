# Data curation & provenance — original vs processed (design)

Status: **design only** (2026-07-28). Post-2.2 candidate; **sequence after / with the
live-incremental design** (they share `raw` + marker + persistence machinery).
Owning issue: [#206](https://github.com/jepegit/cellpy/issues/206) "Original vs processed
data". Related: [#164](https://github.com/jepegit/cellpy/issues/164) (`c.update()`,
[live-incremental design](cellpy2-live-incremental-design.md)); architecture plan
[§3 / §4 patterns](../roadmap/cellpy2-architecture-plan.md).

> **Note (2026-09-08):** since v2.1.4 ([#989](https://github.com/jepegit/cellpy/issues/989))
> vendor capacity/energy columns that do not restart at 0 each cycle are **rebased on load**
> (with a `UserWarning`). Proposed classification for this design: the rebase is part of
> **ingestion** (tester bookkeeping, like `harmonize()`'s reset-granularity normalization),
> so "pristine raw" = *post-rebase* raw; it is **not** the first cleaning-recipe step.
> Registered as Δ8 in the architecture plan §7. Confirm when this design is scheduled.

## 1. Goal

Let a user clean / interpolate / downsample a cell **without destroying the pristine raw
measurement**, and keep the modification **reproducible and reversible**. Target surface
(reconciling #206 with the standard patterns):

```python
c.is_original()              # True if no curation applied
c = c.clean.remove_spikes(threshold=5.0)   # append a step, return new object
c.history()                  # the ordered, human-readable recipe
c = c.revert_to_original()   # drop the recipe; back to ground truth
c.save("cell.cellpy")                       # persists raw + recipe (+ derived cache)
c.save("cell_ed001.cellpy", only_processed=True)
```

## 2. Current state (accurate as of 2.1) — what already exists

cellpy 2 **already implements the sound half** of the common "raw vs processed" advice —
do not rebuild it:

- **Multi-table model, raw is ground truth.** `Data` holds `raw` (high-res) + `steps` +
  `summary`; `steps`/`summary` are **derived** by `make_step_table` / `make_summary` (pure
  functions in cellpycore). There is no column-parallel `voltage_raw`/`voltage` scheme, and
  none is wanted — it breaks under aggregation and doubles the schema (keys-in-columns law).
- **Multi-group container on disk.** v9 `.cellpy` is a zip of `raw.parquet`, `steps.parquet`,
  `summary.parquet`, `fid.parquet` + `meta.json` (`readers/cellpy_file/format.py`). This is
  already the "separate tables per state" layout; a Delta-Lake-style transaction-log
  directory is the wrong tool (heavy `delta-rs` dep vs the lean budget, breaks the single-file
  ergonomic, lakehouse concurrency cellpy does not have).
- **Immutable idiom.** Structural edits already return **new** objects (`c = c.drop_from(...)`
  / `split` / `split_many`, `readers/slicing.py`), matching functional-core/imperative-shell.
- **Source identity.** The `fid` table (paths + size + mtime) is the file-identity record;
  it is also the change-detector `c.update()` keys on.

**What is missing:** cleaning of `raw` *itself* (spikes, interpolation, downsample) with
(a) the pristine original preserved, (b) a reproducible/reversible record of what was done,
and (c) a defined interaction with `c.update()`. Today's cleaning is either **structural**
(drop cycles → new object) or **downstream** (`remove_outliers_from_summary_*` on the derived
summary, `utils/helpers.py`) — never a provenance-tracked edit of raw.

## 3. Design — raw pristine, curation as a serializable recipe

The right spine (not dual columns, not Delta Lake): **keep `raw` immutable and model curation
as an ordered, declarative, serializable recipe** applied *between* raw and the derived frames.

```
raw (pristine)  ──apply(recipe)──►  raw_working  ──make_step_table/make_summary──►  steps/summary
     ▲ ground truth, never mutated       ▲ materialized lazily / cached      (derived, as today)
```

- `derived = f(apply(raw, recipe), config)` — a one-line generalization of today's
  `derived = f(raw, config)`. If the recipe is empty, behaviour is identical to now
  (`is_original() == True`).
- **Recipe = list of declarative ops**, each `{op, params, ts, cellpy_version}` — matches the
  "declarative configuration over code" pattern (like loader declarations), so it is
  trivially serializable and inspectable. Ops call the existing cleaning primitives; the
  recipe never stores lambdas or point-arrays.
- **Lazy/cached materialization.** `raw_working` is computed on demand (polars lazy) and
  cached; only recomputed when the recipe changes. `c.raw` returns the working frame;
  `c.original().raw` (or `c.raw_original`) returns the pristine one.

### 3.1 The op taxonomy — the crux for `c.update()`

Each recipe op declares how it re-applies when new raw rows arrive (via `update()` or a
re-load). This is the design's key contribution and mirrors `update_data`'s overlap-trim:

| Kind | Example | Re-application semantics |
|---|---|---|
| **Parametric** (pure fn of the frame) | `remove_spikes(threshold=5.0)`, `interpolate(...)`, `downsample(every="1s")` | **Re-evaluated over the whole (grown) frame.** Deterministic; safe to replay after append. |
| **Pinned** (positional/manual) | drop points 100–200, mask cycle 7 | **Anchored to `source_datapoint_num`**, applied once, preserved across append (new rows unaffected). |

So `c.update()` appends new raw (core `update_data`, overlap-trimmed), then **replays the
recipe**: parametric ops re-run on the combined frame, pinned ops stay anchored. That keeps
the invariant *processed = apply(recipe, raw)* true at every tip. (#206 asks this question
explicitly; the external brainstorm did not see it.)

## 4. Persistence — extend the v9 container, don't replace it

Additive, backward-compatible (old readers ignore the extra member; its absence ⇒
`is_original`). **No new format version required.**

```
cell.cellpy  (zip)
├── raw.parquet        # PRISTINE ground truth (unchanged meaning)
├── steps.parquet      # derived from raw_working (as today)
├── summary.parquet    # derived from raw_working (as today)
├── fid.parquet        # source identity (unchanged)
├── meta.json          # + a `curation` block: recipe ops + raw content-hash
└── raw_working.parquet # OPTIONAL cache of the processed raw (recomputable; skip on save_slim)
```

- `meta.json.curation = { recipe: [...], raw_sha256: "…", original: bool }`. The raw hash
  gives #206's "compare processed vs actual" and detects external tampering; it complements
  `fid` (source-file identity) with **content** identity. Owned by `meta_archive`.
- `save(only_original=True)` writes raw + empty recipe; `save(only_processed=True, ...)` writes
  raw_working *as* raw with the recipe baked in and flagged (a deliberate, one-way export —
  #206's `_ed001` file). Default save keeps both recoverable.

## 5. Work breakdown (post-2.2)

| # | Where | Item |
|---|---|---|
| 1 | cellpy | `CurationRecipe` model + op registry (declarative, serializable); parametric/pinned taxonomy |
| 2 | cellpy | `c.clean.*` façade appending ops (immutable, returns new cell); wire existing primitives (spike/outlier/interp/downsample) as ops |
| 3 | cellpy | `raw_working` lazy materialization + cache; `c.raw` vs `c.original().raw`; `is_original()` / `revert_to_original()` / `history()` |
| 4 | cellpy_file | `meta.json.curation` block + raw content-hash; `raw_working` optional cache member; `save(only_original=/only_processed=)` |
| 5 | cellpy | `update()` recipe-replay (parametric re-run + pinned-by-`source_datapoint_num`) — the join with #164 |
| 6 | tests | **golden invariants**: `apply(recipe, raw)` deterministic; `save→load` round-trips recipe + reproduces derived; `update()`-then-replay == full-load-then-clean |

## 6. Why post-2.2 (not in the 2.2 scope)

It is a **new subsystem** with a real correctness surface (recipe replay, hash/provenance,
save/load round-trip) and it **couples to `c.update()`** — so it should be designed *after*
the live-incremental feature lands (or co-designed), reusing its `raw`/marker/persistence
machinery rather than duplicating it. It is additive and gates nothing in 2.2. Natural
home: a **2.4 headline**, or a 2.3 secondary if kept minimal — decide once #164 ships and the
`update()` semantics are concrete. SPEED-30 (2.3) is orthogonal but, if it lands first, the
`curation` block simply rides the richer header schema.

## 7. Rejected alternatives (recorded)

- **Parallel `*_raw` columns + `is_cleaned` flag.** Doubles the schema, violates the
  one-native-spelling rule, collapses under row-changing aggregation, does not round-trip.
  Arrow zero-copy is real but irrelevant to that cost.
- **Delta Lake / delta-rs on disk.** Heavy dependency against the lean budget; a `.cellpy`
  that is a transaction-log *directory* breaks the single-file ergonomic; lakehouse ACID
  versioning solves a concurrency problem cellpy does not have. Version-on-disk, if wanted,
  belongs **inside** the existing zip container (a recipe in `meta.json`), not a new format.
