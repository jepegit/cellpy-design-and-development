# cellpy ecosystem — architecture overview

Durable primer for the two-package system. Migration *schedules* live under
[`../roadmap/`](../roadmap/) and [`../archive/`](../archive/); this page states
what the system **is**.

**Companion docs:** [module layout](cellpy2-module-layout-planning.md) ·
[conventions](cellpy2-conventions-plan.md) · coordinating plan
[`../roadmap/cellpy2-architecture-plan.md`](../roadmap/cellpy2-architecture-plan.md).

Per-repo migration guides (authoritative for the package boundary):

- `cellpy-core/.issueflows/04-designs-and-guides/cellpy-core-migration.md`
- `cellpy-core/.issueflows/04-designs-and-guides/cellpy-core-integration-into-cellpy.md`
- `cellpy/.issueflows/04-designs-and-guides/cellpy-workspace-repos.md`

---

## One sentence

**`cellpycore`** is a small, pure, polars-based compute engine (shapes and tools).
**`cellpy`** is the application layer (content and policy): loaders, config,
metadata population, persistence, plotting, batch, CLI.

## Package roles

| Package | Owns | Does not own |
|---|---|---|
| **cellpycore** (`cellpy/cellpy-core`) | Schemas (`Cols` / `RawCols`), step & summary engines, curve extractors, unit converters, metadata *models/scaffolding*, legacy header mapping, merge/update primitives | Instrument I/O, user config, populated experiment metadata, plotting, batch orchestration, CLI |
| **cellpy** (`jepegit/cellpy`) | Instrument loaders + `harmonize()`, config/secrets, attaching real metadata, `.cellpy` file I/O, plotting, batch/collect, utils, Typer CLI | Reimplementing step/summary math |

## Downstream apps (same workspace)

| Checkout | Role |
|---|---|
| **cellpy-simple-gui** (`cellpy/cellpy-simple-gui`) | Desktop explorer (FastAPI + pywebview) on cellpy ≥ 2.1 — and a **dogfood / discovery** vehicle for the library |
| **cellpy-examples** (`cellpy/cellpy-examples`) | Tutorial notebooks/scripts |

**`cellpy-simple-gui` is not only a product demo.** It exercises real researcher
workflows (import raw → journal/metadata → collect/plot → project save) so we can
**surface API pain points, missing conveniences, and UX-driven feature ideas** that
should feed back into `cellpy` (and sometimes Stage plans). Treat friction found
there as design input, not just app bugs.

These sit *above* cellpy; they do not own schemas or the compute engine.

## Layering

```text
┌─────────────────────────────────────────────────────────────┐
│  cellpy — application                                        │
│  get / CellpyCell · loaders · config · meta · plotting ·     │
│  batch/collect · cli_api · .cellpy persistence               │
└───────────────────────────┬─────────────────────────────────┘
                            │ plain values only
                            │ (frames, schema, scalars — no pint,
                            │  no config objects, no file handles)
┌───────────────────────────▼─────────────────────────────────┐
│  cellpycore — engine                                         │
│  Data · make_step_table · make_summary · curves · units ·    │
│  metadata models · legacy bridge · merge/update              │
└─────────────────────────────────────────────────────────────┘
```

## Hard boundaries

1. **Seam = plain values.** Nothing that needs cellpy’s runtime (paths, secrets,
   pint) crosses into core.
2. **Metadata.** Core may ship schemas and helpers; it must not *require*
   populated metadata on `Data`. Attaching experiment metadata is cellpy’s job.
3. **Headers.** Native `Cols` / `RawCols` end-to-end at runtime; legacy dialect
   translated at I/O boundaries (and via the legacy bridge for old fixtures).
4. **Parity by tests.** Contract tests + golden parquet — not vigilance — keep
   cellpy and cellpycore aligned.
5. **Dev wiring.** cellpy pins `cellpycore` on PyPI. Local dual-repo: editable
   install of the sibling checkout; do not commit path overrides in
   `[tool.uv.sources]`.

## Data path (happy path)

1. Loader reads instrument file → harmonized raw (`RawCols`, provenance columns).
2. cellpy builds / updates a `Data`-backed `CellpyCell` (core step + summary).
3. User API reads frames via `c.data.*` and schema-resolved names (`c.schema`).
4. Optional: plot, batch, collect, save v9 `.cellpy` (zip-of-parquet + `meta.json`).

## Where detail lives

| Topic | Document |
|---|---|
| Suggested on-disk package tree | [cellpy2-module-layout-planning.md](cellpy2-module-layout-planning.md) |
| Exceptions, logging, deprecations | [cellpy2-conventions-plan.md](cellpy2-conventions-plan.md) |
| Stage status & release sequencing | [../roadmap/cellpy2-architecture-plan.md](../roadmap/cellpy2-architecture-plan.md) |
| What we’re working through now | [../CURRENT.md](../CURRENT.md) |
