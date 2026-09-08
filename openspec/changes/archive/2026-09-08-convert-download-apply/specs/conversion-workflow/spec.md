## Purpose

Defines the SPA conversion workflow from validation through download, including direct cluster apply from the Download page without a ZIP import round-trip.

## ADDED Requirements

### Requirement: Download page offers cluster apply per converted service

The Download page SHALL display an **Apply to cluster** action for each converted service row that has YAML files, alongside the existing **Download ZIP** action.

#### Scenario: Apply action visible for converted package

- **WHEN** the user navigates to the Download page with at least one `conversionResults` entry containing YAML files
- **THEN** each such entry shows **Apply to cluster** next to **Download ZIP**

#### Scenario: Apply uses CONVERT source

- **WHEN** the user confirms apply for a converted service
- **THEN** the client calls `POST /api/apply` with `source: CONVERT`, the target namespace, the service `packageName`, and YAML files from workflow state for that service

### Requirement: Apply is blocked when cluster is not reachable

The **Apply to cluster** action SHALL be disabled when `appState.clusterVersions.capabilities.clusterReachable` is not `true`.

#### Scenario: Unreachable cluster disables apply

- **WHEN** `capabilities.clusterReachable` is `false` or cluster versions have not been loaded (`clusterVersions` is null)
- **THEN** **Apply to cluster** is disabled for all services on the Download page
- **AND** explanatory copy guides the user to authenticate the backend to OpenShift (`oc login` for local dev) or deploy in-cluster, consistent with cluster-detection UX

#### Scenario: Reachable cluster enables apply when other guardrails pass

- **WHEN** `capabilities.clusterReachable` is `true`
- **AND** other apply guardrails pass for a service
- **THEN** **Apply to cluster** is enabled for that service

### Requirement: Apply is blocked until validation passes without errors

The **Apply to cluster** action SHALL be disabled when validation has not been run for the current conversion snapshot, or when validation reports any `ERROR` item for that service.

#### Scenario: Validation not run

- **WHEN** the user reaches the Download page without a validation snapshot matching the current conversion fingerprint
- **THEN** **Apply to cluster** is disabled
- **AND** copy directs the user to run validation on the Validate page

#### Scenario: Validation ERROR blocks apply

- **WHEN** the stored validation snapshot for a service contains one or more items with status `ERROR`
- **THEN** **Apply to cluster** is disabled for that service
- **AND** copy indicates validation errors must be resolved first

#### Scenario: Validation WARNING does not block apply

- **WHEN** the stored validation snapshot for a service has no `ERROR` items but has `WARNING` items (e.g. unknown CRD)
- **THEN** **Apply to cluster** is enabled for that service when other guardrails pass

### Requirement: Apply is blocked when secrets contain REPLACE_ME

The **Apply to cluster** action SHALL be disabled when any YAML file value in the service's workflow state contains the credential placeholder `REPLACE_ME`.

#### Scenario: Placeholder present

- **WHEN** any YAML file content for the service includes `REPLACE_ME`
- **THEN** **Apply to cluster** is disabled
- **AND** copy directs the user to update secrets (e.g. via YAML Viewer) before applying

#### Scenario: Placeholder resolved

- **WHEN** no YAML file content for the service includes `REPLACE_ME`
- **AND** other guardrails pass
- **THEN** **Apply to cluster** is enabled

### Requirement: Validation snapshot is persisted in workflow state

The SPA SHALL store validation results in workflow state keyed to the current conversion fingerprint so the Download page can enforce apply guardrails without requiring the user to remain on the Validate page.

#### Scenario: Validate writes snapshot

- **WHEN** the user runs validation on the Validate page
- **THEN** the SPA stores per-service validation results together with the conversion fingerprint active at validate time

#### Scenario: Snapshot invalidated on conversion change

- **WHEN** `conversionResults` change such that the conversion fingerprint no longer matches the stored validation snapshot
- **THEN** the stored validation snapshot is cleared or treated as absent
- **AND** apply guardrails treat validation as not run

### Requirement: Apply uses edited YAML from workflow state

Cluster apply from the Download page SHALL send YAML from `appState.conversionResults` for the selected service, including edits saved from the YAML Viewer, not only the original conversion output.

#### Scenario: Edited YAML applied

- **WHEN** the user edits YAML on the YAML Viewer, saves to workflow state, validates, and applies from Download
- **THEN** `POST /api/apply` receives the edited file contents from workflow state

### Requirement: Apply requires user confirmation

The SPA SHALL show a confirmation dialog before invoking cluster apply from the Download page.

#### Scenario: Confirmation shows apply context

- **WHEN** the user clicks **Apply to cluster** for a service
- **THEN** a confirmation dialog displays the target namespace, the number of YAML files to apply, and a note that RHCL may create or update RBAC in the target namespace
- **AND** apply proceeds only after the user confirms

#### Scenario: Cancel does not apply

- **WHEN** the user dismisses or cancels the confirmation dialog
- **THEN** no apply request is sent

### Requirement: Apply results are shown inline on Download

After cluster apply from the Download page, the SPA SHALL display per-file success or failure results inline on the same page.

#### Scenario: Successful apply shows results

- **WHEN** apply completes with one or more successful files
- **THEN** the Download page shows a per-file result table (success/partial/failure) for that apply attempt

#### Scenario: Partial failure shows per-file errors

- **WHEN** apply completes with mixed success and failure
- **THEN** each failed file shows its error message in the result table
- **AND** successful files are shown as succeeded

### Requirement: Successful convert-path apply records CONVERT history

A successful cluster apply from the Download page SHALL create a conversion history entry with `source: CONVERT` using existing backend apply/history behavior.

#### Scenario: History entry after apply

- **WHEN** apply from Download completes with at least one successful file
- **THEN** a history entry is visible on the History page with `source: CONVERT` for that apply run

### Requirement: Target namespace is taken from workflow connection state

Cluster apply from the Download page SHALL use `appState.namespace` (set during Connection setup) as the apply target namespace, with copy explaining that the backend forces this namespace on all applied resources.

#### Scenario: Namespace displayed before apply

- **WHEN** the user views the Download page with apply enabled
- **THEN** the target namespace from workflow state is visible near the apply action or confirmation dialog
