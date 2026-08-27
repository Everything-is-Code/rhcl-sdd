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
| 4 | `/verify-fe <change>` when `frontend/` or wizard/YAML changed (live 3scale Playwright) |
| 5 | `/verify <change>` — `mvn test`, `npm test`, `openspec validate` |
| 6 | `/code-review <change>` |
| 7 | `/adversarial-review` (fresh session recommended) |
| 8 | `/opsx-archive <change>` |
| 9 | `/commit #N` → PR in product repo |

Other commands: `/opsx-explore` · `/opsx-update` · `/opsx-sync`

Full skill map: `docs/skills-inventory.md`

CLI: `openspec list --store rhcl-sdd`

## Key docs

| File | Purpose |
|------|---------|
| `docs/base-standards.md` | Single source of truth |
| `docs/skills-inventory.md` | Skills, commands, canonical workflow |
| `docs/conversion-architecture.md` | Strategy/Registry/Contributor (#40) |
| `docs/sdd-backlog.md` | Prioritized changes ↔ GitHub issues |
| `openspec/config.yaml` | OpenSpec context + apply rules |

## Agents

Canonical source: `ai-specs/agents/` (mirrored to `.cursor/agents/` and `.claude/agents/`).

| Agent | When to use | File |
|-------|-------------|------|
| `solution-architect` | Before large changes — `design.md`, API contracts, cross-layer boundaries; does not implement | `ai-specs/agents/solution-architect.md` |
| `backend-developer` | Quarkus/Java implementation from OpenSpec tasks | `ai-specs/agents/backend-developer.md` |
| `frontend-developer` | React/PatternFly SPA implementation from OpenSpec tasks | `ai-specs/agents/frontend-developer.md` |
| `quality-reviewer` | Independent review after `/opsx-apply`, before archive — findings only, no fixes | `ai-specs/agents/quality-reviewer.md` |
| `product-strategy-analyst` | Ideation, use cases, personas, value props (pre-issue / explore) | `ai-specs/agents/product-strategy-analyst.md` |

After adding or renaming agents, run the `sync-agent-symlinks` skill (also syncs `.cursor/agents/` from `ai-specs/agents/` on Windows).

## Backlog

See `docs/sdd-backlog.md` for active changes. Recent focus: test coverage (#222 PRs), policy conversion epic (#149), export performance (#169).

## Replaces

- Engram-only SDD (legacy)
- `rhcl-ai` program AI docs (sunset planned; not in workspace)
