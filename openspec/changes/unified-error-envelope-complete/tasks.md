## 1. Branch setup and NotFoundException

- [x] 1.1 Create branch `feature/196-unified-error-envelope-complete` from `main` (or from `feature/171-unified-error-envelope` if #195 not merged); verify `exception/` package exists with `ApiException`, `ValidationException`, `ThreeScaleClientException`, `ClusterApplyException`, `ImportParseException`, `ErrorSanitizer`, `ApiExceptionMapperResource`
- [x] 1.2 Create `NotFoundException extends ApiException` with status 404 and caller-specified code; verify by writing a unit test in `ApiExceptionMapperTest` that throws `NotFoundException("HISTORY_NOT_FOUND", "...")` and asserts HTTP 404 with `error.code = "HISTORY_NOT_FOUND"` in the envelope

## 2. Backend — migrate remaining controllers

- [x] 2.1 Migrate `ConnectionController`: on `testConnection` failure throw `ThreeScaleClientException("CONNECTION_TEST_FAILED", "Failed to connect to 3scale...", 502)`; keep success path `200 { success: true, message }`; verify `ConnectionControllerTest` asserts 502 envelope with `CONNECTION_TEST_FAILED` code and 200 success unchanged
- [x] 2.2 Migrate `ExportController`: replace three plain-string 400 responses with `throw new ValidationException("url query parameter and Authorization Bearer token are required")`; remove try/catch for service-layer calls (let mapper handle); verify `ExportControllerTest` asserts 400 envelope (not plain string) for missing params
- [x] 2.3 Migrate `HistoryController`: `getHistoryById` 404 → `throw new NotFoundException("HISTORY_NOT_FOUND", ...)`; download 404 → same; download 500 catch → `throw new ClusterApplyException("HISTORY_DOWNLOAD_FAILED", e.getMessage())`; `deleteByIds` empty list → `throw new ValidationException("No IDs provided")`; verify `HistoryControllerTest` asserts envelope shape on 404, 500, and 400 paths
- [x] 2.4 Migrate `ClusterController.getDomain()`: route not found → `throw new NotFoundException("CLUSTER_ROUTE_NOT_FOUND", ...)`; host not assigned → `throw new NotFoundException("CLUSTER_ROUTE_HOST_PENDING", ...)`; domain extract failed → `throw new NotFoundException("CLUSTER_DOMAIN_EXTRACT_FAILED", ...)`; catch block 500 → remove catch, let mapper produce `INTERNAL_ERROR`; verify `ClusterControllerTest` asserts envelope for each 404 case and 500 fallback
- [x] 2.5 Migrate `GatewayInfoController`: missing params → `throw new ValidationException(...)`; gateway null → `throw new NotFoundException("GATEWAY_NOT_FOUND", ...)`; catch 500 → remove catch, let mapper handle; verify `GatewayInfoControllerTest` asserts 400/404/500 envelopes
- [x] 2.6 Migrate `PackageController`: `downloadZip` missing yamlFiles → `throw new ValidationException("yamlFiles are required")`; `downloadFromHistory` 404 → `throw new NotFoundException("HISTORY_NOT_FOUND", ...)`; verify `PackageControllerTest` asserts 400 envelope (not plain string) and 404 envelope
- [x] 2.7 Migrate `SettingsController`: GET 404 → `throw new NotFoundException("SETTINGS_NOT_FOUND", ...)`; PUT missing value → `throw new ValidationException("value is required")`; verify new or extended `SettingsControllerTest` asserts 404 and 400 envelopes
- [x] 2.8 Verify `ConversionController` dnsHostname validation (from #195): if still `Map.of("error", ...)`, replace with `throw new ValidationException("dnsHostname is required when includeDnsPolicy is true")`; verify `ConversionControllerTest` asserts envelope on that path

## 3. Backend verification

- [x] 3.1 Run `mvn verify` in `backend/` and confirm all tests pass (including `ArchitectureTest` with `exception` layer)
- [x] 3.2 Grep for `Map.of("error"` and `.entity("` (plain string) in `controller/` package; confirm zero matches on error paths (success-path `Map.of` like pagination or `ApplyResult` are allowed)

## 4. Frontend — extend error code map and i18n keys

- [x] 4.1 Add new codes to `ERROR_CODE_I18N` in `apiError.ts`: `CONNECTION_TEST_FAILED`, `HISTORY_NOT_FOUND`, `HISTORY_DOWNLOAD_FAILED`, `CLUSTER_ROUTE_NOT_FOUND`, `CLUSTER_ROUTE_HOST_PENDING`, `CLUSTER_DOMAIN_EXTRACT_FAILED`, `GATEWAY_NOT_FOUND`, `SETTINGS_NOT_FOUND`; verify by inspecting the map has 14 entries total
- [x] 4.2 Add i18n keys for all new codes in `locales/en.json` and `locales/ja.json`; verify both files have matching keys under `error.*` namespace
- [x] 4.3 Add `apiErrorI18nMessage` tests in `apiError.test.ts`: envelope with known code returns `t(key)`, envelope with unknown code returns backend `message`, legacy string error returns raw string, `success: false` pattern returns message; verify `npm test -- --testPathPattern apiError` passes

## 5. Frontend — migrate catch blocks to i18n

- [x] 5.1 Migrate `ConnectionForm.tsx`: replace `apiErrorMessage(e, '')` and `apiErrorMessage(e, t('connection.errorDefault'))` with `apiErrorI18nMessage(e, t, t('connection.errorDefault'))`; verify `npm run typecheck` passes
- [x] 5.2 Migrate `ConnectionPage.tsx`: replace `apiErrorMessage(e, 'Failed to load cluster versions')` with `apiErrorI18nMessage(e, t, t('connection.versionsError'))`; verify typecheck
- [x] 5.3 Migrate `ClusterVersionsPanel.tsx`: replace `apiErrorMessage(e, t('connection.profileSaveError'))` with `apiErrorI18nMessage(e, t, t('connection.profileSaveError'))`; verify typecheck
- [x] 5.4 Migrate `APISelectionPage.tsx`: replace `apiErrorMessage(e, 'Failed to load services')` with `apiErrorI18nMessage(e, t, t('apiSelection.errorFetchFallback'))`; add i18n key; verify typecheck
- [x] 5.5 Migrate `ConversionPage.tsx`: replace `apiErrorMessage(e, 'Conversion failed')` in catch with `apiErrorI18nMessage(e, t, t('conversion.errorConvertFallback'))`; verify typecheck
- [x] 5.6 Migrate `ImportPage.tsx`: replace 3 `apiErrorMessage` calls (upload, download, apply) with `apiErrorI18nMessage(e, t, ...)`; verify typecheck
- [x] 5.7 Migrate `DownloadPage.tsx`: replace `apiErrorMessage(e, 'Download failed')` with `apiErrorI18nMessage(e, t, t('download.errorDownloadFallback'))`; verify typecheck
- [x] 5.8 Migrate `HistoryPage.tsx`: replace 3 `apiErrorMessage` calls (list, download, delete) with `apiErrorI18nMessage(e, t, ...)`; verify typecheck
- [x] 5.9 Migrate `ValidationPage.tsx`: replace `apiErrorMessage(e, 'Validation failed')` with `apiErrorI18nMessage(e, t, t('validation.errorFallback'))`; verify typecheck
- [x] 5.10 Migrate `SupportedPoliciesPage.tsx`: replace bare `catch { setError(t('supportedPolicies.saveError')) }` with `catch (e: unknown) { setError(apiErrorI18nMessage(e, t, t('supportedPolicies.saveError'))) }`; verify typecheck
- [x] 5.11 Migrate `useGatewayUrl.ts`: in catch blocks, check `isAxiosError(e) && e.response?.status` — on 400/404 stop polling and set `apiErrorI18nMessage(e, t, ...)`; on 500/network continue retry; verify existing gateway tests pass

## 6. Frontend verification

- [x] 6.1 Run `npm run typecheck` in `frontend/`; confirm zero errors
- [x] 6.2 Run `npm test` in `frontend/`; confirm all tests pass (17 files, 67 tests)
- [x] 6.3 Grep for `apiErrorMessage(` in `frontend/src/` excluding `apiError.ts` and `apiError.test.ts`; confirm zero production usage (only the internal fallback inside `apiError.ts` remains)

## 7. Documentation and SDD

- [x] 7.1 Update `docs/technical-specifications.md` §5.8 with final error handling description (all controllers, full code registry, `NotFoundException`); verify the section reflects the completed migration
- [x] 7.2 Update `docs/sdd-backlog.md` — add #196 row; verify table is consistent
