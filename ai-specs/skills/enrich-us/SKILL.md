---
name: enrich-us
description: Analyze and enhance a GitHub issue into implementation-ready detail (acceptance criteria, files, tests, edge cases). Use before /opsx-propose.
author: RHCL / adapted from LIDR SpecBoot
version: 2.0.0
---
# enrich-us — GitHub Issues

Enrich a vague GitHub issue before OpenSpec planning.

## Input

Issue reference: `$ARGUMENTS` — issue number (`#149`), URL, or keywords ("the cors policy issue").

## Steps

1. **Fetch issue** with `gh issue view <number> --repo Everything-is-Code/migration-toolkit-rhcl` (or infer repo from workspace).
2. **Read context** from `docs/` in this store (base-standards, conversion-architecture, sdd-backlog).
3. **Assess completeness** against product best practices:
   - User-visible behavior and acceptance criteria
   - API endpoints / DTOs affected
   - Files to touch (`ConversionService.java`, controllers, frontend pages, tests)
   - Test plan (JUnit / frontend)
   - Non-functional: security (secrets), performance (#169), i18n if UI strings
   - Dependencies on #40 / #149 / #170 when relevant
4. **If incomplete**, produce an enhanced issue body in markdown with sections:
   - `## [original]` — paste or summarize current body
   - `## [enhanced]` — improved story with AC, technical notes, file list, test plan, out of scope
5. **Post comment** (only if user confirms): `gh issue comment <number> --body-file ...` with enhanced content.
6. **Do not** close or edit the issue title without explicit user approval.

## Output

Return the enhanced markdown to the user and suggest the OpenSpec change name for `/opsx-propose`.

## Notes

- No Jira MCP — GitHub Issues only.
- For epic #149 children, cross-check `docs/sdd-backlog.md` for naming convention.
