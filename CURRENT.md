# Current work

**Start here** for what the ecosystem is actively planning or executing.
**Status sync:** 2026-07-31.

| | |
|---|---|
| **Stage** | **5 — cellpy 2.2** (additive “complete cellpy 2” features) |
| **Tracking** | [jepegit/cellpy#783](https://github.com/jepegit/cellpy/issues/783) · milestone `v.2.2` (**18 open / 0 closed**) · label `cellpy2-stage5` |
| **Latest cellpy** | **[v2.1.1.post3](https://github.com/jepegit/cellpy/releases/tag/v2.1.1.post3)** (2026-07-30) · `cellpycore==0.2.4` |
| **Coordinator** | [roadmap/cellpy2-architecture-plan.md](roadmap/cellpy2-architecture-plan.md) (Stage table) |
| **Issue set** | [roadmap/stages/stage5-github-issues.md](roadmap/stages/stage5-github-issues.md) |
| **Ecosystem primer** | [ecosystem/overview.md](ecosystem/overview.md) |

Shipped behind us: Stages 0–4 → stable **v2.0.0** / **v2.1.0**. Since the Stage 5 issue
set was cut (2026-07-29), work has been the planned **reactive `v2.1.x` patch stream**
— **no Stage 5 epic issues closed yet**.

---

## Patch stream (out-of-band from Stage 5)

Deliberately separate from 2.2 (stage5 decision #6). Recent ship:

| Release | When | Highlights |
|---|---|---|
| **v2.1.1** | 2026-07-29 | App-builder conveniences: `collect.from_cells` / `Batch.from_cells` (#787), group-avg plot fix (#785), `Collection.is_grouped` (#790), `save(xlsx/json)` (#789), `CurveOptions.mode/method` (#788), quiet `list_instruments` (#786) |
| **v2.1.1.post1** | 2026-07-29 | HISTORY backfill (#802) |
| **v2.1.1.post2** | 2026-07-30 | Collected summary per-panel y-limits (#804) |
| **v2.1.1.post3** | 2026-07-30 | Quieter loader-discovery for apps (#786 follow-through) |

**Still open on `v.2.1.2`:** [#345](https://github.com/jepegit/cellpy/issues/345) batch custom-JSON · [#799](https://github.com/jepegit/cellpy/issues/799) `read_meta` · [#800](https://github.com/jepegit/cellpy/issues/800) per-instrument meta schema. Closed on that milestone: #801 (figure theme hook), #804.

---

## Active focus (Stage 5) — not started

All original L / S / I / R tracker checkboxes still open. Startable now per stage5 §8:
**#778 (L6)**, S-oracle characterization, **#270**, **#306**, **#761**, **#687**.

| Focus | Plan / home | Issues | Notes |
|---|---|---|---|
| **L** Live / incremental | [active/cellpy2-live-incremental-design.md](active/cellpy2-live-incremental-design.md) | #778→#779→#780→**#164**→#781→#782 | cellpy-only; L6 first |
| **S** Step/summary science | core-first | #313 / #312 / #359 | IR, CCCV, discharge-first |
| **I** Instruments / IO | stage5 §I | #270 / #338 / #306 / #761 | parallelizable |
| **R** Remote / discovery | stage5 §R | #687 / #691 | R1 also a patch candidate |
| **M** External metadata *(milestone add)* | [active/cellpy2-metadata-source-integration.md](active/cellpy2-metadata-source-integration.md) | [#784](https://github.com/jepegit/cellpy/issues/784) | now on `v.2.2` (was “2.3+” in early notes) |
| *(opportunistic)* | — | [#352](https://github.com/jepegit/cellpy/issues/352) | initial-OCV batch plot; on `v.2.2`, not in original epic cut |

## Design captured, not yet scheduled

| Topic | Plan | When |
|---|---|---|
| Data curation / provenance (#206) | [active/cellpy2-data-curation-provenance.md](active/cellpy2-data-curation-provenance.md) | post-2.2 (after Epic L) |

## Deferred → 2.3

SPEED-30 versioned headers · GITT/PITT [#73](https://github.com/jepegit/cellpy/issues/73) · [#770](https://github.com/jepegit/cellpy/issues/770) migration-test cleanup — `v.2.3` milestone (**2 open**).

---

## Where to look next

| Need | Path |
|---|---|
| Full stage dashboard | [roadmap/cellpy2-architecture-plan.md](roadmap/cellpy2-architecture-plan.md) |
| Gap analysis / ownership | [roadmap/cellpy2-plans-gap-analysis.md](roadmap/cellpy2-plans-gap-analysis.md) |
| Plans still guiding open work | [active/](active/) |
| Executed topic plans | [archive/](archive/) |
| Evidence / scans | [research/](research/) |
| Package layout & conventions | [ecosystem/](ecosystem/) |
| Old basename → new path | [PATHS.md](PATHS.md) |
