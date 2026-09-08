## Why

[#313](https://github.com/Everything-is-Code/migration-toolkit-rhcl/issues/313): after converting a 3scale API to Kuadrant/Gateway YAML, users must download a ZIP and re-upload it on the Import page to apply resources to OpenShift—even though the backend already supports `POST /api/apply` with `source: CONVERT`. That ZIP round-trip adds friction for in-cluster deployments where the RHCL backend already has cluster access.

## What Changes

- **Frontend — Download page apply:** add **Apply to cluster** per converted service row (alongside **Download ZIP**), calling existing `applyApi.apply(namespace, yamlFiles, 'CONVERT', packageName)`.
- **Frontend — guardrails:** disable apply when `clusterReachable === false`, validation has not run or has ERROR items for that service, YAML contains `REPLACE_ME`, or there are no YAML files; show actionable helper copy (reuse cluster-detection UX patterns).
- **Frontend — validation snapshot:** persist validation results in workflow state (keyed by conversion fingerprint) so Download can enforce guardrails without re-running validate on every visit.
- **Frontend — confirmation:** show a confirmation dialog before apply (namespace, file count, RBAC note).
- **Frontend — results:** display per-file apply results inline via reused `ImportResultTable` (or equivalent); successful apply creates history with `source: CONVERT` (existing backend behavior).
- **Frontend — namespace:** target namespace from `appState.namespace` (Connection setup), with clear copy that ApplyController forces the request namespace on all resources.
- **Shared apply prep:** extract or reuse Import apply preparation (apiVersion normalization, external-backend HTTPRoute port fix) so convert-path apply matches Import behavior.
- **i18n:** new strings in `en.json` and `ja.json`.
- **Tests:** unit tests for Download page apply flow (mock `applyApi`; disabled states; happy path; confirm modal).
- **Docs:** update user guide — conversion workflow may apply directly without ZIP import.

## Capabilities

### New Capabilities

- `conversion-workflow`: SPA conversion workflow from validate through download, including cluster apply from the Download page, validation snapshot persistence, apply guardrails, confirmation UX, and inline apply results.

### Modified Capabilities

_(none — consumes existing `cluster-detection` `clusterReachable` contract without changing it)_

## Impact

- **Frontend:** `DownloadPage.tsx` (primary), `ValidationPage.tsx`, `AppStateContext.tsx` / `conversionWorkflowState.ts`, possible shared apply helper extracted from `ImportPage.tsx`, `ImportResultTable`, new confirm modal component, `locales/en.json`, `locales/ja.json`.
- **Backend:** no API or apply semantics changes (`ApplyController`, history with `source: CONVERT` already exist).
- **Docs:** `migration-toolkit-rhcl/documentation/user-guide.md`.
- **Out of scope:** new backend apply endpoint, bulk apply across services, post-apply curl smoke test (`TestInfoPanel`), multi-cluster apply, auto-apply after validation.
