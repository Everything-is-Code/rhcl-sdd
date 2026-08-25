# RHCL SDD (`rhcl-sdd`)

OpenSpec **store** for spec-driven development of [migration-toolkit-rhcl](https://github.com/Everything-is-Code/migration-toolkit-rhcl).

Harness: [LIDR SpecBoot](https://github.com/LIDR-academy/lidr-specboot) + [OpenSpec](https://github.com/Fission-AI/OpenSpec).

## Sibling layout

```
rhcl/   (parent on disk — optional)
  migration-toolkit-sdd/    ← this repo (GitHub: rhcl-sdd)
  migration-toolkit-rhcl/   ← product code
```

## Quick start

```bash
npm install -g @fission-ai/openspec@latest
openspec store register . --id rhcl-sdd --yes
```

Open **`rhcl-sdd.code-workspace`** in Cursor (both repos in one window).

## Workflow

| Step | Action |
|------|--------|
| 1 | `enrich-us` on GitHub issue |
| 2 | `/opsx-propose <change-name>` |
| 3 | `/opsx-apply <change-name>` → edits `../migration-toolkit-rhcl/` |
| 4 | `/verify` + `/code-review` |
| 5 | `/opsx-archive` + `/commit` |

CLI: `openspec list --store rhcl-sdd`

## Key docs

| File | Purpose |
|------|---------|
| `docs/base-standards.md` | Single source of truth |
| `docs/conversion-architecture.md` | #40 Strategy/Registry/Contributor |
| `docs/sdd-backlog.md` | Prioritized changes ↔ GitHub issues |
| `openspec/config.yaml` | OpenSpec context + apply rules |

## Agents

| Role | File |
|------|------|
| Architect | `ai-specs/agents/solution-architect.md` |
| Backend | `ai-specs/agents/backend-developer.md` |
| Frontend | `ai-specs/agents/frontend-developer.md` |

## Next change

`conversion-strategy-registry` (GitHub #40) — after docs/skills baseline is stable.

## Replaces

- Engram-only SDD (legacy)
- `rhcl-ai` program AI docs (sunset planned; not in workspace)
