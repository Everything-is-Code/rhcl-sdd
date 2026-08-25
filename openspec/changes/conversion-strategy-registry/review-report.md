# AI Code Review

_Disclaimer: AI-generated review. Human maintainer review required._

**Change:** `conversion-strategy-registry` | **Issue:** [#40](https://github.com/Everything-is-Code/migration-toolkit-rhcl/issues/40)  
**Branch:** `feature/conversion-strategy-registry` | **Verify:** PASS (local Windows, 2026-08-25)

## Summary

The refactor achieves the #40 target shape: thin `ConversionService`, `ResourceGeneratorRegistry`, per-file generators, and HTTPRoute/AuthPolicy/Secret contributors with CDI discovery tests. Regression and concurrency coverage are strong. **Before opening the PR, stage and commit all untracked implementation files** — `git diff main` currently shows only ~6 files while `service/generator/`, `service/conversion/`, and most tests remain `??`.

## Major

- **Incomplete git scope for PR** — On `feature/conversion-strategy-registry`, `git status` shows the bulk of the implementation as untracked (`backend/src/main/java/.../service/generator/`, `service/conversion/`, `PolicyFinder.java`, discovery tests, etc.). Only modified files in the index are `ConversionService.java`, `ArchitectureTest`, `ConversionServiceTest`, `AGENTS.md`, `ConversionConstants`, and deletion of root `TECHNICAL_SPECIFICATIONS.md`. A PR from the current index would not build on a clean clone. **Action:** add and commit all product-repo artifacts before `/commit` or `gh pr create`.

## Moderate

- **`ResourceGeneratorRegistry.priorityOf` still uses proxy-blind annotation lookup** — `priorityOf` reads `@Priority` on `generator.getClass()` (line 73–75), not via `ContributorOrdering`. Under CDI client proxies, generator sort order may default to `0` instead of declared values (e.g. `ReadmeGenerator` @1900). Output map order is unlikely to affect YAML correctness today, but this is the same class of bug fixed for contributors. **Suggestion:** reuse `ContributorOrdering.priorityOf` for generators too.

- **Dead parameter `conversionService` in `ManualResourceGeneratorFactory.create`** — `create(ConversionService conversionService)` never uses the argument; `ResourceGeneratorRegistry.manual(this)` still passes it. Remove the parameter or use it (e.g. share `PolicyFinder` from the service) to avoid misleading wiring.

- **Orphan `AbstractPassthroughGenerator`** — No subclasses remain after the refactor; file is dead code. Delete or document if kept for a follow-up.

- **`ReadmeSupport` ↔ `ConversionService.ReadmeNotes` coupling** — `service.conversion.ReadmeSupport` depends on `ConversionService.ReadmeNotes` for the #170 collector. Acceptable for now, but moving `ReadmeNotes` to `conversion` (or a small `readme` package) would complete contributor/registry decoupling from the orchestrator.

## Minor

- **No unit test for `ContributorOrdering`** — Critical for Secret contributor order under CDI; a small parameterized test (real class vs. synthetic subclass with `@Priority`) would lock the proxy fix.

- **Manual test path vs. CDI path divergence** — `ManualResourceGeneratorFactory` wires explicit contributor lists; production uses `Instance<>`. Discovery tests cover CDI; ensure manual list order stays aligned when adding contributors (document in `Manual*ContributorFactory` or a single ordering table in `conversion-architecture.md`).

- **`docs/technical-specifications.md` stub untracked** — Root `TECHNICAL_SPECIFICATIONS.md` deleted (good: canonical doc is SDD store). Ensure the new stub in `docs/technical-specifications.md` is committed so product-repo links do not 404.

## Nit

- **Indentation** — `ResourceGeneratorRegistry` manual-constructor comment (`/** Manual wiring...`) is not aligned with surrounding code (line 28).

- **`ManualResourceGeneratorFactory`** — `conversionService` parameter name suggests dependency that does not exist; rename or drop when cleaning the signature.

## Positive notes

- `ConversionService` reduced to orchestration (~140 lines); no `generate*` or `find*Policy` left in orchestrator.
- ArchUnit: contributors must not depend on `ConversionService` (rule enforced without `allowEmptyShould`).
- Four discovery test classes with positive/negative cases.
- `ContributorOrdering` + `DefaultCredentialsSecretContributor` @900 fixes apiKey secret vs. placeholder regression under CDI.
- `ReadmeSupport.build` uses single `ReadmeNotes` collector (#170 aligned).
- `ConversionServiceConcurrencyTest` exercises parallel `convert()` under `@QuarkusTest`.
- `mvn verify` green locally (506 tests); spec scenarios traced in `verify-report.md`.

## Suggested follow-up (non-blocking, post-merge)

Consolidate into a hygiene issue (pattern #95): delete `AbstractPassthroughGenerator`, apply `ContributorOrdering` to registry, optional `ReadmeNotes` relocation, `ContributorOrderingTest`.

## Verdict

**Approve after addressing Major** (full commit scope). Moderate items are recommended in PR or immediate follow-up; not blocking if CI Linux is green on the complete branch.
