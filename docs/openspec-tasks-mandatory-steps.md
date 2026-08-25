# OpenSpec tasks.md — mandatory steps

When creating or updating `tasks.md` in an OpenSpec change:

## Structure

Group tasks by layer or phase (e.g. design spike, backend, frontend, tests, docs).

Each task must be:

- **One session sized** — completable without context switching
- **Verifiable** — clear done criteria
- **Ordered** — dependencies explicit (design before implementation)

## Required sections

1. **Prerequisites** — issue link, design.md reviewed, branch name
2. **Implementation** — checkbox list `- [ ]` / `- [x]`
3. **Verification** — exact test commands (`mvn test`, `npm test`, manual checks)
4. **Docs** — README / api-spec / backlog updates if applicable

## Rules

- Mark `- [x]` only when fully done, not partial
- If scope grows during `/opsx-apply`, update tasks.md **before** more code
- Backend conversion tasks must name test class (`ConversionServiceTest`)
- Do not add tasks that bypass #40 architecture without architect sign-off in design.md
- Reference GitHub issue in first task line

## Product repo paths

All implementation tasks target `../migration-toolkit-rhcl/` unless explicitly SDD-only.
