# test-coverage-dto-model-entity

**Status:** Superseded — not implemented. Scope merged into [#222](https://github.com/Everything-is-Code/migration-toolkit-rhcl/issues/222) / OpenSpec `test-coverage-service-data-layers`.

## Why (original)

Optional PR-6 of #210 for `dto/`, `model/`, `entity/` layers.

## Baseline on `main` after #210 merges (Aug 2026)

| Package | Line coverage | Action |
|---------|---------------|--------|
| `dto` | **100%** | No work needed |
| `entity` | **~65%** | Carry to #222 if dedicated Panache tests add value |
| `model` | **~50%** | Extend `ModelTest` under #222 |

## Archive reason

#210 tiers A–C merged. Remaining global coverage gap is `service/` root (~19%), not dto/model/entity. Combined follow-up tracked in #222.
