# Skills inventory — RHCL SDD vs [LIDR SpecBoot](https://github.com/LIDR-academy/lidr-specboot/tree/main/ai-specs/skills)

Canonical source: `ai-specs/skills/`. Cursor commands in `.cursor/commands/`.

## From SpecBoot (in this repo)

| Skill | Status | RHCL notes |
|-------|--------|------------|
| `adversarial-review` | ✅ adapted | Red-team before archive; GitHub PR scope |
| `code-auditing` | ✅ | Broad 6-phase audit |
| `commit` | ✅ | PR via `gh`; product repo target |
| `enrich-us` | ✅ adapted | GitHub Issues, not Jira |
| `explain` | ✅ | |
| `meta-prompt` | ✅ | |
| `openspec-sync-specs` | ✅ | `--store rhcl-sdd`; also `/opsx-sync` command |
| `show-spec-working` | ✅ adapted | Quarkus + Vite local demo |
| `sync-agent-symlinks` | ✅ | Windows copy fallback |
| `update-docs` | ✅ | |
| `using-git-worktrees` | ✅ | |
| `writing-skills` | ✅ | |

## RHCL additions

| Skill | Role |
|-------|------|
| `verify-change` | `/verify` — tests + `openspec validate` + verify-report |
| `verify-fe` | `/verify-fe` — frontend typecheck/vitest + live Playwright YAML E2E (prompts for 3scale URL + token) |
| `code-review` | `/code-review` — RHCL Major/Moderate/Minor/Nit PR comment |

## OpenSpec CLI skills (`.cursor/skills/openspec-*`)

Installed by `openspec init` — mirror workflow commands:

| Skill / command | Maps to |
|-----------------|---------|
| `openspec-propose` / `/opsx-propose` | Create change artifacts |
| `openspec-apply-change` / `/opsx-apply` | Implement tasks |
| `openspec-archive-change` / `/opsx-archive` | Archive change |
| `openspec-explore` / `/opsx-explore` | Explore |
| `openspec-update-change` / `/opsx-update` | Update artifacts |
| `openspec-sync-specs` / `/opsx-sync` | Sync specs (see skill above) |

## Recommended workflow

```
enrich-us #N
  → /opsx-propose <change>
  → /opsx-apply <change>
  → /verify-fe <change>       # if frontend/ or wizard/YAML — asks URL + token in chat
  → /verify <change>          # mvn test + npm test + openspec validate
  → /code-review <change>
  → adversarial-review (skill or fresh session)
  → /opsx-archive <change>
  → /commit #N
```

**`/verify-fe`:** Playwright wizard + YAML tab checks (`yaml-expectations.ts`). User pastes **Admin URL** and **PAT** in chat; agent uses env vars only for that run — never commits credentials.

Optional: `show-spec-working` after apply; `code-auditing` for release sweeps.

## After skill changes

Run `sync-agent-symlinks` skill to align `.cursor/skills` and `.claude/skills`.
