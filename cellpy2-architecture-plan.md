# Plan: the total cellpy 2 architecture (MVP → complete)

**Date:** 2026-07-28 (living document — iterate here; see iteration log at the end)
**Status:** v5 — Stages 0–3 complete; stable **v2.0.0** shipped (2026-07-26). Stage 4
(**2.1**) essentially complete — Epics A–H merged; only the tracking umbrella
[#696](https://github.com/jepegit/cellpy/issues/696) and [#345](https://github.com/jepegit/cellpy/issues/345)
(verify/close) remain on the `v.2.1` milestone.
Reconciled against the implemented code and the [#438 decision register](stage0-github-issues.md).
This document *coordinates* the topic plans; it decides nothing they own.
**How to read this document:** §Status for where we are today → §1–3 for the target
(MVP and complete) → §4–5 for the design commitments (patterns + loader contract) →
§6 for the timeline and the **final-legacy-release gate** → §7 for the behavior-delta
register → §8 if you are a developer wanting to touch utils or loaders now.

---

## Status at a glance (updated 2026-07-28)

| Stage | Status | Evidence |
|---|---|---|
| Stage 0 — foundations | ✅ complete 2026-07-11 | all 12 issues closed (tracking [jepegit/cellpy#439](https://github.com/jepegit/cellpy/issues/439)); six #438 decisions recorded in the plan docs; details in [stage0-github-issues.md](stage0-github-issues.md) |
| Stage 1 — v1.x-safe prep | ✅ complete 2026-07-15 | all sub-issues closed (cellpy #446–#458, #479; core#115–#118 shipped as **cellpycore 0.2.0** + re-pin); tracking [#459](https://github.com/jepegit/cellpy/issues/459) closed after §7 delta sign-off; details in [stage1-github-issues.md](stage1-github-issues.md) |
| Final legacy (v1.x) release | ✅ shipped | v1.1 milestone closed; latest `v1.1.0.post3`; `v1.x` branch carries bugfix-only maintenance (12-month window starts on stable 2.0.0, decision #438-6) |
| Stage 2 — the flip | ✅ complete 2026-07-18 | flip Stages 0–6 merged (#511, #548–#557); `native_schema` default on; value-parity oracle real; released as **v2.0.0a5** |
| Stage 3 — 2.0 assembly | ✅ complete 2026-07-26 | assembly issues closed (tracking [#575](https://github.com/jepegit/cellpy/issues/575) closed); scope decision (2 of 4 redesigns in 2.0) in [stage3-github-issues.md](stage3-github-issues.md). Shipped: loader contract + `harmonize()` + tier-1/2 ports + tier-3 decisions (#210, #558–#561), metadata/config/units (#562–#565), ICA + plotting redesigns (#566–#567, #591), CLI/docs/packaging (#568–#573), follow-ons (#580, #594, #651, #654); **cellpycore 0.2.4** re-pin. **Released:** stable [**v2.0.0**](https://github.com/jepegit/cellpy/releases/tag/v2.0.0) (2026-07-26); release checklist [#574](https://github.com/jepegit/cellpy/issues/574) closed |
| Stage 4 — 2.1 | ✅ essentially complete (2026-07-28) | `v.2.1` milestone **37 closed / 2 open** (tracking [#696](https://github.com/jepegit/cellpy/issues/696); #697–#721 delivered). Merged: batch v3 (A #697–#704) + collectors redesign (B #705–#708); utils C1 #709 (**C2 #164 live/incremental → 2.2**); F6 menu (D #710–#712); **all shim removals (E1–E5 #713–#717, breaking)**; docs G1–G3 #718–#720; cellpy-core doc-sync H1 #721; test-fixture gaps H2 [#655](https://github.com/jepegit/cellpy/issues/655) **Phase 1 closed** (dead-ref cleanup, loader goldens, bad-file set; Phase 2 dataset curation deferred — surfaced [#761](https://github.com/jepegit/cellpy/issues/761)). Single-release (Epic E breaking → no 2.0.x split). SPEED-30 + GITT/PITT → 2.2. **Remaining:** [#345](https://github.com/jepegit/cellpy/issues/345) (verify/close) + [#696](https://github.com/jepegit/cellpy/issues/696) umbrella (closes at 2.1 tag) |

**Drift check 2026-07-28 (code vs plan): no structural drift.** Stage 3 assembly
matches the narrowed 2.0 scope (ICA + plotting in; batch/collectors deferred to 2.1).
`readers/cellpy_file/`, `cellpy/config/`, loader `harmonize()` + declarations,
`cellpy.plotting` as the single plotting home, Typer/`cli_api`, Zensical docs, and
the file-format compatibility matrix all match their owning plans. Stage 4 landed
as planned: `cellpy.batch` + `cellpy.collect` top-level packages (utils.* now
re-export shims), the 2.0 deprecation shims removed on schedule (Epic E), and the
prms global shim gone while the `c.mass` property facade was intentionally kept.
Earlier planning reconciliations (loader port in Stage 3; tiered GHA benchmark gate
#476) remain valid. No open architectural gap; remaining work is the 2.1 release
tag itself (umbrella #696) and the #345 verify/close.

---

## 0. The plan set (what owns what)

| Plan document | Owns |
|---|---|
| [cellpy2-configuration-and-parameters-plan.md](cellpy2-configuration-and-parameters-plan.md) | Config stack (pydantic-settings/TOML), prms shim, secrets, OtherPath decision (§5b) |
| [cellpy2-metadata-handling-plan.md](cellpy2-metadata-handling-plan.md) | `CellMeta`/`TestMetaCollection` wiring, meta mapping, `MetaResolver`, meta persistence |
| [unit-handling-cellpy2-plan.md](unit-handling-cellpy2-plan.md) | One unit spec/registry, converter delegation, `units_label`, unit policy |
| [cellpy-file-loading-refactor-plan.md](cellpy-file-loading-refactor-plan.md) | Extracting cellpy-file I/O out of `CellpyCell` (behavior-preserving, v8) |
| [cellpy2-native-headers-migration-plan.md](cellpy2-native-headers-migration-plan.md) | Native `Cols` end-to-end, `translate.py`, v9 format, the flip oracle, accessor shim |
| [cellpy2-polars-port-execution-plan.md](cellpy2-polars-port-execution-plan.md) | pandas→polars on the cellpy side; keys-in-columns; the flip's frame-type half |
| [cellpy2-loader-port-and-extraction-plan.md](cellpy2-loader-port-and-extraction-plan.md) | Loader port to harmonized raw (`harmonize()` + declarations), `cellpycore.curves` |
| [cellpy2-utils-migration-plan.md](cellpy2-utils-migration-plan.md) | Utils/batch/plotting/filters migration in waves; the "utils contract"; live/incremental |
| [cellpy2-batch-redesign-plan.md](cellpy2-batch-redesign-plan.md) | Batch v3 architecture (journal/policy/runner/result/store) and the migration off farm/barn `batch_tools` |
| [cellpy2-plotting-redesign-plan.md](cellpy2-plotting-redesign-plan.md) | One plotting home (prepare→spec→render, figure registry); collectors re-based on it; batch_plotters/easyplot retired |
| [cellpy2-collectors-redesign-plan.md](cellpy2-collectors-redesign-plan.md) | Collection layer: `Collection` with provenance, options dataclasses replacing "elevated arguments", `concat_summaries` consolidation |
| [cellpy2-ica-redesign-plan.md](cellpy2-ica-redesign-plan.md) | ICA + DVA: `dqdv()` and new `dvdq()` on one pure core, `IcaOptions`, specced long-format output frames |
| [cellpy2-conventions-plan.md](cellpy2-conventions-plan.md) | Exceptions, logging, deprecation cadence (`warn_once`, `DEPRECATIONS.md`) |
| [cellpy2-cli-redesign-plan.md](cellpy2-cli-redesign-plan.md) | Click→Typer; library-first `cli_api` so commands are callable from scripts |
| [cellpy2-documentation-plan.md](cellpy2-documentation-plan.md) | Sphinx→Zensical; researcher-first IA; API refs via docstring cleanup |
| [cellpy2-release-and-branching-plan.md](cellpy2-release-and-branching-plan.md) | 2.0 support matrix, trunk-based flip, cross-repo merge order, benchmarks |
| [cellpy2-plans-gap-analysis.md](cellpy2-plans-gap-analysis.md) | The cross-read that produced the set; ownership table in its §3 |

**Builds on (cellpy-core design docs):**
`cellpy-core/.issueflows/04-designs-and-guides/cellpy-core-migration.md`,
`cellpy-core-integration-roadmap.md` (STEPs 01–12), `cellpy-core-integration-into-cellpy.md`.

---

## 1. The total solution in one sentence

Cellpy 2 is a **layered, two-package system**: `cellpycore` is a small, pure,
polars-based compute engine that owns *shapes and tools* (schemas, step/summary/curve
engines, metadata models, unit converters, the legacy mapping), and `cellpy` becomes a
thin application layer that owns *content and policy* (configuration, instrument
loaders, metadata population, persistence, plotting, batch). Everything crossing the
seam is plain values — never config objects, pint quantities, or file handles.
Translation between the v1 dialect and the native schema happens **once, at I/O
boundaries**, not per call. The v1 API survives through deprecation shims on the
shared cadence (introduced 2.0, removed 2.1).

**Where we already are (2026-07-24):** `cellpycore` owns the compute engine (steps,
summary, headers, timestamps, metadata scaffolding, units, curves) behind a lean
core bridge, guarded by contract tests and golden fixtures. On the cellpy side,
Stages 0–2 and Stage 3 *assembly* are done: `cellpy_file/` (v8/v9), pydantic-settings
config + secrets, public `c.schema`, declaration-driven loaders + `MetaResolver`,
ICA (`dqdv`/`dvdq`) and `cellpy.plotting` as the single plotting home, Typer/`cli_api`,
Zensical docs, and dependency budget. Runtime defaults are native headers + polars +
v9. **v2.0.0rc1** is on PyPI (`pip install --pre cellpy==2.0.0rc1`); stable `v2.0.0`
is the remaining Stage-3 gate. Every package still has an owning plan; the six #438
decisions stay recorded in those docs. Stage 4 (batch/collectors redesigns, utils
waves 3–4, shim removals) is not started.

---

## 2. MVP cellpy 2 (the 2.0 release)

**Redefined 2026-07-09.** The release/branching plan fixed the 2.0 support matrix:
cellpy 2.0 ships **polars frames, native headers, and v9 files** — the flip is *in*
the MVP, not after it. (The previous draft's "keep v8 + legacy dialect" MVP is now
the *preparation stage*: everything behavior-preserving lands on master v1.x-safe
before the flip; see the timeline, §6.) What keeps 2.0 minimal is the shim strategy —
every documented v1 pattern still works, warning once — and the wave structure:
only utils waves 1–2 (batch, helpers, filters, plotting) are 2.0-blocking.

```text
┌──────────────────────────────────────────────────────────────────┐
│ USER API (v1 patterns work via shims, warn once → DEPRECATIONS.md)│
│   cellpy.get() · CellpyCell facade · batch/helpers/filters/      │
│   plotutils/collectors (utils waves 1–2) · cellpy convert CLI    │
│   c.mass etc. → property facade · prms.* → config shim           │
│   c.schema = public header API (headers_* become the shim)       │
├──────────────────────────────────────────────────────────────────┤
│ APP LAYER (cellpy 2)                                             │
│   config/       pydantic-settings + TOML + platformdirs;         │
│                 override() ctx-mgr; provenance; setup migrate    │
│   MetaResolver  kwargs > journal/db > raw file > config defaults │
│   cellpy_file/  v9 parquet native (write) · v8/legacy read →     │
│                 to_native() once · save(format="v8") compat      │
│   instruments/  vendor parse() + declaration-driven harmonize(); │
│                 tier-1/2 loaders emit harmonized raw + drafts    │
│   conventions   exception tree rooted in core · opt-in logging · │
│                 warn_once helper · secrets env-only (SecretStr)  │
├────────────────── seam: plain values only ───────────────────────┤
│ ENGINE (cellpycore, PyPI pin) — native path                      │
│   CellpyCellCore · Data (polars, native Cols, test_id keys)      │
│   make_step_table · make_summary · curves · merge · timestamps   │
│   metadata models/io · units spec+converters · legacy/mapping    │
│   (OldCellpyCellCore serves only the v1.x maintenance branch)    │
└──────────────────────────────────────────────────────────────────┘
    oracle: value parity through the mapping on the golden cells
    guard:  benchmarks vs the v1.x baseline — no metric slower
```

2.0 definition of done (from the release plan, restated): reads v8+v9, writes v9
(+ `save(format="v8")`), v<8 handled by `cellpy convert` on 1.x; pytables demoted to
a `legacy-files` extra; dependency deltas per release plan §5; benchmark acceptance;
every shim registered in `DEPRECATIONS.md`.

---

## 3. Complete cellpy 2 (2.x)

What the complete system adds over the 2.0 MVP:

```text
┌──────────────────────────────────────────────────────────────────┐
│ USER API                                                         │
│   canonical access only (shims removed in 2.1):                  │
│   c.data.meta.mass · c.data.tests[tid].… · c.schema.raw.potential│
│   units_label() / with_cellpy_unit() presentation helpers        │
│   live/incremental: poll a running test, refresh steps/summary   │
│   batch reports with test-level summaries + native-only columns  │
│   (energies, powers, durations) · exclude_step_types on summaries│
├──────────────────────────────────────────────────────────────────┤
│ APP LAYER                                                        │
│   utils waves 3–4 done: ica + ocv_rlx on cellpycore.curves;      │
│   live.py/processor.py rebuilt on core incremental (or dropped   │
│   deliberately) · easyplot removed at 2.0 · docs/CLI plans executed     │
│   corrected IR semantics as default (documented oracle exception)│
├──────────────────────────────────────────────────────────────────┤
│ IO / PLUGIN LAYER                                                │
│   entry-point loader registry incl. third-party loaders          │
│   tier-3 decisions executed (biologics/batmo ported; nda parked) │
│   BDF adapter placement decided · JSON-LD/DB transport           │
│   implementable against core's io stubs (TestMeta.uuid linkable) │
├────────────────── seam: plain values only ───────────────────────┤
│ ENGINE (cellpycore)                                              │
│   volumetric mode implemented · test-level summaries             │
│   SPEED-30 headers (value+unit+dtype, versioned) slotted behind  │
│   the Schema indirection — resolves per-column units for good    │
│   OldCellpyCellCore + shims deleted when v1.x support ends       │
└──────────────────────────────────────────────────────────────────┘
```

Deliberately deferred with a parking spot (not forgotten): cell-identity UUIDs +
BattINFO/EMMO ontology mapping (metadata plan open question 6, F10), fsspec/remote
schemes beyond ssh (config plan §5b revisit trigger), narwhals (polars plan
decision 2 revisit trigger), per-test `raw_units` for mixed-instrument merges
(metadata plan open question 6).

---

## 4. Design patterns we commit to

| Pattern | Where it applies in cellpy 2 |
|---|---|
| **Hexagonal / ports-and-adapters** | The governing decision — "core owns shapes and tools; consumer owns content and policy" *is* this pattern. Core is the pure domain (no file/env/network I/O, enforced by CI lint); loaders, persistence, and db are adapters. |
| **Functional core, imperative shell** | Engines are pure functions over frames; all side effects (file reads, config resolution) happen in the shell and results cross the seam **by value** (the factors-by-value unit seam). Dependency injection done the pythonic way — pass values, not containers. |
| **Facade** | `CellpyCell` shrinks from a 7,400-line god object to a thin facade delegating to core engines, `cellpy_file/`, and `cellpycore.curves`. |
| **Adapter / anti-corruption layer, at I/O only** | `translate.py` (`to_native`/`to_legacy`) over `cellpycore.legacy.mapping`: translation happens **once at file boundaries**, reversing today's per-call rename sandwich (native-headers plan D1). The legacy HDF5 import shim and `save(format="v8")` are the same pattern. Deliberately disposable. |
| **Repository** | Stateless `cellpy_file/` package (format registry, `read/legacy_read/write/translate`) replacing hidden-mutable-state extraction methods. Core ships interface stubs; cellpy implements them. |
| **Strategy + plugin registry** | Instrument loaders behind one formal contract (§5), discovered via `importlib.metadata` entry points; built-ins use the same registry. |
| **Declarative configuration over code** | Loader normalization is driven by *declarations* (column_map, units, tz, reset granularity, aux_map) validated at registration time — a typo'd declaration fails at import, not mid-load (loader plan §2.2). |
| **Pipeline (two-stage ingestion)** | `vendor file → parse() → vendor frame + declarations → harmonize() → native raw`. The hard-won vendor parsing stays per-loader; everything after it is one shared, framework-owned stage (loader plan §2.1). |
| **Chain of responsibility (layered resolution)** | `MetaResolver` precedence (kwargs > journal/db > raw file > config defaults) and pydantic-settings source layering — both with inspectable provenance. |
| **Null object / graceful degradation** | Engines run with empty `CellMeta`/`TestMeta`; populated metadata is never required on the hot path (guarded by tests). |
| **Lazy singleton, scoped mutation** | Config as a lazily-created module attribute (PEP 562 `__getattr__`), never import-time I/O — the same rule now extended to logging (conventions plan §2: `NullHandler`, explicit `cellpy.log.setup()`). `override()` + pytest fixture end global-state test leaks. One memoized pint registry per process. |
| **Typed models at the boundary** | Plain dataclasses in core; pydantic v2 (`validate_assignment`, `SecretStr`) only in the app layer — validation fails loudly at load time. Fail-loudly posture codified in the conventions plan §4 (exactly two warn-only escape hatches: `local_instrument`, `accept_old`). |
| **Deprecation facade, one cadence** | All shims follow the conventions plan §3: introduced 2.0, removed 2.1, `warn_once` with the replacement named, self-registered in a generated `DEPRECATIONS.md`. Private `_` API breaks without deprecation. |
| **Golden master + contract tests** | Characterization before touching; field-by-field parity contracts; golden fixtures regenerated by script, never by hand (shared convention F8). **The oracle changes character at the flip**: byte-parity → value parity through the mapping, with documented exceptions for frozen legacy bugs (the IR semantics case, F4). |
| **Structural typing (`typing.Protocol`)** | The loader contract (§5) is a Protocol, not an ABC — third-party loaders need no cellpy base class; conformance is structural, checked in the registry and by the test-kit. |
| **Immutable-by-convention frames** | Engines return new frames rather than mutating in place; *keys live in columns, never in an index* is law (polars plan decision 3), lint-enforced (polars plan Phase D). |
| **Single exception hierarchy, rooted in core** | `cellpycore.exceptions.CellpyError` at the root; the app derives `ConfigurationError`, `UnitsError`, `LoaderError`, `CorruptCellpyFile`, … No bare `Exception`, no blanket `except Exception` at boundaries (conventions plan §1). |

---

## 5. The formal instrument-loader contract

Updated 2026-07-09 to align with the
[loader-port plan](cellpy2-loader-port-and-extraction-plan.md)'s two-stage design.
The Protocol below is the **outer** contract (what the registry and framework see);
the loader plan's `parse()` + `harmonize()` split is the **sanctioned implementation**
of it for built-in loaders.

### 5.1 Design decisions

1. **`typing.Protocol`, not an ABC.** Third-party loaders conform structurally; no
   inheritance, no import of a cellpy base class required. `@runtime_checkable` so the
   registry can `isinstance`-check at registration (the test-kit does the real
   conformance check).
2. **Two-stage implementation (loader plan §2.1).** Loaders keep their vendor-specific
   `parse()` (the hard-won part) and *declare* the rest — column_map, units, tz rule,
   reset-granularity per cumulative column, aux_map — in their configuration. The
   framework-owned `harmonize()` performs rename → cast (`RawCols.dtype_map()`) →
   timestamp conversion → reset-granularity normalization (core #42) → identity/
   provenance stamping → `validate_raw_frame` (strict; warn-only for
   `local_instrument`). Built-in loaders therefore implement `load()` as
   `harmonize(parse(source), declarations)`; third-party loaders may do the same or
   emit native frames directly — the contract only judges the result.
3. **The result is a frozen dataclass (`LoaderResult`),** never attribute-poking on
   `Data`. Loaders fill **only what the file actually knows**; provenance
   (`source_kind/type/uri`, `source_uuid`, `raw_file_names`, `loaded_datetime`,
   `uuid`, `test_id`, `mask`) is stamped by the framework.
4. **The raw frame is polars in the harmonized-raw schema** (`config.RawCols`:
   `potential` not voltage, `test_time` in seconds, `epoch_time_utc` int64 ns UTC,
   `datapoint_num` as a *column*). During the transition, legacy pandas loaders run
   behind a framework-side `LegacyLoaderAdapter`; the adapter dies at loader-plan
   Step 6.
5. **Units are declared, validated, and mandatory.** `raw_units` is a
   `cellpycore.units.CellpyUnits` (pint-parsable labels, checked by
   `validate_units()`) — never an ad-hoc dict, never float scale factors.
6. **Metadata is drafts.** `test_meta` is a partial `TestMeta`; `cell_meta` an
   optional partial `CellMeta`. Population policy belongs to `MetaResolver`.
7. **Capability metadata is class-level** (`name`, `instrument`,
   `supported_suffixes`) so the registry routes without instantiating.
8. **Sources are path-likes the framework resolves** (config plan §5b): OtherPath /
   remote handling is centralized in one boundary function; loaders receive a local
   `Path` and never encode ssh/scheme semantics (keeps the fsspec door open).

### 5.2 The contract (target home: `cellpy/instruments/contract.py`)

```python
from __future__ import annotations

from dataclasses import dataclass, field
from pathlib import Path
from typing import ClassVar, Protocol, runtime_checkable

import polars as pl

from cellpycore.metadata.models import CellMeta, TestMeta
from cellpycore.units import CellpyUnits


@dataclass(frozen=True, slots=True)
class LoaderResult:
    """What an instrument loader hands back to the loading framework.

    The frame must conform to the harmonized-raw schema (config.RawCols,
    epoch_time_utc as int64 ns UTC, datapoint_num as a column — never an
    index). Meta objects are *drafts*: only fields the source file actually
    knows are filled. Provenance (source_*, raw_file_names, loaded_datetime,
    uuid, test_id, mask) is stamped by the framework.
    """

    raw: pl.DataFrame
    raw_units: CellpyUnits
    test_meta: TestMeta
    cell_meta: CellMeta | None = None
    warnings: tuple[str, ...] = field(default=())


@runtime_checkable
class InstrumentLoader(Protocol):
    """Structural contract for instrument loaders (cellpy 2).

    Implementations do NOT need to import or inherit anything from cellpy;
    conformance is structural. Register via the ``cellpy.loaders`` entry-point
    group. Loaders must be stateless across calls. Built-in loaders implement
    load() as harmonize(parse(source), declarations) — see the loader-port
    plan §2.1; only the LoaderResult is judged.
    """

    # -- capability metadata (class-level; registry routes on these) --------
    name: ClassVar[str]                        # e.g. "arbin_res"
    instrument: ClassVar[str]                  # e.g. "arbin"
    supported_suffixes: ClassVar[tuple[str, ...]]  # e.g. (".res",)

    # -- optional cheap pre-check -------------------------------------------
    def can_load(self, source: Path) -> bool:
        """Cheap sniff (suffix/magic bytes). Must not fully parse the file."""
        ...

    # -- the one required operation -----------------------------------------
    def load(
        self,
        source: Path,                             # local; framework resolves
                                                  # OtherPath/remote first (§5.1.8)
        *,
        instrument_config: object | None = None,  # typed per-instrument model
                                                  # from cellpy.config.instruments
        **kwargs: object,                         # loader-specific knobs, must be
                                                  # ignorable (lenient) when unknown
    ) -> LoaderResult:
        """Parse one source into a LoaderResult.

        Must raise LoaderError (wrapping vendor parse errors) on failure —
        never return a partial result silently (conventions plan §1).
        """
        ...
```

### 5.3 Registration and discovery

```toml
# in a loader package's pyproject.toml (works for cellpy's own built-ins too)
[project.entry-points."cellpy.loaders"]
arbin_res = "cellpy.instruments.arbin_res:ArbinResLoader"
neware_xlsx = "cellpy.instruments.neware_xlsx:NewareXlsxLoader"
mycycler = "cellpy_mycycler.loader:MyCyclerLoader"   # third-party
```

The registry (`cellpy/instruments/registry.py`) loads the `cellpy.loaders`
entry-point group lazily on first use, `isinstance`-checks against
`InstrumentLoader`, **validates each loader's declarations at registration time**
(loader plan §2.2 — a typo'd declaration fails at import of the configuration), and
routes by explicit `instrument=` argument first, then by `supported_suffixes` +
`can_load()`. Built-ins register through the same mechanism — no privileged path.

### 5.4 Framework responsibilities (not the loader's)

- Resolve the source: OtherPath/remote → local temp copy via the single boundary
  function in `cellpy_file/` (§5.1.8).
- Run `harmonize()` for declaration-based loaders; run `validate_raw_frame` strict on
  every load.
- Stamp provenance on the draft `TestMeta` (`source_kind=FILE`, `source_type`,
  `source_uri`, `source_uuid` = stable file-identity hash, `raw_file_names`,
  `loaded_datetime=now`, `uuid` = uuid4 at first load, preserved through
  save/load/merge).
- Assign `test_id` (and `mask=True`) into the raw frame rows.
- Run `MetaResolver` over (user kwargs, journal/db, drafts, config
  `ScienceDefaults`) to produce final `CellMeta` / `TestMetaCollection`.
- Convert meta quantities to cellpy units **once, at ingestion**.
- During transition: wrap legacy pandas loaders in `LegacyLoaderAdapter`
  (frame + header conversion via the mapping; unit-dict → `CellpyUnits`).

### 5.5 Conformance test-kit (ships with cellpy)

`cellpy.instruments.testing.check_loader(loader, fixture)` asserts:

1. `LoaderResult` types exact (polars frame, `CellpyUnits`, `TestMeta`).
2. The frame passes the harmonized-raw schema check (columns ⊆ `RawCols`, dtypes per
   `dtype_map()`, int64-ns timestamps, monotone `datapoint_num`, no index promotion).
3. `raw_units` validates (every label pint-parsable).
4. The draft `TestMeta` contains **no** provenance fields (a filled `source_uri` in a
   draft is a contract violation).
5. `can_load()` is cheap (time/IO budget on a large dummy file).
6. Determinism: two `load()` calls on the same file give equal results.
7. **Reset-granularity property test** (loader plan §5 — the correctness landmine):
   per-cycle capacities recomputed from the normalized raw equal the legacy pipeline's
   summary capacities on the loader's golden fixture.

Fixture convention (F8): committed small vendor file + expected harmonized parquet,
regenerated by script only, never by hand.

### 5.6 Open questions on the contract (iterate here)

- ~~**Multi-test sources**~~ — **resolved (decision, 2026-07-19, maintainer):** `load()`
  **always returns `tuple[LoaderResult, ...]`**, including for the single-test case.
  Uniform over `LoaderResult | list`: every caller writes one unpacking path, and a
  format that grows multi-test support later does not become a breaking change for its
  consumers. Frozen with the Protocol in 2.0 ([jepegit/cellpy#210](https://github.com/jepegit/cellpy/issues/210)).
- **Streaming/chunked loading** for very large files and the live/incremental path
  (utils plan wave 4): a second, optional protocol (`SupportsIncrementalLoad`) rather
  than complicating the base contract. Design it together with wave 4, since core's
  incremental summarization is the consumer.
- **`instrument_config` typing:** `object` now; tighten once the config plan's typed
  per-instrument models land.
- ~~OtherPath in the signature~~ — **resolved** (config plan §5b): loaders get a plain
  local `Path`; the framework resolves remote sources at one seam.

---

## 6. Timeline — which plan, in which order

The release plan's branch strategy shapes everything: **trunk-based**, all
behavior-preserving preparation on master (v1.x stays green), the two genuine
flag-days (native-headers Phase 3 + polars Phase C) combined into **one short-lived
flip branch** (>4 weeks alive = stop and re-slice), and `v1.x` forking **at** the
flip. Cross-repo rule (F9): core-first for additions (core PR → PyPI release →
cellpy re-pin), cellpy-first for removals.

### Stage 0 — foundations ✅ complete (2026-07-11)

All twelve issues closed (tracking #439); deliverables and the six #438 decisions
listed in [stage0-github-issues.md](stage0-github-issues.md). Highlights: golden
fixture convention + regeneration script, cellpy-file round-trip characterization
(incl. the limits-prefix trap pin), config inventory contract, per-loader and curve
golden snapshots, value-parity comparator (`tests/parity.py`), benchmark harness with
GHA ubuntu-latest baselines, `cellpy/_deprecation.py` + exception-tree stubs.

### Stage 1 — v1.x-safe preparation on master ✅ complete (2026-07-15)

Issue mapping in [stage1-github-issues.md](stage1-github-issues.md) (tracking #459,
closed). All Stage-1 tracks shipped: config stack + prms shim (#452–#454),
`cellpy_file/` + dormant `translate.py` (#446–#449, #458), header cleanup + polars
Phase A (#455, #457), core seam re-pins (#450–#451; core#115–#118 → **cellpycore
0.2.0**), conventions + easyplot deprecation (#456, #479). §7 behavior-delta
sign-off closed with the tracking issue.

**Timeline note (kept from 2026-07-14):** the loader `harmonize()` port was
deliberately *not* in Stage 1 — the flip used `to_native()` at ingestion. The port
landed in Stage 3 (row 3.5) and retired `LegacyLoaderAdapter`.

### 6.1 The final legacy (v1.x) release — ship gate

**When can we ship the last 1.x?** At the **Stage-1 exit**, from the same commit the
flip branch starts and the `v1.x` maintenance branch forks from. One maximally
prepared release: every behavior-preserving refactor in, every deprecation warning
users should hear *before* 2.0 exists (easyplot ✅, prms shim with #453, the v<8
freeze message in `cellpy convert` ✅), plus `setup migrate` so users convert configs
early. It then gets bugfix-only support for 12 months from the 2.0 release date
(decision #438-6).

Ship **after** these close, in dependency order:

1. core#115 → core tag + cellpy re-pin → **#451** (converter delegation).
2. core#116 → core tag + re-pin → **#458** (dormant `translate.py`; its
   v8→native→v8 round-trip is simultaneously the flip gate).
3. **#453** M2 (call-site migration, in flight) + M3 (kill import-time init).
4. **#454** (`cellpy setup` writes TOML + `setup migrate` UX).
5. **#457** (Phase A de-indexing; tiered benchmark gate in band).
6. **Behavior-delta sign-off (§7):** every observed 1.0.3→1.0.4 delta confirmed
   intended (→ release notes) or fixed as a regression. The CE inversion is the
   headline item — do not ship the final legacy release with it undecided.

**Not** gating: core#117 (meta mapping) and core#118 (curves) — additive core work
consumed from Stage 2/3 on; include if ready. Everything already closed
(#446–#450, #452, #455, #456, #479) is in.

### Stage 2 — the flip ✅ complete (2026-07-18; **v2.0.0a5**)

Native-headers Phase 3 + polars Phase C shipped on one short-lived branch:
`CellpyCell.core` → lean `CellpyCellCore`, frames native-named **and** polars,
rename sandwich deleted, value-parity oracle with the documented exception list
(§7 + F4 IR). `v1.x` maintenance branch forked at the flip. Loaders that were still
legacy-shaped kept working through `to_native()` at ingestion until Stage 3.5.

### Stage 3 — 2.0 assembly 🟡 release gate (assembly ✅; **v2.0.0rc1** 2026-07-24)

Issue mapping, the 2.0-vs-2.1 scope decision and the sequencing constraints live in
[stage3-github-issues.md](stage3-github-issues.md) (tracking
[#575](https://github.com/jepegit/cellpy/issues/575)).

**Scope correction (2026-07-19) to row 3.2 below:** only **two** of the four utils
redesign plans are 2.0-blocking — **ica** (its output frame is a data contract) and
**plotting** (2.0 already breaks that surface via the easyplot removal). **batch v3**
and the **collectors** collection redesign move to Stage 4 behind facades. The test
applied: 2.0 finalizes what cannot be shimmed; API sprawl that rides on `warn_once`
is cleaned in 2.1 on the conventions cadence.

| # | Work | Status | Plan |
|---|---|---|---|
| 3.1 | Native-headers Phase 4: `c.schema` public, `headers_*` → attribute shim; v9 + `convert --to v9` | ✅ #558, #569, #573 | native-headers |
| 3.2 | Utils waves 1–2; **ica** + **plotting** redesigns in 2.0 (`units_label` pulled forward); batch/collectors deferred | ✅ #564, #566, #567, #591 | utils + redesign plans, units Phase 4 |
| 3.3 | Metadata Steps 3–6: loader drafts, `MetaResolver`, v9 meta persistence, merge, journal | ✅ #562, #563 | metadata |
| 3.4 | Config Steps 6–7: secrets hardening, generated docs | ✅ #565 | config |
| 3.5 | Loader port: `harmonize()` + declarations + pilot → tiers 1–2 → tier-3 decisions; retire `LegacyLoaderAdapter` | ✅ #210, #559–#561 | loader |
| 3.6 | Release: deps/docs/DEPRECATIONS/compat matrix → **cellpy 2.0** | 🟡 **rc1 shipped**; stable via [#574](https://github.com/jepegit/cellpy/issues/574) | release |

**Remaining for Stage 3 exit:** soak `v2.0.0rc1` → tag stable `v2.0.0` from clean
`master` → conda-forge feedstock bump → announce the 12-month `v1.x` window → close
#574 / #575. [#655](https://github.com/jepegit/cellpy/issues/655) (fixture gaps) is
explicitly non-blocking.

### Stage 4 — the complete system (2.0.x / 2.1) ⬜

Utils waves 3–4 (ica/ocv_rlx on curves; live/incremental rebuilt on core's engine or
deliberately dropped) · F6 feature menu (test-level summaries, `exclude_step_types`,
native-only columns in reports) · docs/CLI inventory (G11) · SPEED-30 headers behind the Schema indirection ·
**2.1: shim removals per `DEPRECATIONS.md`** (prms, headers_*, property facade,
legacy curve-frame names). *(`easyplot` removed at 2.0, issue #438 — not a 2.1 shim.)*

---

## 7. Known behavior deltas (1.0.3 → 1.0.4/2.0) — the delta register

**New finding (2026-07-09, absorbed here 2026-07-14).** The validation notebooks in
`cellpy-core-dev/notebooks/` (marimo; frozen 1.0.3 reference parquet in
`cellpy-core-dev/data/reference_v103/`) compared real cellpy 1.0.3 output against
1.0.4a3 and against the cellpycore engine on six test files. Full details:
[cellpy-v103-vs-v104a3-observations.md](cellpy-v103-vs-v104a3-observations.md).
The deltas are **discrete, systematic convention changes plus one bug fix** — not
numerical drift. Raw data is identical; capacities, end voltages, IR and c-rates
agree to 1e-6.

**Rule:** every row below gets an explicit verdict — *intended* (→ release notes +
an entry in the #434 comparator's exception list) or *regression* (→ fixed before
the final legacy release, §6.1 gate 6). The comparator carries a named exception
list; it never silently widens tolerances.

| # | Delta (observations §) | Proposed verdict | Where it lands |
|---|---|---|---|
| Δ1 | `coulombic_difference` family sign flip (§3.1: exactly ×−1, incl. cumulated + specific variants, `shifted_*`, `cumulated_ric*`) | decide: intended convention change? | release notes + comparator exception |
| Δ2 | `coulombic_efficiency` inverted — reciprocal, anode cycle-mode (§3.2). **Most user-facing delta found**: first-cycle CE moves across 100%; user QC thresholds change answers | decide — headline release-note item either way; if unintended it is a blocker | release notes + comparator exception, or fix |
| Δ3 | Six `shifted_*_{gravimetric,areal,absolute}` summary columns dropped (§3.3) | consistent with native-headers D3 "drop and recompute on demand" — confirm | D3 policy table |
| Δ4 | Seven `reference_voltage_*` step aggregates dropped (§2.3) while raw keeps the column | decide: return via the native path (ties to core#43 `ref_potential`)? | loader/native path, core#43 |
| Δ5 | Cycle-9 step misclassification in 1.0.3 fixed (empty type, negative duration → correct `cv_charge`, §2.1) | bug fix — keep; document like the F4 IR case | comparator exception (per-file) |
| Δ6 | steps `sub_type`: `"None"` string → real null (§2.2) | normalize in the legacy bridge or release-note; cheap either way | legacy bridge |
| Δ7 | `discharge_c_rate` differs between legacy and cellpycore engines (§4) | **investigate before the flip** — likely nominal-capacity plumbing; unexplained engine deltas are not acceptable oracle exceptions | flip-gate task |

Δ1/Δ2 look like two faces of one convention decision (charge−discharge direction and
the CE ratio direction under anode mode) — decide them together, once, and record in
the decision-register style of #438.

---

## 8. Developer guardrails — safe-to-use surfaces

### 8.1 Safe-to-use API for util developers (start improving plotutils/batch now)

The usage report §5 showed the utils' real surface is small; the utils plan
(principle 1) promotes exactly that surface to the documented **utils contract**.
If you start improving a util *today*, code against the first column and your work
survives the flip with at most a mechanical rename pass.

**Safe — the utils contract (kept working through 2.x, shim-protected at the flip):**

| Surface | Notes for new code |
|---|---|
| `c.data` → `data.summary` / `data.steps` / `data.raw` | The only sanctioned door to the frames. Read via header *objects*, never string literals (`"voltage"`, `"cycle_index"` are what the flip renames). |
| `c.empty` | Guard before processing, as today. |
| `c.cell_name` | Titles/labels. |
| `c.get_cap` / `get_ccap` / `get_dcap` / `get_ocv` | Survive as thin wrappers over `cellpycore.curves`; signatures stable. The *returned frame's* column names get the spec'd `CurveCols` at the flip (legacy names via shim for one release) — don't hardcode `"capacity"`/`"voltage"` on results either. |
| `c.get_cycle_numbers` | Stable. |
| `c.make_summary()` / `c.make_step_table()` | Already delegate to core; stable. |
| `c.load` / `c.from_raw` / `c.save` / `c.to_csv` | Stable entry points (internals move; signatures keep). |
| `c.set_mass` / `c.set_nom_cap` / `c.mass = …` | Kept, routed through the meta facade during deprecation. |
| `c.drop_from(...)` | Returns a new object — keep the `c = c.drop_from(...)` idiom. |
| `c.cellpy_units`, `c.data.raw_units` | Fine for *reading unit values*. Use `c.data.raw_units` (not the `c.raw_units` forwarder) — one access path. Do **not** hand-compose axis-label f-strings from them; `units_label(prop, mode=…)` replaces that (wave 2). |

**Use with care (works now, changes at the flip — write it the new way when you touch it):**

- `c.headers_summary` / `headers_step_table` / `headers_normal` — become the
  attribute-level shim; after the flip the public header API is `c.schema`. Attribute
  access (`hdr.charge_capacity`) migrates mechanically; string literals don't.
- `c.data.meta_common` / meta forwarders (`c.data.nom_cap`, `.loading`,
  `.cell_name`) — become `c.data.meta.<field>` and `c.data.tests[tid].<field>`
  (facade keeps ~8 names warm through 2.0).
- pandas index idioms (`summary.loc[cycle, …]`, `raw` indexed by `data_point`,
  journal `pages.loc[cell_id]`) — the keys-in-columns rule is law and lint-enforced;
  use column filters/joins even while frames are still pandas (Phase A removes the
  indexes ahead of the flip).

**Avoid (dies without a long deprecation, or already a known trap):**

- Writing `c.ensure_step_table = True` or `c.overwrite_able = False` — these become
  load/save kwargs (utils plan principle 1).
- Mutating `prms.*` globals from utils — pass explicit kwargs; use
  `cellpy.config.override()` once it lands.
- Anything `_`-prefixed — private API breaks without deprecation (conventions §3).
- Anything in the usage report's §3 "not used by any util" lists — those members have
  **no parity guarantee** through the migration; don't add new dependencies on them
  without flagging it in the utils plan.
- Storing a `CellpyCell` in an attribute named `data` (the `ocv_rlx`
  `self.data.data.steps` trap — being renamed in wave 3).

### 8.2 Safe-to-use surface for loader developers

If you write or port a loader now, build against the loader plan's target and your
work is flip-proof:

**Safe — build on these:**

| Surface | Notes |
|---|---|
| Vendor `parse()` kept separate from normalization | The two-stage rule (§5.1.2). Your parsing is the durable part; never mix renames/casts into it. |
| Declarations in the configuration module | `column_map` (vendor→**native** names), `raw_units` (a `CellpyUnits`), `tz` rule, `reset_granularity` per cumulative column, `aux_map`, optional `post_hooks`. Validated at registration. |
| `cellpycore` toolbox | `RawCols` + `RawCols.dtype_map()` (dtype truth), `validate_raw_frame`, `Data.from_raw_frame`, `cellpycore.timestamps` (ns ↔ datetime), `StepType`/`StepMode`/`CycleType` enums, `CellpyUnits` + `validate_units`, the harmonized-raw spec (`docs/data_format_specifications/harmonized_raw.md`). |
| Draft metadata | Return a draft `TestMeta` (+ partial `CellMeta` only when the file carries masses etc.). Fill only file-known fields; leave every provenance field empty (the test-kit rejects filled ones). |
| Fixture + tests | Committed vendor sample + expected harmonized parquet (script-regenerated, F8); run `check_loader` incl. the reset-granularity property test — it is the test that catches the silent capacity-corruption bug. |

**Avoid:**

- Promoting `data_point` to a pandas index — `datapoint_num` is a column, full stop.
- Poking metadata onto `Data` attributes (`data.start_datetime = …`) — drafts only.
- Ad-hoc `raw_units` dicts or float scale factors — validated `CellpyUnits` only.
- Emitting legacy `HeadersNormal` names as your target — declare vendor→native; the
  legacy dialect exists only inside `cellpy_file/legacy_read` and the shims.
- Milliseconds or naive datetimes without a declared `tz` rule — `epoch_time_utc` is
  int64 ns UTC; the naive-datetime rule is a shared decision (native-headers D3),
  recorded on `TestMeta.time_zone`.
- Reading `prms.Instruments.*` (or any global config) inside a loader — the typed
  per-instrument model arrives as the `instrument_config` argument.
- Encoding ssh/OtherPath/remote semantics — you receive a local `Path`; the framework
  owns remote resolution (§5.1.8).

---

## 9. Iteration log

- **2026-07-09 (v1)** — initial version: total-solution synthesis, MVP vs. complete
  illustrations, design-pattern commitments, formal loader contract (§5).
- **2026-07-09 (v2)** — reconciled with the gap analysis and the six new plans:
  added the plan-set index (§0); **redefined the MVP** to match the release plan's
  2.0 support matrix (the flip — native headers + polars + v9 — is in 2.0; the old
  "keep v8/legacy dialect" MVP is now the Stage-1 preparation state); updated both
  illustrations (curves module, v9/convert, translate-at-I/O, conventions);
  extended the pattern table (declarative loader configs, two-stage pipeline,
  translate-once-at-I/O, single exception hierarchy, one deprecation cadence);
  reconciled §5 with the loader plan's `parse()`+`harmonize()` design, added the
  reset-granularity check to the test-kit, resolved the OtherPath open question
  (config plan §5b); added the timeline chapter (§6: Stages 0–4, flip gate,
  cross-repo merge order) and the safe-to-use API sub-chapters for util developers
  (§6.1) and loader developers (§6.2).
- **2026-07-14 (v3)** — post-implementation checkup after Stage 0 completed and
  Stage 1 reached ~60%: added the status dashboard (top) with a drift verdict
  (no structural drift; `cellpy_file/` and `cellpy/config/` match their plans);
  rewrote the timeline (§6) with per-track/per-issue status and two corrections —
  the loader port moved from Stage 1 to Stage 3 (the flip uses `to_native()` at
  ingestion instead of waiting for ported loaders) and the benchmark gate is tiered
  on GHA baselines (#476); added the **final-legacy-release ship gate** (§6.1:
  after #451, #453 M2/M3, #454, #457, #458 + core#115/#116 re-pins + the §7
  sign-off); added the **behavior-delta register** (§7, Δ1–Δ7) absorbing the
  1.0.3-vs-1.0.4a3 validation-notebook findings, which no plan previously
  referenced; renumbered developer guardrails to §8 and this log to §9; folded the
  #438 decisions (timezone, curves home, v9 container, IR switch, easyplot at 2.0,
  12-month maintenance window) into §1/§6.
- **2026-07-24 (v4)** — status sync after Stage 3 assembly closed and
  **v2.0.0rc1** shipped: dashboard + §1 "where we already are" + §6 Stages 1–3
  marked complete through assembly; Stage 3.1–3.5 ✅ with issue refs; Stage 3.6
  held open on [#574](https://github.com/jepegit/cellpy/issues/574) (soak → stable
  `v2.0.0` → conda-forge); noted non-blocking [#655](https://github.com/jepegit/cellpy/issues/655);
  drift check refreshed (no structural drift — remaining gap is release process).
- **2026-07-28 (v5)** — status sync after stable **v2.0.0** shipped (2026-07-26,
  [#574](https://github.com/jepegit/cellpy/issues/574)/[#575](https://github.com/jepegit/cellpy/issues/575)
  closed) and Stage 4 (**2.1**) landed: dashboard Stage 3 → ✅ complete, Stage 4 →
  ✅ essentially complete (`v.2.1` milestone 37 closed / 2 open). All Stage-4 epics
  merged — batch v3 (A) + collectors (B), utils C1 (**C2 #164 live/incremental → 2.2**),
  F6 menu (D), the breaking shim removals (E1–E5), docs (G1–G3), cellpy-core
  doc-sync (H1), and test-fixture gaps H2 [#655](https://github.com/jepegit/cellpy/issues/655)
  Phase 1 (Phase 2 dataset-curation deferred; surfaced loader-robustness
  [#761](https://github.com/jepegit/cellpy/issues/761)). Drift check refreshed (no
  structural drift; `cellpy.batch`/`cellpy.collect` packages + shim removals match
  their plans). Remaining: [#345](https://github.com/jepegit/cellpy/issues/345)
  verify/close + [#696](https://github.com/jepegit/cellpy/issues/696) umbrella (2.1 tag).
