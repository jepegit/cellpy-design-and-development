# Stage 4 — cellpy 2.1 implementation plan (issue set)

**Date:** 2026-07-26 · **Status:** ✅ approved — single **v.2.1** release (patch-release
distribution considered and rejected: Epic E is breaking, so a semver-clean 2.1 carries
all of Stage 4). Creating the issue set now.

Stage 4 = **the 2.1 release**: the redesigns and cleanups the 2.0 major deliberately
deferred behind `warn_once` shims. The test 2.0 applied was *"finalize what cannot be
shimmed"*; Stage 4 is the mirror image — **spend the shims**: land the batch/collectors
redesigns their facades were holding a place for, surface the engine features that
already exist, and remove every deprecation registered for `removal: 2.1`.

All issues carry the label **`cellpy2-stage4`** and the existing **`v.2.1`** milestone
(#16 — already open, do **not** create a new one).
Tracking issue: **[#696](https://github.com/jepegit/cellpy/issues/696)** (created 2026-07-26).

**Created issue map (2026-07-26):**
A1–A8 → #697–#704 · B1–B4 → #705–#708 · C1 → #709, **C2 = #164** (rewritten) ·
D1–D3 → #710–#712 · E1–E5 → #713–#717 · G1–G3 → #718–#720 · H1 → #721, **H2 = #655** ·
folded: #345 → A2 (#698), linked: #668 → A1/A8/E4.

### Reconciliation with issues already on the `v.2.1` milestone

The `v.2.1` milestone (#16) already holds work that maps onto these epics — fold, do
**not** duplicate:

| Existing | Maps to | Action |
|---|---|---|
| [#164](https://github.com/jepegit/cellpy/issues/164) "Allow for `c.update()`" | **C2** | This *is* the incremental-refresh feature (load only new data, merge, resummarize). Rewrite #164 as C2 rather than create a new issue. |
| [#345](https://github.com/jepegit/cellpy/issues/345) "batch - read custom json" | **A2** | Journal custom-json readers + file-search-after-read. Fold as an A2 acceptance item / link. |
| [#655](https://github.com/jepegit/cellpy/issues/655) "Test data gaps" | **H2** | Already the carry-over; keep as-is, relabel `cellpy2-stage4`. |
| [#668](https://github.com/jepegit/cellpy/issues/668) "bugs in batch" (v1-follow-up) | **A1/A8** | `b.plot`/`batch_plotters` bugs the redesign deletes; A1's net must cover, A8 closes. Link, don't duplicate. |
| [#73](https://github.com/jepegit/cellpy/issues/73) "GITT/PITT routines" | **C3?** | New analysis routine; **scope decision pending** (§7 Q6) — keep in 2.1 as a C-series analysis issue, or move to v.2.2. |

Not pulled forward: the v.2.2 backlog (#691 filefinder, #687 scp regression, #313 IR
endpoints, #312 CCCV split, #306 Arbin-db export, #270 Biologic mpr v3, #359 discharge-
first) stays on v.2.2 unless explicitly promoted.
Earlier records: [stage3-github-issues.md](stage3-github-issues.md) (tracking
[#575](https://github.com/jepegit/cellpy/issues/575)).

> **Live status:** once issues exist, progress is tracked in one place only — the
> status dashboard in [cellpy2-architecture-plan.md](cellpy2-architecture-plan.md)
> (Stage 4 row) and the `v2.1.0` milestone. Per-arc records live in each owning plan.

---

## 1. Scope decision (agreed 2026-07-26, maintainer)

| In 2.1 | Out (→ 2.2 / parked) |
|---|---|
| **A** Batch v3 redesign (`cellpy/batch/`) | **F** SPEED-30 value+unit+dtype headers (parked 2.2; `units_label()` re-points later) |
| **B** Collectors redesign (`cellpy/collect/`) | UUID / BattINFO-EMMO ontology mapping (metadata OQ6, F10) |
| **C** Utils wave 3 (`ocv_rlx`) + wave 4 (**live/incremental rebuilt on core**) | fsspec/remote beyond ssh; narwhals; per-test `raw_units`; volumetric mode |
| **D** F6 feature menu (test-level summaries, `exclude_step_types`, native-only cols) | |
| **E** Shim removals — all 20 `removal: 2.1` entries + `batch_tools/` | |
| **G** Docs / example-data v9 refresh / migration guide | |
| **H** Housekeeping (F7 core doc-sync, #655 fixture gaps) | |

**Live/incremental verdict (2026-07-26):** *rebuild on core*, not drop — `live.py` /
`processor.py` become thin consumers of core's incremental summarization (ROADMAP:
DONE), realizing the optional `SupportsIncrementalLoad` protocol flagged in
[architecture plan §5.6](cellpy2-architecture-plan.md). "Poll a running test, refresh
steps/summary" is a headline 2.1 feature.

---

## 2. Epic A — Batch v3 ([batch redesign plan](cellpy2-batch-redesign-plan.md))

New package `cellpy/batch/` built alongside the old `utils/batch_tools/`; the beloved
notebook surface survives on a facade. Target ≤2 500 lines (from ~8 100). Fixes two
live bugs: the `str.lstrip` label-mangling (`xenon_cell`→`enon_cell`) and the
shared-by-reference `empty_farm` aliasing.

| Issue | Content | Plan |
|---|---|---|
| A1 | **Phase 0 characterization net**: two-cell batch end-to-end golden (journal→load→update→summaries→QC, *values* not shapes); journal JSON (both shapes) + Excel round-trip tests; grep docs/tutorials/notebooks for `batch.` surface → facade must-keep list | §6 Phase 0 |
| A2 | `journal.py` + `layout.py` — `Journal` model, json/excel readers, legacy JSON loader; `BatchPaths` pure path calc + explicit `ensure_dirs()` (kills the `paginate()` mkdir side effect); pages keys-to-columns | §4.1 |
| A3 | `policy.py` + `resolve_specs()` — `LoadPolicy`/`CellSpec`/`SourcePreference`; the single precedence merge (journal < policy < per-cell), property-tested against today's documented order | §4.2 |
| A4 | `runner.py` + `result.py` + `store.py` — `load_cell(spec)→CellResult`, serial `run()`, `BatchResult` (per-cell outcome/timing/`raise_if_failed()`); `CellStore` Mapping + IPython key completion (removes `x_`/`lstrip`) | §4.3–4.4 |
| A5 | `aggregate.py` + `qc.py` + `outputs.py` — `combine_summaries()→` tidy long frame (cell/group/sub_group); `_check_cell_*`→one QC frame; pure `write_csv/excel/parquet` | §4.5 |
| A6 | `facade.py` + `__init__.py` — `Batch`, single `load()` + `from_journal()`; **golden parity vs old impl** (shared columns, mapped through header renames) | §4.6 |
| A7 | **Phase 2** — `cellpy.utils.batch` re-implemented as shims over `cellpy.batch` (signatures preserved; exotic kwargs→`LoadPolicy` or helpful `DeprecationWarning`); `batch_tools` import-time warnings; published old→new symbol map; repo docs/tutorials/notebooks moved to new API | §6 Phase 2 |
| A8 | **Phase 3** — process-pool executor (results as frames+paths, Windows-pickle-safe); collectors interface handover point; delete dead stubs (`ImpedanceExperiment`, stub engines, `batch_reporters.py`, `OriginLabExporter`, `do2`, `_init_old/_new`) | §6 Phase 3 |

Depends on: native flip (done). Blocks: **B** (needs `batch.aggregate` at A5), **E4**.

## 3. Epic B — Collectors ([collectors redesign plan](cellpy2-collectors-redesign-plan.md))

New package `cellpy/collect/`. Collection-as-a-product with provenance; options
dataclasses replace the three-layer "elevated arguments" merge. Target ~900–1 000
lines. Fixes the **cross-cell cycle-narrowing correctness bug** (each cell's rate
filter silently narrowed selection for all later cells).

| Issue | Content | Plan |
|---|---|---|
| B1 | **Phase 0** characterization: golden collect frames (summary/cycles/ica, two-cell, values) incl. `group_it` + rate-filter cases; encode the cross-cell bug as an **xfail→fix pair**; inventory which elevated args are actually used → options field list | §4 Phase 0 |
| B2 | **Phase 1** pipeline: `options.py`, `cells.py` (per-cell isolation), `summary.py` (port `concat_summaries`, minus deprecated twin), `curves.py`, `ica.py`, `collection.py` (`Collection` + `load_collection`, parquet+csv+meta.json); golden parity gate; **bug fix lands here** | §4 Phase 1 |
| B3 | **Phase 2** class re-based: `collector.py`; old `BatchCollector`+subclass aliases → shims (unknown kwargs raise w/ field list); `standard_gravimetric` → declarative recipe; remove constructor I/O (`parse_units` full-batch load) | §4 Phase 2 |
| B4 | **Phase 3** plotting handover: `Collection.plot()` / `BatchCollector.plot()` → `cellpy.plotting`; save bundle via `plotting.figures` | §4 Phase 3 |

Depends on: **A** (A5 `batch.aggregate`), plotting (done). Blocks: **E4**.

## 4. Epic C — Utils waves 3–4 ([utils plan](cellpy2-utils-migration-plan.md) §2–3)

| Issue | Content | Plan |
|---|---|---|
| C1 | ✅ **done** (#709). **Wave 3** `ocv_rlx` refactor: renamed the `self.data`-holds-a-`CellpyCell` trap → `.cell`; header access → native `schema.*`; scipy fitting stays app-side | wave 3 |
| C2 | **→ 2.2** (design captured). **Wave 4** live/incremental. The core primitives already exist (`update_core_data`/`merge_core_data`); the missing pieces are the `SupportsIncrementalLoad` loader protocol + `CellpyCell.update()` + `live.py` rebuild — a cross-package feature. Full design: [cellpy2-live-incremental-design.md](cellpy2-live-incremental-design.md) | wave 4, G10 |

(ICA — utils wave 3's other half — already shipped in 2.0, #566.)

**Decision (2026-07-27):** C2 deferred to 2.2 — it depends on a loader-side
incremental protocol + per-loader `load_since` that doesn't exist yet (the core
*engine* primitives do). `#164` retargeted to the 2.2 milestone. Epic C closes
for 2.1 with C1 done.

## 5. Epic D — F6 feature menu ([utils plan](cellpy2-utils-migration-plan.md) §1.4/wave 4, [gap analysis F6](cellpy2-plans-gap-analysis.md))

Surface engine capabilities that already exist but never reach the user. Additive;
ride on A/B.

| Issue | Content |
|---|---|
| D1 | `exclude_step_types` (#54) exposed on `make_summary` kwargs (restores the selector-exclusion users lost in core#45) |
| D2 | Test-level summaries (`build_tests_summary`) surfaced in batch reports (`b.tests` summaries) |
| D3 | Native-only report columns (energies, powers, durations, per-direction stats) as opt-in in batch/collectors + plotting |

Consider folding [#303](https://github.com/jepegit/cellpy/issues/303) (cycle-statistics,
moved off the 2.0 milestone) here.

## 6. Epic E — Shim removals ([conventions plan](cellpy2-conventions-plan.md) §3)

The defining obligation of 2.1: every entry in `DEPRECATIONS.md` marked `removal: 2.1`.
A removal is only allowed once its replacement ships — hence E4 trails A/B.

| Issue | Content | Depends |
|---|---|---|
| E1 | Remove plotting shims: `interactive=`→`backend=` (`cycle_info_plot`/`cycles_plot`/`dva_plot`/`ica_plot`/`raw_plot`/`summary_plot`), `xlim/ylim`→`x_range/y_range`, `Batch.plot(backend="seaborn")`, `plotutils.summary_plot_legacy` | — |
| E2 | Remove ICA shims: `ica.Converter`, `dqdv(cycle=/label_direction=/split=/tidy=)`, `dqdv_cycle`/`dqdv_cycles`/`dqdv_np`, the `dq` output column | — |
| E3 | Remove `headers_*` legacy attribute access (→ `c.schema.*`), `make_new_cell` (→ `CellpyCell.vacant`) | — |
| E4 | Remove `utils/batch_tools/` and collectors' elevated-kwarg shims; keep `cellpy.utils.batch`/`utils.collectors` import paths as permanent one-line re-exports | **A7, B3** |
| E5 | **DECISION + execution** — do the **prms shim** and the **property facade** (`c.mass` etc., #§8) also go in 2.1, or extend? Audit `DEPRECATIONS.md` for completeness (these + curve-frame names are named in arch §Stage 4 but not yet in the generated table); regenerate the table | §7 Q4 |

## 7. Epic G / H — Docs, example data, housekeeping

| Issue | Content |
|---|---|
| G1 | Regenerate hosted example data as **v9** (utils plan `example_data.py`) |
| G2 | Docstring-driven API-reference cleanup (docs plan tail) |
| G3 | **Migration guide 2.0→2.1** + release notes: every E removal with old→new, the collectors cross-cell bug-fix before/after, batch symbol map (write last — describes what shipped) |
| H1 | F7 cellpy-core doc-sync pass (~1 h; stale "partly done" statuses) |
| H2 | Carry-over [#655](https://github.com/jepegit/cellpy/issues/655) fixture gaps (non-blocking) |

---

## 8. Sequencing (the DAG)

```
              ┌─ E1 (plotting shims) ─┐
              ├─ E2 (ica shims) ──────┤
 start now ──►├─ E3 (headers/vacant) ─┤
              ├─ C1 (ocv_rlx) ────────┤
              ├─ C2 (live on core) ───┤
              ├─ D1 (exclude_step) ───┤
              └─ E5 decision ─────────┘

 A1 ─► A2 ─► A3 ─► A4 ─► A5 ─► A6 ─► A7 ─► A8
                         │            │
                         ▼            ▼
                   B1 ─► B2 ─► B3 ─► B4        (B needs A5 aggregate; B4 with A8)
                                │  │
                                ▼  ▼
                               E4 (batch_tools + elevated-kwarg removal)
                                │
 D2/D3 ride on A/B ────────────┤
                                ▼
                          G3 (migration guide, last)
```

Constraints (mirroring [stage3 §Sequencing](stage3-github-issues.md)):
1. **A5 before B2** — collectors' `summary.py` builds on `batch.aggregate.combine_summaries`.
2. **A7 + B3 before E4** — cannot remove `batch_tools`/elevated kwargs until the shims exist.
3. **B1 before B2** — the xfail→fix pair fixes the cross-cell bug deliberately.
4. **E5 decided early** — gates whether prms/property-facade removal is in the set.
5. **G3 last** — the migration guide describes what actually shipped.
6. Startable immediately (independent of A/B): **E1, E2, E3, C1, C2, D1, H1, G1**.

**yolo candidates** (additive/alias-guarded, crisp criteria, covered by oracles):
E1, E2, E3, D1, H1. Everything in A/B and E4/E5 is a structural move or a decision — not yolo.

---

## 9. Decisions (resolved 2026-07-26, maintainer)

1. **Batch plotting backends** (batch §4.7): **drop bokeh** (and seaborn, already
   deprecated in 2.0). plotly (primary) + matplotlib (print/CLI) remain; note the bokeh
   drop in release notes since it appears in old tutorials. → **A8/E1**.
2. **`iterate_batches` / `process_batch`** (batch §8 Q2): **docs recipes** over the new
   `load()`, not maintained API — deprecate + re-express in docs. → **A7 shim + G3**.
3. **Package names**: **`cellpy.batch` + `cellpy.collect`** (separate top-level, matching
   `cellpy.plotting`); `utils.batch` / `utils.collectors` stay as permanent re-export
   shims. → **A2+, B2+**.
4. **prms shim + property facade** (E5): **remove the `prms.*` global shim** on the 2.1
   cadence; **extend the `c.mass` property facade past 2.1** (beloved, cheap, low-risk).
   E5 therefore = prms removal + `DEPRECATIONS.md` audit/regenerate; the property facade
   moves to a "kept past 2.1" note, not a removal.
5. **Collections** (collectors §6): parquet default save, hdf5→legacy writer;
   `standard_gravimetric` callable kept as a shim on the 2.1 cadence. → **B2/B3** (settle
   final wording when writing those issues).

**Go-ahead pending.** On approval I create: the tracking issue, the `cellpy2-stage4`
label + `v2.1.0` milestone, then A1–H2 (~24 issues) wired to the tracking issue, and add
the Stage 4 row to the architecture-plan dashboard.
