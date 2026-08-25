---
name: adversarial-review
description: Independent red-team review before archiving an OpenSpec change. Use after /verify, before /opsx-archive. Different session/agent than implementer preferred.
author: LIDR.co / RHCL adapted
version: 1.1.0
---
# adversarial-review Skill

Independent adversarial reviewer: assume gaps, flaws, or unsafe behavior until argued against with evidence.

**Window:** after `/opsx-apply` and `/verify`, **before** `/opsx-archive`.

**Preferred agent:** `quality-reviewer` — this skill should be run with the `quality-reviewer` agent, not the implementer. The reviewer's fresh perspective catches what the implementer normalizes.

## Inputs

- GitHub issue `#N`, change name, or PR URL (`Everything-is-Code/migration-toolkit-rhcl#N`)
- Infer from active change: `openspec list --store rhcl-sdd`

## Mindset

- Try to **break** the system, not only happy paths
- Hunt wrong assumptions: authz, idempotency, cache keys, token handling, YAML edge cases
- Cross-boundary risks: API + UI, ConversionService + tests, apply + history
- Trace spec vs code mismatches as first-class findings

## Workflow

### 1 — Spec side

Read `openspec/changes/<name>/`: proposal, design, spec(s), `tasks.md`. List acceptance criteria and non-goals.

### 2 — Implementation side

- PR or `git diff` in `../migration-toolkit-rhcl/` against `main`
- Map files to spec sections and tasks

### 3 — Adversarial pass

For each criterion: how could it still fail? Check negative paths, #40 alignment, #170, secrets in YAML/logs, CRLF test false positives.

### 4 — Severity

| Level | Meaning |
|-------|---------|
| **Blocker** | Wrong behavior, security, spec violation — stop archive |
| **Major** | Likely bug or significant gap |
| **Minor** | Clarity, maintainability |
| **Question** | Needs human confirmation |

Fix in: code / tests / OpenSpec artifacts / docs

### 5 — Verdict

- **PASS (adversarial)** — no blockers/majors
- **PASS WITH GAPS** — minors only, tracked
- **FAIL** — blocker or major remains

Save: `openspec/changes/<name>/adversarial-review.md`

## Output format

```markdown
## Adversarial review

**Scope:** <change / #issue / PR>
**Sources:** <spec paths + diff>

### Spec alignment
- ...

### Findings
| Severity | Area | Finding | Evidence | Fix (code/spec/tests) |

### Verdict
PASS | PASS WITH GAPS | FAIL

### Before archive
- ...
```

## Guardrails

- Do not rubber-stamp; read OpenSpec artifacts when they exist
- Do not praise to "balance" unless it mitigates a documented risk
- RHCL review style: English, no auto-approve unless user asks
