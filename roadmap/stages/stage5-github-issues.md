# Stage 5 — cellpy 2.2 scope plan (issue set)

**Date:** 2026-07-28 (issue set cut 2026-07-29) · **Status sync:** 2026-09-08 ·
**Status:** 🟡 **issue set cut; execution not started** (0 Stage 5 epic issues closed) —
**revised 2026-09-08 (§11)** against five weeks of `v2.1.x` patch work.
Tracking [#783](https://github.com/jepegit/cellpy/issues/783); scope decisions in §9, revision in §11.
2.1 shipped (v2.1.0 + post1); reactive patch stream through **v2.1.3.post3** (2026-09-05),
**v2.1.4 pending** (`v.2.1.4` milestone 14 closed / 0 open, `HISTORY.md [Unreleased]`) — see §10.

**Created issue map (2026-07-29):** tracking **#783** · label `cellpy2-stage5` · milestone
`v.2.2`. **L** L1–L6 → #779/#780/**#164 (L3)**/#781/#782/#778 · **S** S1–S3 → #313/#312/#359 ·
**I** I1–I4 → #270/#338/#306/#761 · **R** R1–R2 → #687/#691. Deferred: `v.2.3` milestone
created (#73 GITT/PITT moved there; SPEED-30 to follow).

2.2 is a **minor, additive feature release**, the mirror of 2.1. Where 2.0 "finalized what
can't be shimmed" and 2.1 "spent the shims", **2.2 delivers the deferred "complete cellpy 2"
features** from [architecture plan §3](../cellpy2-architecture-plan.md) — the ones parked with a
spot, not forgotten. The 2.2 test is not "can it be shimmed" but **"is it additive and does
it version cleanly?"** The items that would touch a *contract* (the SPEED-30 header format;
a `cycle_mode` convention change) are handled with schema-version + oracle-exception care —
and the heaviest of them (SPEED-30) is deferred to 2.3 to keep 2.2 shippable.

**Reactive capacity — kept deliberately.** A wave of **v2-migration bug reports** is expected
as users move onto the v2 series. Those are handled **out-of-band as `v2.1.x` patch releases
off `master`** (trunk-based; same path as the v2.1.0.post1 docs patch) — *not* folded into
the 2.2 feature line. 2.2 scope is therefore held **lean on purpose**, so the feature release
can absorb slippage and the team keeps headroom for urgent fixes. If an incoming migration
issue turns out to be a feature rather than a bug, it joins 2.2 explicitly; bugs do not.

Coordinating doc: [cellpy2-architecture-plan.md](../cellpy2-architecture-plan.md).
Tracking: **[#783](https://github.com/jepegit/cellpy/issues/783)** (*cellpy 2.2 (Stage 5) — tracking*).
Day-to-day: [`../../CURRENT.md`](../../CURRENT.md).

**Milestone `v.2.2` (2026-09-08: 20 open / 0 closed):** original set
#778/#779/#780/#164/#781/#782 · #313/#312/#359 · #270/#338/#306/#761 · #687/#691 ·
tracking #783 — plus milestone adds **#784** (external metadata sources),
**#352** (initial-OCV batch plot), and (since 2026-08) **#827** (PEC multi-cell csv → **I5**)
and **#888** (reference-performance-test filters → **S4**), see §11. Net of the decisions
below: **#73 → 2.3**, **#761 → pulled into 2.2**. `v.2.3` now also holds **#889** (Fredrik ICA).

---

## 1. Scope (agreed)

| In 2.2 | Deferred → 2.3 | Parked (spot kept) |
|---|---|---|
| **L** Live / incremental refresh (`c.update()`, #164) — **cellpy-only**, design done | **F** SPEED-30 value+unit+dtype versioned headers — the **2.3 structural headline** (§6) | UUID / BattINFO-EMMO ontology mapping (metadata OQ6, F10) |
| **S** Step/summary science: IR-at-endpoints #313, CCCV split #312, discharge-first #359 (on the **current** schema); *(2026-08 add)* RPT filters **#888** | **#73** GITT/PITT — greenfield analysis routine, early-2.3; *(2026-08 add)* **#889** Fredrik ICA | narwhals evaluation (polars plan decision 2) |
| **I** Instruments / IO: Biologic mpr v3 #270, better csv #338, Arbin-db exporter #306, loader robustness **#761**; *(2026-08 add)* PEC multi-cell csv **#827** | | per-test `raw_units`; fsspec beyond ssh; **volumetric mode** |
| **R** Remote paths & discovery: scp SSH-alias regression #687, project-scoped filefinder #691 | | |

**Design-only, post-2.2 (captured, not scheduled):**
- **Data curation & provenance** ([#206](https://github.com/jepegit/cellpy/issues/206)) —
  pristine-raw + serializable-cleaning-recipe subsystem
  ([cellpy2-data-curation-provenance.md](../../active/cellpy2-data-curation-provenance.md)); couples to
  `c.update()` (Epic L) → sequenced **after** L (2.4 headline / 2.3-secondary).
- **External metadata sources** — pluggable `MetadataSource` protocol + adapters (first: the
  BatBase Django/PostgreSQL HTTP API), realizing metadata-plan Step 7
  ([cellpy2-metadata-source-integration.md](../../active/cellpy2-metadata-source-integration.md)); adds
  `CellMeta.uuid`, pairs with BattINFO vocab. Adjacent issue #243. 2.3+.

**Cross-repo (F9):** only the **S-epic engine changes** (IR/CCCV/cycle-mode tools) are
**core-first** (core PR → PyPI → cellpy re-pin). **L needs no new core work** — the
incremental primitive (`update_data`) already ships, and its loader protocol lives in cellpy
(§2). I and R are cellpy-side. So 2.2 has exactly **one** cross-repo dependency (Epic S), not
the three the first draft implied.

---

## 2. Epic L — Live / incremental refresh (#164)

The flagship, fully specified in **[cellpy2-live-incremental-design.md](../../active/cellpy2-live-incremental-design.md)**
(poll a running test; refresh `raw`/`steps`/`summary` reading only new rows). The core
engine primitives already exist (`update_data`/`update_core_data`); the rest is app-side.

**Decision — protocol home = cellpy (not cellpycore).** `SupportsIncrementalLoad` is a
contract *only* between cellpy loaders and `CellpyCell.update()`; core's `update_data` never
sees it (it just takes `new_raw`). Hosting it in cellpy keeps core pure and — with
`update_data` already shipped — makes the **entire L epic cellpy-only, no core release in the
critical path.** This is the main de-risking of the flagship.

| Issue | Where | Content |
|---|---|---|
| L1 | cellpy | `SupportsIncrementalLoad` protocol + `LoadMarker`/`IncrementalChunk` types (arch §5.6), hosted app-side |
| L2 | cellpy loaders | `load_since(source, marker)` for cheap-partial sources first (arbin_res, arbin_sql, neware_txt, maccor_txt); others stay full-read and fall back |
| L3 | cellpy `CellpyCell` | `.update()` — change-detect (FileID size+mtime / tip marker) → protocol path or full-reload fallback; `_load_marker` state **persisted into the cellpy file** |
| L4 | cellpy utils | `live.py` `poll(...)` loop (replaces the stub); **delete `processor.py`** (fold thread-pool into `batch.runner`). ~~fix `batch_core.py:180` `lstrip`~~ — **obsolete** (§11): `batch_tools/batch_core.py` is gone and `batch/store.py` already uses `removeprefix` |
| L5 | cellpy batch | `b.update(live=True)` / `b.poll(interval, until)` — iterate journal cells, re-run collectors/report per tick |
| L6 | tests | **golden equality anchor**: load a truncated file, append the tail, assert `update()` == full-load summary. Write first |

Owning issue **#164** = L3. Internal order: **L6 first** (oracle) → **L1 → L2 → L3 → L4/L5.**

## 3. Epic S — Step/summary science engine

New derived quantities, all **additive columns** on `steps`/`summary`, on the **current
schema** (SPEED-30 deferred, §6). Engine math lands in **cellpycore** (core-first); cellpy
gets a thin surface.

| Issue | Content | Notes |
|---|---|---|
| S1 = **#313** | Internal resistance at start/end of (dis)charge from the rest→step potential jump (Ohm's law): `ir_start`/`ir_end` on `steps`; `ir_{start,end}_{charge,discharge}` on `summary`. Optionally relaxation/OCV-potential columns from the following rest step. | Relates to the **corrected-IR-semantics** note (arch §3); today's `_ir_to_summary()` is Maccor-value-based. Decide default vs opt-in; record any oracle exception. |
| S2 = **#312** | Split single-step CCCV into CC + CV **substeps** (prefer `sub_type`/substep columns, keep the cycler's original designation); compute **CV-share** into `summary`. | Needs a robust CC→CV transition detector (potential plateaus, current decays) tested on noisy files. Ties to the `cellreader.py` "split merged steps" TODO. |
| S3 = **#359** | Full cells that **start with a discharge**: a `cycle_mode`/cycle-counter option (or `update_cycle_counter(...)`) so CE and curve extraction stay correct for commercial cells. | Labelled **`to core`** — cycle-mode polarity is a core-engine contract (`_guard_mixed_cycle_modes`). User-visible convention change → release-note + comparator exception. |
| S4 = **#888** *(added 2026-08-11)* | **Reference-performance-test (RPT) filters**: a structured filter (`ReferencePerformanceTest(...)`: c-rate, temperature-with-slack, step pattern) usable as `filter=` on `get_cap` / summary access, composable with single criteria; bonus: mark RPT cycles on raw/summary plots. | **cellpy-side** on the existing `cellpy.filters` package (`filter_cycles`, summary filters; design in `filters-and-plot-filtering.md`). No core work unless cycle classification needs new step-table columns. Commercial-cell / lifetime-test workflow — pairs with S3. IFE `VLT` filter code is the reference. |

**#73 (GITT/PITT) → 2.3.** Grounding: `docs/examples/05_GITT.md` already demonstrates OCV
*extraction* with existing tools, but there is **no dedicated GITT/PITT routine** in the
package (the diffusion-coefficient analysis is greenfield). Summer-student work (Vilde) can
seed it; integrating research code well is its own effort → early-2.3, doesn't gate 2.2.

Order within S: **S1 + S2 land their summary columns together** (one schema addition, one
migration-guide entry). S3 is independent but shares the CE/convention delta register.

## 4. Epic I — Instruments / IO

Additive loader coverage + one robustness fix. All cellpy-side, behind the two-stage loader
contract ([loader plan](../../archive/foundations/cellpy2-loader-port-and-extraction-plan.md)); each ships a golden
fixture (F8) + `check_loader`.

| Issue | Content | Notes |
|---|---|---|
| I1 = **#270** | Biologic **mpr v3** — adapt `biologics_mpr.py`. Reference: galvani (echemdata/galvani#36). | Distinct from #187 (mpt) / #87. Vendor-`parse()`-stage extension. |
| I2 = **#338** | Better **csv** support: make `instrument=/model=` discoverable, easy to tweak, more auto format-inspection. | UX + declarations ergonomics. Candidate to fold **#161** (Arbin SQL enhancements). |
| I3 = **#306** | Exporter dumping new-Arbin data → one file per test (`.h5`/v9). | An **exporter** in `cellpy/exporters/`, not a loader. |
| I4 = **#761** | Loader robustness: a custom loader **silently omits a declared-but-absent column** — should warn/fail (fail-loud posture). Carried from the 2.1 H2 Phase-2 deferral. | **Decided: pull into 2.2.** On `v.2.2`. Posture precedent now shipped: #938 (missing `mdb-export` / pyodbc raise `OptionalDependencyError`; `list_instruments()` rows carry `available` / `reason`). |
| I5 = **#827** *(added 2026-08-03)* | **PEC csv with several cells in one file** (repeated header blocks): `pec_csv` detects the blocks and returns one `Data` per cell. | The loader contract already returns `tuple[LoaderResult, ...]` (arch §5.6 decision) — this is the **first built-in multi-test source** exercising it end-to-end (`cellpy.get` → several cells / batch fan-out). Sample file attached to the issue → golden fixture. |

All independent and parallelizable. New loader-side dependencies must go into an extra
(the dependency-budget test guards the base install since #937).

## 5. Epic R — Remote paths & file discovery

| Issue | Content | Notes |
|---|---|---|
| R1 = **#687** | Regression: `scp://<ssh-config-Host-alias>/...` stopped resolving after the UPath/fsspec OtherPath switch. Either restore `Host`→`HostName` (+ `IdentityFile`/`User`) rewriting, **or** fail loudly + fix the misleading `remote_paths.md`. | **Bug.** UPath/Paramiko treats the URL host as DNS, skips OpenSSH `Host` rewriting. Acceptance incl. a regression test / `check_connection`. **Likely also a v2.1.x patch candidate** if a migrating user hits it before 2.2. |
| R2 = **#691** | Scope discovery to a **project subfolder** of `rawdatadir` (wire `project_dir` through `create_journal`/`auto_use_file_list`), with **fuzzy match hints** on a missing folder instead of a silent empty list. | Companion to the #688 full-tree perf fix. **Half shipped in v2.1.3 (#900):** `auto_use_file_list` is wired into `journal_from_db` / `find_files`, scoped to `project` by an **exact** join, and a missing project folder now **raises** with the joined path. **Remaining R2 = the fuzzy-match hints** (near-miss folder names in that error) + scoping the per-cell `search_for_files` path the same way. |

R1 is a standalone bug (fixable anytime, possibly ahead of 2.2 as a patch); the v2.1.3
OtherPath work (#901 fs reuse, #961 clearer auth errors) did **not** touch SSH-alias
resolution, so #687 is untouched. R2 is now a small follow-on to #900.

## 6. SPEED-30 headers → 2.3 (deferred, with reasoning)

Parked from 2.1 ([stage4 §1](stage4-github-issues.md)): value+unit+dtype, **versioned**
headers behind the Schema indirection — resolves per-column units for good; `units_label()`
re-points onto it.

**Decision: 2.3, as its own structural headline — not 2.2.** Rationale:

- The "born on SPEED-30 avoids a double migration" argument is weak: new S-epic columns
  migrate *with every other column* whenever SPEED-30 lands, regardless of which release
  first added them. Being born on the new format saves almost nothing.
- SPEED-30 is a **persisted format-contract change** (v9→v10-shaped) with its own read-compat,
  migration and oracle surface — the same weight class as the native-headers flip, which got
  its **own** stage. Bundling it with four science features + a cross-package live feature
  makes 2.2 heavy and slow.
- Deferring it also removes SPEED-30 as a **sequencing pivot** for 2.2: with it out, the
  S-epic just starts on the current schema.

2.2 therefore adds S columns to the current schema; SPEED-30 is the 2.3 headline where a
format change gets the focus it deserves.

## 7. Candidates considered (dispositions)

| Issue | Fits | Disposition |
|---|---|---|
| **#761** loader silent-omit | **I4** | **Pulled into 2.2** (retarget to `v.2.2`). |
| #303 cycle-statistics · #315 specify summarised values | S | **Optional-additive** S siblings — fold if a contributor picks them up; don't gate 2.2. |
| #340 plotly plotting too slow | perf | **Separate triage** — a perf investigation, not a 2.2 feature; don't bury it in-scope. |
| #161 Arbin SQL · #187 Biologic mpt · #238 voltaic | I | Loader coverage; fold near I1/I2 opportunistically. |
| **#206** original vs processed | own subsystem | **Design captured** ([cellpy2-data-curation-provenance.md](../../active/cellpy2-data-curation-provenance.md)); post-2.2, sequenced after Epic L. |
| #318 legacy-Arbin metadata | metadata | Metadata-plan adjacent; leave unmilestoned for now. |
| #302 Apache Spark · #243 sqlite dbreader | parked | Spark → narwhals-parking; sqlite dbreader is a larger refactor. |

## 8. Sequencing (the DAG)

```
 cellpycore-first ──►  S engine tools (#313 IR / #312 CCCV / #359 cycle-mode)
                             │  (core PR → PyPI → cellpy re-pin)
 cellpy-side:                ▼
              S1 + S2 ── land summary columns together ──┐
              S3 (cycle_mode) ── convention delta ────────┤─► migration/release notes (last)

              L6 (golden equality, first) ─► L1 (protocol) ─► L2 (load_since)
                                                                 ─► L3 (c.update) ─► L4/L5
              (entire L epic cellpy-only — no core in the critical path)

              I1 · I2 · I3 · I4 · I5 (all independent, parallel)
              R1 (bug, anytime)      R2 (fuzzy hints on top of #900)
              S4 (RPT filters, cellpy.filters) — independent of the core-first S1–S3
```

Constraints:
1. **S1/S2 share one schema addition** — coordinate, don't bump twice.
2. **S3 is a user-visible convention change** — goes in the #-delta register with an explicit
   verdict + comparator exception, like the 2.0 CE flip.
3. **L6 first** (the equality oracle anchors the feature); then **L1 → L2 → L3 → L4/L5**.
4. Only Epic **S** is core-first (F9). L, I, R are cellpy-only.
5. Startable immediately: **L6, S-oracle characterization, I1, I3, I4, R1**.

**yolo candidates** (additive, crisp criteria, fixture-guarded): I1, I3, I4, R2. L*, S1/S2/S3
and R1 involve engine contracts, a convention change, or root-cause debugging — not yolo.

## 9. Decisions (resolved 2026-07-28, maintainer)

1. **SPEED-30 → 2.3**, its own structural headline; 2.2 S-columns land on the current schema
   (§6). Removes the sequencing pivot from 2.2.
2. **Live-incremental protocol home = cellpy** (not cellpycore) → the whole L epic is
   cellpy-only, no core release in the critical path (§2).
3. **GITT/PITT (#73) → 2.3** — greenfield analysis routine; the OCV-extraction workflow is
   already covered by existing tools + the 05_GITT tutorial.
4. **#761 → pulled into 2.2** (Epic I4); retarget to `v.2.2`.
5. **Volumetric mode → parked**; **#303/#315 → optional-additive**; **#340 → separate perf
   triage** (§7).
6. **Reactive stream kept separate:** v2-migration bug reports ship as **`v2.1.x` patches off
   master**, not folded into 2.2; 2.2 scope stays lean to preserve that headroom.

**Done (2026-07-29):** tracking [#783](https://github.com/jepegit/cellpy/issues/783) +
`cellpy2-stage5` label created; L1–L6 / S1–S3 / I1–I4 / R1–R2 wired to the tracker; #761
pulled into `v.2.2`; `v.2.3` milestone created with #73 moved there; Stage 5 row added to the
architecture dashboard. SPEED-30 and GITT/PITT open the 2.3 planning.

---

## 10. Progress (updated 2026-09-08)

| Epic / item | Status | Evidence |
|---|---|---|
| **L** Live / incremental (#778→#164→#782) | ⬜ not started — **prerequisites moved** (§11.1) | all open; no open PRs. `live.py` still the 10-line stub, `processor.py` still 57-line scratch; `batch_core.py` deleted (L4 lstrip item gone) |
| **S** Step/summary science (#313/#312/#359 + **#888**) | ⬜ not started | all open; no core PRs. Core `main` is 1 fix ahead of the `0.2.4` pin (#143) — first S PR ships as `cellpycore 0.2.5` |
| **I** Instruments / IO (#270/#338/#306/#761 + **#827**) | ⬜ not started | all open; #938 shipped the fail-loud posture I4 needs |
| **R** Remote / discovery (#687/#691) | 🟡 R2 half done | #900 (v2.1.3) did the project-scoped exact join + raise-on-missing; fuzzy hints remain. #687 untouched |
| **#784** MetadataSource | ⬜ open (design ready) | on `v.2.2`; design in [`active/cellpy2-metadata-source-integration.md`](../../active/cellpy2-metadata-source-integration.md) |
| **#352** initial OCV batch plot | ⬜ open | opportunistic add to `v.2.2` |
| **Reactive `v2.1.x` stream** | 🟢 shipping, **heavy** | **v2.1.1.post4–post8 · v2.1.2 (+post1) · v2.1.3 (+post1–post3)**; ≈85 issues closed 2026-07-31→09-08 (`v.2.1.2` 25, `v.2.1.3` 27, `v2.1.3.post` 5, `v.2.1.4` 14 — **2.1.4 not yet tagged**) |
| Tracking #783 | open | checklist unchanged (all `[ ]`); body still shows the 2026-07-31 progress note |

**Startable now** (revised, §11.4): L6 (#778), S-oracle characterization, **S4 (#888)**,
I1 (#270), I3 (#306), I4 (#761), **I5 (#827)**, R1 (#687), R2 remainder (#691).

---

## 11. Revision 2026-09-08 — what five weeks of patches changed for Stage 5

Between the 2026-07-31 sync and today the project shipped **no Stage 5 work** but
**≈85 closed issues** across `v2.1.1.post4` … `v2.1.3.post3` (plus the 14 waiting in
`v.2.1.4`). Most of it was driven by dogfooding from `cellpy-simple-gui` and the new
`cellpy-mcp` server ([#840](https://github.com/jepegit/cellpy/issues/840)). Decision #6
(patches off master, 2.2 lean) **held** — but the patch stream moved the ground under
several Stage 5 items. This section records what changed; §1–§9 stay as the original plan
with inline corrections.

### 11.1 Epic L — prerequisites landed in the patch stream

The design ([live-incremental](../../active/cellpy2-live-incremental-design.md)) is intact:
core `update_data` / `update_core_data` unchanged, protocol still cellpy-hosted, L6 still
first. What changed is the **app-side substrate L3–L5 build on**:

| Shipped | Where | Effect on L |
|---|---|---|
| `SourcePreference.NEWEST` + `check_file_ids` raw-vs-cellpy freshness check (#825, v2.1.1.post6) | `batch/policy.py`, `cellreader.py` | **L3 change-detect reuses this** (FileID size + mtime) instead of a new mechanism |
| `refresh_after(...)` + `SUMMARY_META_DEPENDENCIES` (#846, v2.1.2) | `CellpyCell` | The app-side "derived refresh" hook L3 step 3 wanted already exists — extend it, don't add a second one |
| Orchestrated `batch.load` (#822), `force_recalc`, skip-redundant-save (#825), executors fixed for `processes` (#920), tqdm progress (#916) | `cellpy.batch` | **L5 rides on `Batch.update()` + runner**; `b.poll()` is a loop over that, not new orchestration. Progress callbacks already exist for the tick UI |
| Journal JSON `version` field (#1000, 2.1.4) | `batch/journal` | If L5 persists per-cell load markers in the journal, that is the **first journal version bump (v1 → v2)** — the mechanism is now in place |
| `batch_tools/batch_core.py` **deleted**; `batch/store.py` uses `removeprefix` | — | **L4 lstrip fix is obsolete** (struck in §2). L4 = `live.py` poll loop + delete `processor.py` only |
| Atomic `.cellpy` writes (#845) | `cellpy_file/` | A poll loop that saves every tick can no longer corrupt the file mid-write — one L3 risk gone |

**Consumers are real now.** `cellpy-simple-gui` (desktop) and `cellpy-mcp` (agent tools)
both want "is the test still running / refresh" — L6/L3 should ship a surface those two can
call without a batch (`c.update()` → bool changed) *before* L5.

### 11.2 Epic S — a convention change already slipped into the patch stream

- **#989 (v2.1.4): vendor capacity/energy that does not restart at 0 each cycle is now
  rebased on load**, with a `UserWarning` when values actually change. 1.x kept the tester
  column as-is. This is a **behavior delta vs 1.x** shipped as a patch → it must get a
  delta-register row (**Δ8**, architecture plan §7) and a comparator exception, exactly
  like S3 will. It also touches [#206](https://github.com/jepegit/cellpy/issues/206)
  (pristine raw vs processed): the rebase mutates `raw` on load, so the curation design's
  "pristine raw" needs to say whether the rebase is part of *ingestion* (harmonize) or the
  first *cleaning recipe step*. Recommendation: ingestion — it is a tester bookkeeping fix,
  not analysis — and record that in the curation doc.
- **Core is one fix ahead of the pin.** `cellpy-core` `main` carries #143 (legacy
  `cycle_mode` list unwrapping) unreleased over `v0.2.4`. The first S-epic core PR ships as
  **`cellpycore 0.2.5`** and cellpy re-pins per `v2-cellpycore-pin-gate.md`. No S work has
  started in core (0 open core issues).
- **EFC** (`equivalent_full_cycles`, core #138 in 0.2.4) is already surfaced in the
  summary — an example of an additive S-style column that landed without ceremony; S1/S2
  follow the same route (one schema addition, together).
- **S4 = #888 (RPT filters)** joins Epic S as the commercial-cell / lifetime-test companion
  of S3, but is **cellpy-only** on `cellpy.filters` — it does not wait for the core-first
  S1–S3 chain.
- Rule added to decision #6 (§11.5): a *bug fix that changes 1.x-visible numbers* is
  allowed in a patch, **but** it gets a Δ-row and a HISTORY line saying so — #989 is the
  precedent.

### 11.3 Epics I and R — posture shipped, one item half done

- **I4 (#761)**: the fail-loud posture it asks for now has a shipped precedent — #938
  (`OptionalDependencyError` for missing `mdb-export`/pyodbc; `list_instruments()` rows carry
  `available` / `reason`). I4 applies the same posture to declared-but-absent columns.
- **I5 = #827** (PEC multi-cell csv) joins Epic I. The contract already returns a tuple of
  results; I5 is the first built-in loader to *use* it, so it also tests `cellpy.get` /
  batch fan-out for a multi-test source. Fixture from the issue attachment.
- **I2 (#338)** overlaps with what `cellpy-mcp` / GUI now need from `list_instruments()`
  (discoverable `instrument=` / `model=`) — the discovery half of I2 may already be
  satisfied by #786/#938; re-scope I2 to the **format-inspection** half when picked up.
- **R2 (#691)** is half shipped by #900 (exact-join project scoping + raise-on-missing).
  Remaining: fuzzy near-miss hints in that error and per-cell `search_for_files` scoping.
- **R1 (#687)** untouched by the OtherPath perf/error work (#901/#961). Still a standalone
  bug and still a `v2.1.x` patch candidate.
- Dependency budget (#937: matplotlib/ipykernel optional; #969: xlrd added) — any new loader
  dependency for I1/I3/I5 goes in an extra.

### 11.4 Revised startable set and yolo fitness

Startable now: **L6 (#778)**, S-oracle characterization, **S4 (#888)**, **I1 (#270)**,
**I3 (#306)**, **I4 (#761)**, **I5 (#827)**, **R1 (#687)**, **R2 remainder (#691)**.

yolo candidates (additive, crisp, fixture-guarded): I1, I3, I4, **I5**, R2. **Not** yolo:
L*, S1–S3 (engine contracts / convention change), **S4** (design surface — filter API shape
needs a plan), R1 (root-cause debugging), #784.

### 11.5 Decisions (2026-09-08, proposed for maintainer confirmation)

7. **Ship `v2.1.4` before Stage 5 code starts.** 14 issues sit closed on `v.2.1.4` with an
   `[Unreleased]` HISTORY block; tagging it flushes the patch queue so 2.2 work starts from
   a released master. Also close the empty `v2.1.3.post` milestone.
8. **#827 → I5, #888 → S4** (both already on `v.2.2`; add the `cellpy2-stage5` label and
   the checklist lines on #783). **#889 (Fredrik ICA) → 2.3** with GITT/PITT (#73) — both are
   greenfield analysis routines.
9. **#989 → Δ8** in the delta register + comparator exception; curation design (#206) records
   the rebase as an ingestion step.
10. **L4 re-scoped** to `live.py` + `processor.py` only (lstrip item done via `batch/store.py`);
    **R2 re-scoped** to the fuzzy-hint remainder of #900.
11. **Decision #6 amendment:** patch-stream fixes that change 1.x-visible numbers get a
    Δ-row and an explicit HISTORY note (precedent #989). Everything else about #6 stands.
12. **Ecosystem:** `cellpy-mcp` is a sibling / dogfood repo alongside `cellpy-simple-gui`;
    Stage 5 L-epic surfaces are designed with both as consumers (§11.1).

Housekeeping still to do on GitHub (not done by this revision): update the #783 body
(progress note + S4/I5 checklist lines), label #827/#888 `cellpy2-stage5`, tag `v2.1.4`.
