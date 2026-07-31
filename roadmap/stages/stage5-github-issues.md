# Stage 5 — cellpy 2.2 scope plan (issue set)

**Date:** 2026-07-28 (issue set cut 2026-07-29) · **Status sync:** 2026-07-31 ·
**Status:** 🟡 **issue set cut; execution not started** (0 Stage 5 epic issues closed).
Tracking [#783](https://github.com/jepegit/cellpy/issues/783); scope decisions in §9.
2.1 shipped (v2.1.0 + post1); reactive patch stream through **v2.1.1.post3** (see §10).

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

**Milestone `v.2.2` (2026-07-31: 18 open / 0 closed):** original set
#778/#779/#780/#164/#781/#782 · #313/#312/#359 · #270/#338/#306/#761 · #687/#691 ·
tracking #783 — plus milestone adds **#784** (external metadata sources) and
**#352** (initial-OCV batch plot). Net of the decisions below: **#73 → 2.3**,
**#761 → pulled into 2.2**.

---

## 1. Scope (agreed)

| In 2.2 | Deferred → 2.3 | Parked (spot kept) |
|---|---|---|
| **L** Live / incremental refresh (`c.update()`, #164) — **cellpy-only**, design done | **F** SPEED-30 value+unit+dtype versioned headers — the **2.3 structural headline** (§6) | UUID / BattINFO-EMMO ontology mapping (metadata OQ6, F10) |
| **S** Step/summary science: IR-at-endpoints #313, CCCV split #312, discharge-first #359 (on the **current** schema) | **#73** GITT/PITT — greenfield analysis routine, early-2.3 | narwhals evaluation (polars plan decision 2) |
| **I** Instruments / IO: Biologic mpr v3 #270, better csv #338, Arbin-db exporter #306, loader robustness **#761** | | per-test `raw_units`; fsspec beyond ssh; **volumetric mode** |
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
| L4 | cellpy utils | `live.py` `poll(...)` loop (replaces the stub); **delete `processor.py`** (fold thread-pool into `batch.runner`); fix `batch_core.py:180` `lstrip`→`removeprefix` if still present |
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
| I4 = **#761** | Loader robustness: a custom loader **silently omits a declared-but-absent column** — should warn/fail (fail-loud posture). Carried from the 2.1 H2 Phase-2 deferral. | **Decided: pull into 2.2.** Currently unmilestoned → retarget to `v.2.2`. |

All independent and parallelizable.

## 5. Epic R — Remote paths & file discovery

| Issue | Content | Notes |
|---|---|---|
| R1 = **#687** | Regression: `scp://<ssh-config-Host-alias>/...` stopped resolving after the UPath/fsspec OtherPath switch. Either restore `Host`→`HostName` (+ `IdentityFile`/`User`) rewriting, **or** fail loudly + fix the misleading `remote_paths.md`. | **Bug.** UPath/Paramiko treats the URL host as DNS, skips OpenSSH `Host` rewriting. Acceptance incl. a regression test / `check_connection`. **Likely also a v2.1.x patch candidate** if a migrating user hits it before 2.2. |
| R2 = **#691** | Scope discovery to a **project subfolder** of `rawdatadir` (wire `project_dir` through `create_journal`/`auto_use_file_list`), with **fuzzy match hints** on a missing folder instead of a silent empty list. | Companion to the #688 full-tree perf fix. Depends on batch knowing `project` (it does). |

R1 is a standalone bug (fixable anytime, possibly ahead of 2.2 as a patch). R2 rides on
batch/journal wiring.

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

              I1 · I2 · I3 · I4     (all independent, parallel)
              R1 (bug, anytime)      R2 (needs batch project wiring)
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

## 10. Progress (updated 2026-07-31)

| Epic / item | Status | Evidence |
|---|---|---|
| **L** Live / incremental (#778→#164→#782) | ⬜ not started | all open; no open PRs |
| **S** Step/summary science (#313/#312/#359) | ⬜ not started | all open; no core PRs for these |
| **I** Instruments / IO (#270/#338/#306/#761) | ⬜ not started | all open |
| **R** Remote / discovery (#687/#691) | ⬜ not started | all open |
| **#784** MetadataSource | ⬜ open (design ready) | on `v.2.2`; design in [`active/cellpy2-metadata-source-integration.md`](../../active/cellpy2-metadata-source-integration.md) |
| **#352** initial OCV batch plot | ⬜ open | opportunistic add to `v.2.2` |
| **Reactive `v2.1.x` stream** | 🟢 shipping | **v2.1.1** … **v2.1.1.post3** (collect/plot app work); `v.2.1.2` still open (#345/#799/#800) |
| Tracking #783 | open | checklist unchanged (all `[ ]`) |

**Startable now** (unchanged from §8): L6 (#778), S-oracle characterization, I1 (#270),
I3 (#306), I4 (#761), R1 (#687).
