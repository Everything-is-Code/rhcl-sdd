# Conversion Architecture — Target Design

`ConversionService.java` (~1300 lines) converts 3scale policies to Kuadrant / Gateway API / Istio YAML. The agreed target architecture (issue **#40**, not fully implemented) has two levels.

## Level 1 — Strategy + Registry (per output file)

One `ResourceGenerator` per Kubernetes resource file:

| Output file | Resource kinds |
|-------------|----------------|
| `gateway.yaml` | Gateway |
| `httproute.yaml` | HTTPRoute |
| `policy.yaml` | AuthPolicy, RateLimitPolicy, etc. |
| `secret.yaml` | Secret |
| `destinationrule.yaml` | DestinationRule |
| `serviceentry.yaml` | ServiceEntry |
| `configmap.yaml` | ConfigMap |

Lookup via a **registry** instead of hardcoded branches in `convert()`.

## Level 2 — Collector / Contributor (within complex files)

Some files aggregate fragments from **multiple** unrelated 3scale policies:

- **HTTPRoute**: `header_modification`, `cors`, `url_rewriting`, mapping rules, retry, multi-backend
- **AuthPolicy** (`policy.yaml`): several auth-related policies
- **Secret**: credential-related policies

Model each policy's contribution as a `Contributor` against a shared builder — not inline `if` chains in the generator.

## Before Adding a New Policy Conversion

Check:

1. **#149** — epic for 19 recognized-but-unconverted policies (each has issue + spec + target mapping).
2. **#40** — implement in target shape where practical, even before full refactor lands.
3. **#170** — `generateReadme(...)` uses a growing positional note list; do not add another positional parameter.

## Adapter Integration

YAML generation delegates to **from-3scale-to-connectivity-link** (external adapter). Phase 2 SDD will cover that repo; this store documents toolkit integration points only.

## Testing

- `ConversionServiceTest` — YAML string assertions are whitespace-sensitive.
- **Windows CRLF**: local `mvn test` may fail on YAML assertions with `core.autocrlf=true` while CI (Linux) is green. Trust CI before treating as regression.
