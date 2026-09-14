# Design: HTTPRoute CORS + external URLRewrite fix (#332)

## Root cause (hypothesis)

`HttpRouteBuilder.injectCorsIntoRule` when `filters:` already exists inserts raw CORS YAML **before** `matches:` instead of appending to the `filters` list. Combined with Fabric8 serialization of URLRewrite for external backends, this yields **two sibling `filters:` keys** in the emitted YAML.

Existing tests (`HttpRouteBuilderTest.injectCorsFilters_appendsCorsBeforeMatchesWhenFiltersExist`) use simplified YAML without quoted types and do not cover full `ConversionService.convert` with external backend + `corsNative=true`.

## Fix approach

| Area | Change |
|------|--------|
| `HttpRouteBuilder.injectCorsIntoRule` | When `filters:` exists, insert CORS filter **items** immediately after the last filter entry (or after `filters:` line), not as a new `filters:` block |
| `ConversionServiceTest` | New test: external `https://httpbin.org:443` + cors policy + `opts.corsNative=true` → assert exactly one `filters:` per rule, contains URLRewrite + CORS |
| `ValidationService` | Add `validateDuplicateYamlKeys` — ERROR if duplicate keys at same mapping level (per document); optional HTTPRoute rule check |
| `serviceentry.yaml` | Investigate double `---` in multi-doc generator (minor) |

## Validation enhancement

Current `ValidationService.validate` stops at parse + CRD + namespace + loose references. Extend with:

```java
// duplicate key scan on raw content per document (stack-based indent)
// OR SnakeYAML SafeConstructor with duplicate key detection if available
```

Mark package `valid: false` when any ERROR item exists (unchanged rule).

## Threat matrix

N/A — conversion/validation only; no new cluster privileges.

## Files (product repo)

| File | Action |
|------|--------|
| `HttpRouteBuilder.java` | Fix injectCorsIntoRule |
| `HttpRouteBuilderTest.java` | Strengthen assertions (count `filters:` keys) |
| `ConversionServiceTest.java` | Workshop repro integration test |
| `ValidationService.java` | Duplicate key / HTTPRoute structure checks |
| `ValidationServiceTest.java` | Fixture from `rhcl-seed-cors/httproute.yaml` → valid=false |
