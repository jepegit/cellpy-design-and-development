# Current work

**Start here** for what the ecosystem is actively planning or executing.
**Status sync:** 2026-09-08.

| | |
|---|---|
| **Stage** | **5 — cellpy 2.2** (additive “complete cellpy 2” features) — **plan revised 2026-09-08** ([stage5 §11](roadmap/stages/stage5-github-issues.md#11-revision-2026-09-08--what-five-weeks-of-patches-changed-for-stage-5)) |
| **Tracking** | [jepegit/cellpy#783](https://github.com/jepegit/cellpy/issues/783) · milestone `v.2.2` (**20 open / 0 closed**) · label `cellpy2-stage5` |
| **Latest cellpy** | **[v2.1.3.post3](https://github.com/jepegit/cellpy/releases/tag/v2.1.3.post3)** (2026-09-05) · `cellpycore==0.2.4` · **v2.1.4 pending** (`v.2.1.4`: 14 closed / 0 open, HISTORY `[Unreleased]`) |
| **Coordinator** | [roadmap/cellpy2-architecture-plan.md](roadmap/cellpy2-architecture-plan.md) (Stage table) |
| **Issue set** | [roadmap/stages/stage5-github-issues.md](roadmap/stages/stage5-github-issues.md) |
| **Ecosystem primer** | [ecosystem/overview.md](ecosystem/overview.md) |

Shipped behind us: Stages 0–4 → stable **v2.0.0** / **v2.1.0**. Since the Stage 5 issue
set was cut (2026-07-29), all work has been the planned **reactive `v2.1.x` patch stream**
— ≈85 issues closed, **no Stage 5 epic issues closed yet**. The 2026-09-08 revision
records what that stream changed for Stage 5 (prerequisites landed, one item half done,
two milestone adds, one convention delta to register).

---

## Patch stream (out-of-band from Stage 5)

Deliberately separate from 2.2 (stage5 decision #6, amended by #11). Shipped since 2.1.0:

| Release | When | Highlights |
|---|---|---|
| **v2.1.1** (+post1–post3) | 2026-07-29–30 | App-builder conveniences: `from_cells` (#787), group-avg fix (#785), `is_grouped` (#790), `save(xlsx/json)` (#789), per-panel y-limits (#804), quiet loader discovery (#786) |
| **v2.1.1.post4–post6** | 2026-07-31 → 08-02 | `read_meta` (#799), `instrument_meta_schema` (#800), custom-JSON journals (#345), figure theme hook (#801); orchestrated `batch.load` + `AUTO`/`NEWEST`/`force_recalc` (#822/#825) |
| **v2.1.2** (+post1) | 2026-08-09–10 | Plot families → `SummaryOptions` (#868), `raw_plot` `max_points` (#867), `collect_dva` (#863), static export (#818); config on the loaded file (#851/#853), thread-local `override()` (#850), no plaintext SQL creds (#849); **atomic `.cellpy` writes** (#845), **`refresh_after`** (#846), CLI cold start (#837); tutorials de-duplicated (#869/#866) |
| **v2.1.3** (+post1–post3) | 2026-08-16 → 09-05 | Batch-load speed epic #896 (`auto_use_file_list` project scoping #900, `find -L` #897/#899, OtherPath fs reuse #901, `arbin_sql_h5` one read #902, zstd v9 #912); `executor="processes"` persist (#920), tqdm (#916), `export_project` (#878); CLI quietness (#891); collector polish (#923–#928, #947); filefinder-miss → `FAILED` (#962); ICA recipe unification (#987), conda pins (#969) |
| **v2.1.4** *(pending)* | — | Capacity rebase-on-load **#989** (Δ8 candidate), journal `version` (#1000), `cellpy setup` TOML-only (#960), fail-loud missing tools (#938), matplotlib/ipykernel optional (#937), `b.plot(ir=True)` (#949), group labels from db (#982), MCP integration #840 → new `cellpy-mcp` repo |

**Open on the patch line:** nothing — `v.2.1.4` has 0 open issues and only needs a tag.

---

## Active focus (Stage 5) — not started

All L / S / I / R tracker checkboxes still open. Startable now per stage5 §11.4:
**#778 (L6)**, S-oracle characterization, **#888 (S4)**, **#270**, **#306**, **#761**,
**#827 (I5)**, **#687**, **#691 remainder**.

| Focus | Plan / home | Issues | Notes |
|---|---|---|---|
| **L** Live / incremental | [active/cellpy2-live-incremental-design.md](active/cellpy2-live-incremental-design.md) | #778→#779→#780→**#164**→#781→#782 | cellpy-only; L6 first. Substrate landed in patches: `NEWEST`/`check_file_ids`, `refresh_after`, orchestrated `Batch.update()`, journal `version`. L4 lstrip item obsolete |
| **S** Step/summary science | core-first (S1–S3); cellpy `filters` (S4) | #313 / #312 / #359 · **#888** | IR, CCCV, discharge-first; **S4 RPT filters** (milestone add). Core `main` = 0.2.4 + #143 → first S PR ships `cellpycore 0.2.5` |
| **I** Instruments / IO | stage5 §I | #270 / #338 / #306 / #761 · **#827** | parallelizable; **I5 PEC multi-cell** (milestone add, first tuple-returning built-in loader); #938 set the fail-loud posture for I4 |
| **R** Remote / discovery | stage5 §R | #687 / #691 | R2 half done by #900 (fuzzy hints remain); R1 untouched, still a patch candidate |
| **M** External metadata | [active/cellpy2-metadata-source-integration.md](active/cellpy2-metadata-source-integration.md) | [#784](https://github.com/jepegit/cellpy/issues/784) | on `v.2.2` |
| *(opportunistic)* | — | [#352](https://github.com/jepegit/cellpy/issues/352) | initial-OCV batch plot; on `v.2.2`, not in original epic cut |

**Before Stage 5 code starts (stage5 decision #7):** tag **v2.1.4**; update #783 body with
S4/I5 lines; label #827/#888 `cellpy2-stage5`.

## Design captured, not yet scheduled

| Topic | Plan | When |
|---|---|---|
| Data curation / provenance (#206) | [active/cellpy2-data-curation-provenance.md](active/cellpy2-data-curation-provenance.md) | post-2.2 (after Epic L). **Note:** #989 rebase-on-load mutates `raw` at ingestion — doc must classify it (stage5 §11.2) |

## Deferred → 2.3

SPEED-30 versioned headers · GITT/PITT [#73](https://github.com/jepegit/cellpy/issues/73) ·
Fredrik ICA [#889](https://github.com/jepegit/cellpy/issues/889) · [#770](https://github.com/jepegit/cellpy/issues/770)
migration-test cleanup — `v.2.3` milestone (**3 open**).

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
| Extensions / wishlist (unscheduled) | [ecosystem/wishlist.md](ecosystem/wishlist.md) |
| Old basename → new path | [PATHS.md](PATHS.md) |
