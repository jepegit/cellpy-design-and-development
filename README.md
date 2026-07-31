# cellpy-design-and-development

Authoritative **cellpy 2 architecture and migration plans** for the
`cellpy-workspace` multi-repo checkout.

Formerly **`architecture-plan`** (GitHub + local folder renamed 2026-07-31).

**Agents / humans: start at [`CURRENT.md`](CURRENT.md).**

## Layout

```text
cellpy-design-and-development/
├── CURRENT.md              ← what we’re working through now
├── PATHS.md                ← basename → current path (historical links)
├── README.md               ← this file
├── ecosystem/              ← durable: what the system is
├── roadmap/                ← coordinator + stage issue sets
├── active/                 ← plans still guiding open work
├── archive/                ← executed topic plans (foundations / redesigns)
└── research/               ← scans & evidence (not decisions)
```

| Doc / folder | Use when you need… |
|---|---|
| [`CURRENT.md`](CURRENT.md) | Active stage, open epics, links into active plans |
| [`PATHS.md`](PATHS.md) | Old flat filename → where it lives now |
| [`ecosystem/`](ecosystem/) | cellpy vs cellpy-core roles, module layout, conventions |
| [`roadmap/`](roadmap/) | Stage dashboard, gap analysis, stage issue sets |
| [`active/`](active/) | Designs still constraining open / near-term work |
| [`archive/`](archive/) | Finished topic plans (provenance) |
| [`research/`](research/) | Reports that informed the plans |

## Sibling repositories

| Checkout | Role |
|---|---|
| `../cellpy/` | Consumer library; issue-flow under `.issueflows/` |
| `../cellpy-core/` | Compute engine |
| `cellpy-design-and-development/` (this repo) | Plan documents |
| `../cellpy-examples/` | Example notebooks/scripts |
| `../cellpy-simple-gui/` | Desktop explorer app (FastAPI + pywebview) |

Canonical remote: [`cellpy/cellpy-design-and-development`](https://github.com/cellpy/cellpy-design-and-development).
Cloud mirror: [`jepegit/cellpy-design-and-development`](https://github.com/jepegit/cellpy-design-and-development).

## Note for agents

Plans **used to** live under a `code-reviews/` folder in the workspace, then in a
repo named `architecture-plan`. Prefer **`CURRENT.md`** and the folders above; if
you only have a basename (`cellpy2-….md`), look it up in [`PATHS.md`](PATHS.md).

Cross-reference in cellpy:
`cellpy/.issueflows/04-designs-and-guides/cellpy-workspace-repos.md`.
