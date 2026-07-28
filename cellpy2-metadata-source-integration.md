# External metadata sources — pluggable API integration (design)

Status: **design only** (2026-07-29). Post-2.2 candidate. Makes concrete the **metadata plan
[Step 7](cellpy2-metadata-handling-plan.md) "DB/API (JSON-LD) — deferred, but keep the door
open"**, now that there is a **concrete consumer**: *BatBase*, an in-house Django + PostgreSQL
app holding cell/test metadata, exposed over an HTTP API. Related: #243 (dbreader refactor —
adjacent), [[battinfo-alignment]] (vocabulary), metadata-plan open-question 6 (identity /
ontology parking → this is where `CellMeta.uuid` gets added).

## 1. Goal

Let cellpy **pull** (and optionally **push**) cell/test metadata from an **external source**,
so a user can enrich a loaded cell or a batch journal from their lab's system of record
instead of retyping masses, loadings, protocols, etc.

**Source-agnostic by construction.** BatBase is the *first adapter*, never the interface.
Anyone with another store (a different lab DB, a LIMS, a spreadsheet service) can implement
the same contract. **First transport assumption: an HTTP API** (GET to read, POST/PUT to
write) — that keeps the initial surface small; non-HTTP sources can implement the same
Protocol later (the contract is transport-agnostic; only the first *adapter kit* is HTTP).

```python
c = cellpy.get("cell_042.res")
c = c.fetch_meta(source="batbase", key="XCEL-042")   # pull, merge into c.data.meta
b = batch.from_source("batbase", project="LongLife")  # journal populated from the source
```

## 2. Current state — the seam is already reserved

cellpy 2 deliberately left the door open; do not rebuild this scaffolding:

- **`MetaResolver` already has the slot.** Metadata is resolved with a fixed precedence
  `kwargs > journal/db > raw file > config defaults` (`readers/meta_resolver.py`), via a
  `Layer` enum and a `Resolution` provenance record; **any mapping** (draft dataclass /
  pydantic / dict) is normalized and merged with per-field provenance. An external source is
  simply a **new contributor to the JOURNAL/DB layer** — authoritative lab metadata correctly
  outranks what a cycler wrote (`journal/db > raw file`), and provenance already records which
  layer won.
- **The linkable key exists.** `TestMeta.uuid` is minted at first load (uuid4) and preserved
  through save/load/merge — the metadata plan added this *specifically* so records are linkable
  when Step 7 got a consumer.
- **A db-source shape exists.** `dbreader.Reader` / `sql_dbreader.SQLReader` subclass
  `BaseSimpleDbReader` (`select_batch()` + `get_<field>(serial)`); today Excel- and SQL-backed.
  The journal/db already feeds the resolver as a *source, not a store*.
- **Secrets are handled.** Env-only `SecretStr` (config plan §5b) is where an API token/key
  lives — never a config file, never a URL parameter.
- **Step 7 stubs.** `fetch_from_db` / `push_to_db` are stubs awaiting exactly this consumer.

## 3. Design — a `MetadataSource` protocol + adapter, mirroring the loader contract

The established cellpy pattern for "third parties plug in without importing a base class" is the
loader contract ([architecture §5](cellpy2-architecture-plan.md)): a `typing.Protocol` +
entry-point registry + declarative mapping + a conformance test-kit. Reuse it wholesale.

### 3.1 The contract (cellpy owns it; BatBase is an adapter behind it)

```python
@runtime_checkable
class MetadataSource(Protocol):
    name: ClassVar[str]                 # "batbase"; registry routes on this

    def fetch(self, query: MetaQuery) -> tuple[MetaRecord, ...]:
        """Look up metadata for one or more cells/tests. Read-only.
        Returns partial CellMeta/TestMeta drafts + the source's own id/uri
        (for linkage). Empty tuple = not found (never raise on 'unknown')."""
        ...

@runtime_checkable
class SupportsMetadataPush(Protocol):        # optional, opt-in
    def register(self, record: MetaRecord) -> str:
        """Push a cellpy cell (uuid + chosen fields) to the source; return the
        source id. Side-effecting → only ever via an explicit user call."""
        ...
```

- `MetaRecord` = a **draft** `CellMeta` / `TestMeta` mapping (only source-known fields) **plus**
  `source_name` + `external_id` + `source_uri` for linkage and provenance. It is exactly the
  mapping shape `MetaResolver` already ingests.
- `MetaQuery` = the lookup key(s) the source understands — `serial`, `cell_name`,
  `project`+`sample`, an `external_id`, or the cellpy `uuid` once a link exists (§3.3).
- **Anti-corruption boundary — the answer to "the BatBase API isn't settled yet."** cellpy
  pins the `MetadataSource` Protocol and `MetaRecord`; the **BatBase adapter absorbs all API
  churn** at one seam. The adapter can even live in a **separate package** discovered via the
  `cellpy.metadata_sources` entry-point group (same mechanism as third-party loaders), so
  BatBase API changes ship in the adapter, never a cellpy release.

### 3.2 Wiring into `MetaResolver`

A registered source contributes its `MetaRecord` mapping to the **JOURNAL/DB layer**. Resolver
precedence is unchanged; provenance is extended so `Resolution` can name *which source* (not
just "journal/db") won a field. Multiple sources → a configured priority order. **Null-object /
graceful degradation** (a cellpy law): an unreachable or unknown source yields an **empty
layer** — the resolver still runs and the cell still loads. External metadata is never on the
hot path.

### 3.3 Identity & linkage (the real work — and where `CellMeta.uuid` lands)

cellpy's `uuid` is cellpy-minted and will *not* match BatBase's ids, so linkage needs:

1. A **query key** the source recognizes (serial / name / project+sample / external id).
2. A stored **back-link** on the cell: `external_id` + `source_uri` per source, so a re-fetch
   is deterministic and provenance is inspectable.
3. **`CellMeta.uuid`** — the metadata plan explicitly parked this "until the DB/API work gets a
   concrete consumer" (OQ6). This is that moment: add `CellMeta.uuid` as the cell-level join
   key, alongside the existing `TestMeta.uuid`.

Optional **push** (`SupportsMetadataPush`) registers a cellpy uuid into the source, closing the
loop (source id ⇄ cellpy uuid). Push is **read-after-write side-effecting → explicit call
only**, never automatic (fail-loud/permission posture, conventions §4).

### 3.4 Declarative field mapping + vocabulary

Source field → `CellMeta`/`TestMeta` field is a **declaration** (validated at registration,
like loader `column_map`), not hand-wiring in the adapter body. Controlled vocabularies
(`test_family`/`test_type`/`source_type`) normalize toward **BattINFO/EMMO terms** here — this
is the natural home for the [[battinfo-alignment]] mapping the metadata plan flagged (OQ4/OQ6).

### 3.5 Auth, transport, dependencies

- **Auth:** `SecretStr` env-only (token / API key / basic); `base_url` + static headers in
  per-source config. Never in URL query strings (privacy).
- **Transport:** GET = pull, POST/PUT = push. Timeouts, retry/backoff, and a `check_connection`
  diagnostic (mirrors the remote-paths story).
- **Dependency budget:** keep the HTTP client **inside the adapter**, not cellpy core — since
  adapters can be plugin packages, the core stays lean (an `httpx`/`requests` dep rides with
  the source, not with `import cellpy`). Open question if the built-in adapter ships in-tree.

### 3.6 Conformance test-kit

`check_metadata_source(source, fixture)` (mirrors the loader test-kit §5.5) against a **fake
in-process HTTP source**: `fetch` returns well-typed `MetaRecord` drafts (no provenance fields
pre-filled); unknown key → empty tuple, not an exception; determinism; auth failure surfaces a
clear error, not a silent empty.

## 4. Work breakdown (post-2.2)

| # | Where | Item |
|---|---|---|
| 1 | cellpy | `MetadataSource` Protocol + `MetaRecord`/`MetaQuery` + `cellpy.metadata_sources` entry-point registry (mirror the loader registry) |
| 2 | cellpy | Resolver wiring: source → JOURNAL/DB layer; extend `Resolution` provenance to name the source; multi-source priority config |
| 3 | cellpy | `CellMeta.uuid` + per-source `external_id`/`source_uri` back-link; save/load round-trip in v9 `meta.json` |
| 4 | cellpy | Declarative source→meta field map + BattINFO-oriented vocab normalization |
| 5 | cellpy | Surface: `c.fetch_meta(source=, key=)`, `batch.from_source(...)`; caching + offline degrade (null-object) |
| 6 | adapter | **BatBase HTTP adapter** (first concrete source) — GET pull; optional POST push behind `SupportsMetadataPush`; `SecretStr` auth; in-tree or plugin package |
| 7 | tests | `check_metadata_source` kit + a fake HTTP source; resolver-precedence + linkage round-trip |

## 5. Why post-2.2 (not in the 2.2 scope)

It is a new cross-cutting subsystem (protocol + registry + resolver change + a new identity
field + an adapter) with a real correctness and **security** surface (auth, side-effecting
push, offline behavior). It is additive and gates nothing in 2.2. It also pairs with the
BattINFO vocabulary work and the identity/ontology parking (OQ6). Natural home: **2.3+**,
co-planned with SPEED-30's schema work only insofar as vocabulary/units touch headers.
Sequence the **read path first** (protocol + `fetch` + resolver + BatBase GET adapter); make
**push** a later, opt-in follow-on.

## 6. Open questions

- **Query key(s)** BatBase exposes first (serial? project+sample? a stable external id?).
- **Adapter placement:** built-in BatBase adapter in-tree, or a separate `cellpy-batbase`
  plugin package (keeps the HTTP dep out of core; matches third-party-loader precedent)?
- **Cache invalidation / offline:** how long is fetched metadata trusted; explicit refresh vs TTL.
- **Push permission model:** who may POST, and the confirmation UX (never silent writes).
- **Multi-source precedence** when two sources answer the same field.

## 7. Rejected alternatives (recorded)

- **Hardcoding BatBase into `MetaResolver`/cellpy.** Couples cellpy to one lab's unsettled API;
  violates the ports-and-adapters principle. The Protocol + adapter keeps BatBase churn at one
  seam and lets other sources plug in.
- **Embedding an HTTP client in cellpy core.** Bloats the dependency budget for a feature many
  users won't use. The transport dep rides with the (optionally third-party) adapter.
- **Credentials in config files or URL parameters.** `SecretStr` env-only; never in a URL.
- **Making the source authoritative on the hot path.** External metadata is a *layer*, not a
  requirement — engines must run with an empty source (null-object law).
