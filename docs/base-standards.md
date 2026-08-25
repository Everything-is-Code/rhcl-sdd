# Agent guide — RHCL SDD store (`rhcl-sdd`)

OpenSpec store for **[migration-toolkit-rhcl](https://github.com/Everything-is-Code/migration-toolkit-rhcl)**. Specs and docs live here; product code lives in the sibling folder `../migration-toolkit-rhcl/`.

## Store identity

- **GitHub repo**: `Everything-is-Code/rhcl-sdd`
- **Local folder**: `migration-toolkit-sdd/` (sibling to product repo)
- **OpenSpec store id**: `rhcl-sdd`
- **CLI**: `openspec list --store rhcl-sdd`

## Workspace (sibling layout)

Open **`rhcl-sdd.code-workspace`** in this repo — it loads both folders side by side in Cursor:

| Folder in workspace | Path |
|---------------------|------|
| `rhcl-sdd` | this store |
| `migration-toolkit-rhcl` | `../migration-toolkit-rhcl` |

`/opsx-apply` reads artifacts here and edits files under `migration-toolkit-rhcl/`.

## Commands (Cursor)

| Command / skill | Purpose |
|-----------------|---------|
| `/opsx-explore` | Explore problem space |
| `/opsx-propose` | Create change artifacts — **solution-architect** owns `design.md` |
| `/opsx-apply` | Implement tasks in product repo |
| `/opsx-archive` | Complete change |
| `enrich-us` | Enrich GitHub issue before propose |

## Agent roles

| Agent | Use when |
|-------|----------|
| `solution-architect` | Architecture, #40 refactor, `design.md`, interfaces |
| `backend-developer` | Quarkus / ConversionService implementation |
| `frontend-developer` | React / PatternFly implementation |
| `product-strategy-analyst` | Epic scope / prioritization (optional) |

Canonical definitions: `ai-specs/agents/*.md`

## Read first

1. `docs/base-standards.md`
2. `docs/sdd-backlog.md`
3. `openspec/config.yaml`

## Do not

- Add application code here (except `docs/`, `ai-specs/`, `openspec/`)
- Use Engram for new work
- Create `openspec/` in the product repo

## Language

English only.
