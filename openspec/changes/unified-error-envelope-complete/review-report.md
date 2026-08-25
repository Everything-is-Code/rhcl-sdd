## AI Code Review

_Disclaimer: AI-generated review. Human maintainer review required._

**Change:** unified-error-envelope-complete | **Issue:** [#196](https://github.com/Everything-is-Code/migration-toolkit-rhcl/issues/196)

**Branch:** `feature/196-unified-error-envelope-complete` (1 committed commit from #171 + uncommitted #196 work)

**Test results (this review):**
- `mvn test -Dtest=ConnectionControllerTest,GatewayInfoControllerTest,ApiExceptionMapperTest,HistoryControllerTest,ClusterControllerTest,PackageControllerTest,ExportControllerTest` — **PASS**
- `npm run typecheck` — **PASS**
- `npm test` — **PASS** (17 files, 67 tests)
- Full `mvn test` — **3 errors** (`ApplyControllerTest` / `ExportCorsOriginsOverrideTest` Quarkus startup, `PlaywrightE2EIT` NPE); not reproduced in scoped controller suite — treat as environmental unless CI confirms

---

### Blocker

_(none for runtime behavior once `NotFoundException.java` is staged — see Major M-1)_

---

### Major

#### M-1. `NotFoundException.java` is untracked — build breaks if omitted from commit

**Evidence:** `git status` shows `?? backend/.../exception/NotFoundException.java` while seven controllers import it (`HistoryController`, `ClusterController`, `GatewayInfoController`, `PackageController`, `SettingsController`).

**Risk:** A commit/push that includes controller changes without this file fails compilation on a clean checkout.

**Fix:** Stage `NotFoundException.java` in the same commit as controller migrations.

---

#### M-2. Controller envelope tests incomplete vs. `tasks.md` claims

`tasks.md` marks envelope assertions done for History, Cluster, Package, Export, and Settings controllers. Evidence shows gaps:

| Controller | Envelope `error.code` asserted? | Notes |
|---|---|---|
| `ConnectionController` | Yes (`CONNECTION_TEST_FAILED`) | Updated in uncommitted diff |
| `GatewayInfoController` | Yes (`VALIDATION_FAILED`, `GATEWAY_NOT_FOUND`, `INTERNAL_ERROR`) | Updated |
| `HistoryController` | **No** | `getHistoryById_notFound_returns404`, `downloadYaml_notFound_returns404`, `deleteByIds_emptyList_returns400` only check status (or `error` presence) |
| `ClusterController` | **No** | No tests for `CLUSTER_ROUTE_NOT_FOUND`, `CLUSTER_ROUTE_HOST_PENDING`, `CLUSTER_DOMAIN_EXTRACT_FAILED`, or 500 `INTERNAL_ERROR` |
| `PackageController` | **No** | `downloadZip_missingYamlFiles_returns400`, `downloadFromHistory_notFound_returns404` only check status |
| `ExportController` | **No** | `getServices_missingParams_returns400` only checks status |
| `SettingsController` | **Missing test class** | No `SettingsControllerTest` at all |

`ApiExceptionMapperTest` covers `NotFoundException` mapping via `ExceptionTestResource`, but that does not substitute for controller-path integration tests (routing, transaction boundaries, ZIP content-type on download errors, etc.).

**Fix:** Add REST-assured assertions (`.body("error.code", equalTo("..."))`) to the tests above; create `SettingsControllerTest` for 404 `SETTINGS_NOT_FOUND` and 400 `VALIDATION_FAILED`; add `HISTORY_DOWNLOAD_FAILED` test (mock/corrupt `exportedYaml` to force ZIP failure).

---

#### M-3. All #196 product changes are still uncommitted

**Evidence:** `git log main..HEAD` shows only `9c9b1e3 feat: unified error envelope with ApiException hierarchy (#171)`. The seven controller migrations, frontend i18n adoption, locale keys, and `useGatewayUrl` fix exist only in the working tree.

**Risk:** Review/CI on the pushed branch does not include #196 scope until committed and pushed.

**Fix:** Commit uncommitted changes before opening PR; verify CI runs against the full diff.

---

### Moderate

#### Mo-1. Duplicate root-level `"error"` namespace in locale files

**Files:** `frontend/src/locales/en.json`, `frontend/src/locales/ja.json`

Two sibling `"error"` objects exist (first at ~line 382 with 6 keys from #171, second at ~line 410 with 14 keys from #196). JSON parsers keep the **last** key, so runtime i18n works today, but the first block is dead code and confuses maintainers. Text also diverges slightly between duplicates (e.g. `threescaleClient` wording).

**Fix:** Merge into a single `"error"` block with all 14 keys; remove the duplicate.

---

#### Mo-2. `useGatewayUrl` terminal-error behavior has no unit tests

**File:** `frontend/src/components/import/useGatewayUrl.ts`

Design D8 requires 400/404 → stop polling + `apiErrorI18nMessage()`; 500/network → retry. Implementation adds `isNonRetryable()` and matches the design, but there is no `useGatewayUrl.test.ts`. Regression risk is real because polling timing makes this hard to catch manually.

**Fix:** Add Vitest tests mocking `gatewayApi.getInfo` rejections with statuses 400, 404, 500, and network error; assert `setError` called vs. retry loop.

---

#### Mo-3. `HISTORY_DOWNLOAD_FAILED` forwards raw exception message

**File:** `HistoryController.java:144`

```java
throw new ClusterApplyException("HISTORY_DOWNLOAD_FAILED", e.getMessage());
```

`ErrorSanitizer` redacts tokens in the mapper, but Jackson/IO exception messages can still expose internal paths or parse details. Spec requires a "sanitized message."

**Fix:** Use a fixed user-facing message (e.g. `"Failed to create download archive"`) and put technical detail in `details` or logs only.

---

#### Mo-4. `ClusterController` / `GatewayInfoController` still wrap unexpected errors in `RuntimeException`

**Files:** `ClusterController.java:140-142`, `GatewayInfoController.java:83-85`

Design task 2.4/2.5 said remove catch blocks and let the mapper produce `INTERNAL_ERROR`. Controllers rethrow `ApiException` but wrap other exceptions in `new RuntimeException(...)`. The mapper still returns the correct envelope, but this adds an extra wrapper layer and duplicates the message in logs.

**Fix:** Remove the outer catch or rethrow without wrapping so the fallback mapper handles the original exception directly.

---

#### Mo-5. Proposal/spec drift on `CONVERSION_DNS_REQUIRED`

**Proposal** lists `CONVERSION_DNS_REQUIRED` as a new code; **design** allows `VALIDATION_FAILED` or that code; **implementation** uses `ValidationException` (`VALIDATION_FAILED`) for missing `dnsHostname` (`ConversionController.java:77`). Spec registry lists 14 codes without `CONVERSION_DNS_REQUIRED`.

**Fix:** Either add the dedicated code + i18n key (if UX differentiation is wanted) or remove `CONVERSION_DNS_REQUIRED` from `proposal.md` to match implementation.

---

### Minor

#### Mi-1. Redundant double-localization pattern in page catch blocks

Pages such as `ConversionPage`, `APISelectionPage`, `ImportPage` wrap `apiErrorI18nMessage()` inside another `t('...', { message })` template. When the envelope code maps to a full sentence (e.g. `CONNECTION_TEST_FAILED`), users see `"Conversion failed: Failed to connect to 3scale..."` — grammatically fine but duplicates context the i18n key already provides.

**Fix:** Optional cleanup per design D5 — use bare `apiErrorI18nMessage(e, t, t('...fallback'))` where the code-specific message is sufficient.

---

#### Mi-2. `ERROR_CODE_I18N` keys not validated against locale files

`locales.smoke.test.ts` checks key parity between `en.json` and `ja.json` but does not assert that every `ERROR_CODE_I18N` value resolves to an existing key. A typo in the map would silently fall back to backend English `message`.

**Fix:** Add a small test iterating `ERROR_CODE_I18N` values against flattened locale keys.

---

#### Mi-3. `frontend-standards.mdc` still prescribes `apiErrorMessage()` in catch blocks

Product rule file was not updated (#196 scope may include docs only in SDD store). New contributors may follow stale guidance.

**Fix:** Track in docs follow-up or update `frontend-standards.mdc` error-handling section.

---

### Nit

#### N-1. `HistoryControllerTest.deleteByIds_emptyList_returns400` asserts `.body("error", notNullValue())` instead of `error.code`

Weak assertion — passes for any envelope shape. Tighten to `VALIDATION_FAILED` when addressing M-2.

---

#### N-2. `technical-specifications.md` §7 table row still says "#196 — not yet tracked" while `sdd-backlog.md` lists #196 as in progress

Cosmetic SDD inconsistency; update when archiving.

---

### Summary

The uncommitted implementation correctly completes the controller migration pattern established in #171: typed exceptions, unified envelope via existing mapper, 14 frontend `ERROR_CODE_I18N` entries, and full production adoption of `apiErrorI18nMessage()`. `grep` confirms zero `Map.of("error"` in controllers and zero production `apiErrorMessage()` calls outside `apiError.ts`.

Top risks before merge: **stage `NotFoundException.java`**, **commit and push the working tree**, and **close the controller test gaps** (especially `SettingsControllerTest`, Cluster 404 variants, and `HISTORY_DOWNLOAD_FAILED`). Merge is not recommended until M-1–M-3 are resolved; Mo-1 (duplicate locale `error` block) should be fixed in the same PR to avoid shipping invalid JSON structure.
