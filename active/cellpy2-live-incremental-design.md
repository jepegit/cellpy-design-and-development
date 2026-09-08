# Live / incremental refresh — design (C2, #164) → 2.2

Status: **design only** (2026-07-27; **substrate re-checked 2026-09-08**, see §2a). Deferred out of 2.1 execution; targets **2.2**.
Owning issue: [#164](https://github.com/jepegit/cellpy/issues/164) "Allow for `c.update()`".
Context: [utils plan wave 4 / G10](../archive/redesigns/cellpy2-utils-migration-plan.md), architecture
plan [§5.6](../roadmap/cellpy2-architecture-plan.md) (the optional `SupportsIncrementalLoad`
protocol).

## 1. Goal

Poll a **running** (still-growing) test file and refresh a loaded cell's
`raw` / `steps` / `summary` **incrementally** — reading only the new rows, not
re-loading and re-summarizing the whole file. Ergonomic surface:

```python
c = cellpy.get("running_test.res")   # initial load
...                                  # test keeps running, file grows
c.update()                           # append only-new rows, refresh derived frames
```

plus a batch-level live refresh (`b.update(live=True)` / a poll loop).

## 2. Current state (accurate as of 2.1)

**The core engine already has the incremental primitives** — the gap analysis
(G10) was right that the ROADMAP marks this DONE:

- `cellpycore.merge.update_data(data, new_raw, *, schema, nom_cap_abs, partition_col=…)`
  — trims the overlap at `source_datapoint_num` (falls back to `datapoint_num`),
  refreshes only the affected step rows via `make_step_table(from_data_point=…)`,
  and rebuilds the per-cycle summary on the combined frames. Returns a new `Data`.
- `cellpycore.merge.merge_data(left, right, …)` — full concat/renumber (campaign merge).
- `CellpyCellCore.update_core_data(data, new_raw, …)` and `.merge_core_data(left, right, …)`
  — schema-bound wrappers. cellpy's `CellpyCell` inherits both.
- cellpy already exposes **campaign** merge: `CellpyCell.merge(cells, mode="campaign", …)`
  (cellreader.py) → core `merge_core_data`.

**What is missing is entirely app-side:**

1. **No loader can return "only the new rows."** Every loader does a full read.
   There is no capability to fetch rows *since a marker* (byte offset / last
   `source_datapoint_num` / row count).
2. **No `CellpyCell.update()`** wiring loader → `update_core_data`.
3. `utils/live.py` (10-line `warnings.warn("to be implemented")` stub) and
   `utils/processor.py` (a hard-coded-path `__main__` benchmark scratch) are
   **not real implementations** — nothing to "rebuild," they are green-field.

## 2a. What the `v2.1.x` patch stream added (checked 2026-09-08)

Still true as of v2.1.3.post3: no `load_since`, no `CellpyCell.update()`, `live.py` and
`processor.py` unchanged. But the app-side pieces §4–§6 lean on have moved:

| Now exists | Use in this design |
|---|---|
| `SourcePreference.NEWEST` + `CellpyCell.check_file_ids(rawfiles, cellpyfile)` (#825) | §4 step 1 change-detect = this FileID size+mtime check; do not add a parallel mechanism |
| `CellpyCell.refresh_after(fields)` + `SUMMARY_META_DEPENDENCIES` (#846) | §4 step 3 "app-side derived refresh" — extend this hook rather than adding a second refresh path |
| Orchestrated `batch.load` → `Batch.update()` with `force_recalc`, progress callbacks / tqdm (#822, #825, #916), `executor="processes"` persist fixed (#920) | §6 `b.update(live=True)` / `b.poll()` = a loop over `Batch.update()` + the existing runner; the tick UI uses the existing `on_progress` callback |
| Journal JSON `version` field, missing = 1 (#1000) | If per-cell load markers are persisted in the journal (batch-level poll state), that is the first bump to journal **v2** — mechanism ready |
| Atomic `.cellpy` writes (#845) | A poll loop saving each tick cannot corrupt the file mid-write |
| `batch_tools/batch_core.py` **deleted**; `batch/store.py` uses `removeprefix` | §5 lstrip fix is **obsolete** |
| Consumers: `cellpy-simple-gui` (desktop) and `cellpy-mcp` (agent tools, #840) | Both want "has the test grown → refresh" without a batch: ship `c.update() -> bool` (L3) as a callable surface before L5 |

## 3. The missing protocol — `SupportsIncrementalLoad`

An **optional** loader capability (architecture §5.6): a second protocol beside
the normal full-read loader, advertised by loaders whose source can be re-read
cheaply from a marker (res/sqlite by `datapoint`/rowid; text/csv by byte offset).

```python
@runtime_checkable
class SupportsIncrementalLoad(Protocol):
    def load_since(self, source, marker: LoadMarker | None) -> IncrementalChunk:
        """Return rows appended since `marker` (all rows if marker is None),
        plus the new marker to store for the next call."""

@dataclass(frozen=True)
class LoadMarker:
    # whichever the source supports; update_data keys on source_datapoint_num
    last_source_datapoint_num: int | None = None
    byte_offset: int | None = None
    row_count: int | None = None

@dataclass
class IncrementalChunk:
    new_raw: "DataFrame"     # native-schema raw rows (may overlap the tail)
    marker: LoadMarker       # marker to persist for the next poll
    complete: bool = False   # loader thinks the test has ended (optional hint)
```

Marker semantics deliberately match `update_data`'s contract: `new_raw` **may
overlap** the tail of the existing `raw` (the loader can re-read the last
partition safely); `update_data` trims the overlap on `source_datapoint_num`.
Loaders that cannot do cheap partial reads simply do **not** implement the
protocol — `update()` then degrades to a full reload + `merge`/replace.

## 4. `CellpyCell.update()` flow

```
c.update():
  1. change-detect: has the source grown? (FileID/fid: size + mtime; or a cheap
     loader "tip marker" call). If unchanged → no-op.
  2. if loader isinstance SupportsIncrementalLoad and c has a stored marker:
        chunk = loader.load_since(source, c._load_marker)
        c.data = c.update_core_data(c.data, chunk.new_raw, nom_cap_abs=…)
        c._load_marker = chunk.marker
     else:  # no protocol / no marker / marker rejected
        full reload into a fresh Data; replace (or merge if same test_id)
  3. re-run app-side derived refreshes that are not in core update_data
     (e.g. ICA caches) if/when present.
```

`update_data` raises `ValueError` if `new_raw` starts at/before the existing
range (a full reload) or if multiple `test_id`s appear — `update()` catches this
and falls back to a full reload. Store `_load_marker` on the cell (and persist it
into the cellpy file so `update()` survives a save/load round-trip).

## 5. App-layer rebuild (`live.py` / `processor.py`)

- `live.py` — the thin consumer: a `poll(cell_or_path, interval, on_update=…,
  stop_when_complete=True)` loop calling `c.update()`, plus the single-shot
  `c.update()` path above. This is where the streaming/live use case lives; keep
  it small (the heavy lifting is core `update_data`).
- `processor.py` — the parallel-load helper (currently scratch): either fold its
  one real idea (thread-pool `cellpy.get` fan-out) into `batch.runner`'s existing
  `executor="threads"` path (which already exists post-A) and **delete**
  `processor.py`, or keep a documented thin wrapper. Recommendation: delete —
  `batch.runner` already owns parallel load.
- ~~Fix the label-mangling bug (`batch_core.py:180` `lstrip` → `removeprefix`)~~ —
  **done / obsolete (2026-09-08):** `batch_tools/batch_core.py` no longer exists and
  `batch/store.py` documents the `removeprefix` fix.

## 6. Batch live-refresh

`b.update(live=True)` iterates the journal cells calling `c.update()`; a
`b.poll(interval=…, until=…)` convenience wraps the loop and re-runs the
collectors/report on each tick. Rides entirely on §4 — no new core needs.

## 7. Work breakdown (for 2.2)

| # | Where | Item |
|---|---|---|
| 1 | **cellpy** (decided, stage5 §2) | `SupportsIncrementalLoad` protocol + `LoadMarker`/`IncrementalChunk` types — cellpy-hosted; core never sees it |
| 2 | cellpy loaders | Implement `load_since` for the cheap-partial sources first (arbin_res / arbin_sql / neware_txt / maccor_txt); others stay full-read |
| 3 | cellpy `CellpyCell` | `.update()` (change-detect + protocol/fallback), `_load_marker` state + persistence in the cellpy file |
| 4 | cellpy utils | `live.py` poll loop; delete/retire `processor.py` (~~`batch_core.py` lstrip~~ — gone) |
| 5 | cellpy batch | `b.update(live=True)` / `b.poll(...)` |
| 6 | tests | incremental-refresh smoke tests: load a truncated file, append the tail, assert `update()` == full-load summary (a golden equality test — the strongest correctness net) |

The **golden equality test** (item 6) is the anchor: *incremental refresh of a
split file must equal a full load of the whole file.* Everything else is
ergonomics around that invariant.

## 8. Why 2.2, not 2.1

The core primitives exist, but the loader protocol + per-loader `load_since` +
cell state persistence is a **cross-package** feature (cellpycore protocol,
multiple cellpy loaders, file-format persistence) with a real correctness
surface (overlap trimming, marker persistence, test-end detection). It does not
gate any 2.1 deliverable and is safely additive later. 2.1 closes Epic C with
C1 (`ocv_rlx`) done and C2 captured here as a ready-to-execute 2.2 design.
