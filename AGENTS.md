# AGENTS.md — Project context for AI assistants

## Project objective

Build and maintain **resource-state-metrics**: a Kubernetes controller that watches `ResourceMetricsMonitor` custom resources, evaluates user-defined metric expressions (CEL, Starlark, or raw field paths) against arbitrary Kubernetes resources, and exposes the results as Prometheus metrics.

- **Success looks like**: Correct, performant metric generation for any Kubernetes resource type, stable CRD API, clear extension points for new resolvers, and passing upstream CI.
- **Non-goals**: This is not a general monitoring platform—it only exposes metrics defined via `ResourceMetricsMonitor` CRs. It does not replace kube-state-metrics, Prometheus, or custom controllers.

## How we work (agentic first)

- **Human role**: intent, architecture trade-offs, security-sensitive decisions, final review and merge.
- **Agent role**: implementation drafts, refactors, tests, docs—always reviewed by a human familiar with the area.
- **Issues**: prefer **Goal + Acceptance criteria**; link files and constraints. Commits and PRs carry the ticket key (`MON-123`) or `NO-JIRA` prefix.
- **Workflow**: propose a short plan for non-trivial work before implementing. Prefer small, reviewable steps.
- **Verification**: run `make verify` after edits (or the relevant subset: `make lint`, `make test`).

## Architecture (high level)

```
API Server → Informer/Lister → Controller → Configurer → Store → Resolver → Metric/Family → Writer → HTTP /metrics
```

- **Entry point**: `main.go`.
- **`internal/`**: Controller reconcile loop, configuration parsing, metric store, Prometheus writer, cardinality enforcement, HTTP server.
- **`pkg/resolver/`**: Expression resolvers (CEL, Starlark, field-path) behind a shared `Resolver` interface.
- **`pkg/apis/`**: CRD type definitions (`ResourceMetricsMonitor` v1alpha1) — the API contract.
- **`pkg/generated/`**: Auto-generated clientset, informers, listers — **never edit by hand**.
- **Integrations**: Kubernetes API server (watch/list), Prometheus (metrics exposition).

## Repository layout

| Path | Purpose |
|------|---------|
| `main.go` | Binary entry point |
| `internal/` | Private application code (controller, server, store) |
| `pkg/apis/` | CRD type definitions |
| `pkg/resolver/` | Expression resolvers (CEL, Starlark, Unstructured) |
| `pkg/generated/` | Auto-generated K8s client code — **never edit by hand** |
| `pkg/options/` | CLI flag definitions |
| `pkg/metricutil/` | Prometheus helpers |
| `hack/` | Shell scripts (codegen, license headers, conventional-commit check) |
| `tests/` | Integration tests, golden files, test manifests |
| `tests/golden/` | Per-resolver golden test files (YAML with `.in` and `.out` sections) |
| `manifests/` | Kubernetes deployment manifests (CRD, RBAC, Deployment, Service) |
| `jsonnet/` | Jsonnet templates and generated manifests |
| `vendor/` | Vendored Go dependencies |
| `Dockerfile` | Upstream container image |
| `Dockerfile.ocp` | OpenShift downstream container image |

## Conventions

- **Style / lint**: `make lint` (Go, YAML, Markdown, Jsonnet, Makefile linters) and `make test` must pass before merge.
- **Full verification**: `make verify` runs lint + test + generated-code verification.
- **Commits**: Use [conventional commits](https://www.conventionalcommits.org) (enforced by pre-commit hook). Allowed types: `build chore ci docs feat fix perf refactor revert style test`. Include `Assisted-by:` or `Generated-by:` trailers when AI assisted.
- **License headers**: All `.go`, `.yaml`, and `.jsonnet` files require Apache 2.0 boilerplate. Run `make lint` to check, `make lint_fix` to auto-fix.
- **Secrets**: Never commit real credentials.
- **Generated code**: After modifying `pkg/apis/`, run `make codegen` and `make manifests`. After modifying `jsonnet/`, run `make jsonnet_manifests`.
- **Golden tests**: Each resolver has golden test files under `tests/golden/`. Tests compare actual output against `.out.metrics` in these files.
- **Branch**: Downstream work targets `openshift-main`; upstream work targets `main`.
- **Code review**: focus on semantic correctness, hallucinated APIs, over-engineering, and consistency with existing patterns. Agent-assisted review supplements but never replaces human review.

## Common tasks (copy-paste prompts)

- "Add a new resolver in `pkg/resolver/` following the pattern in `pkg/resolver/cel.go`; implement the `Resolver` interface from `resolver.go`; add golden tests under `tests/golden/`."
- "Add a new metric family: update the `ResourceMetricsMonitor` CRD in `pkg/apis/`, run `make generate`, then implement handling in `internal/`."

## Things agents often get wrong here

Each pitfall includes a **Bad** (what agents do) vs **Good** (what to do instead) example.

### Editing `pkg/generated/`

This directory is auto-generated. Never edit files there directly.

```go
// BAD — editing pkg/generated/clientset/versioned/typed/.../resourcemetricsmonitors.go
func (c *resourceMetricsMonitors) Get(ctx context.Context, name string, ...) {
    // adding custom logic here — will be overwritten by codegen
}
```

```go
// GOOD — edit pkg/apis/resourcestatemetrics/v1alpha1/types.go, then run:
//   make codegen && make manifests
```

### Wrong resolver pattern

New resolvers must implement the `Resolver` interface from `pkg/resolver/resolver.go`.

```go
// BAD — ad-hoc function, not implementing the interface
func resolveMyExpression(query string, obj map[string]interface{}) string {
    return obj["spec"].(map[string]interface{})["field"].(string)
}
```

```go
// GOOD — implement the Resolver interface
type MyResolver struct{ /* ... */ }

func (r *MyResolver) Resolve(query string, unstructuredObjectMap map[string]interface{}) map[string]string {
    result := make(map[string]string)
    // resolve and populate result
    return result
}
```

### Missing golden tests

Every resolver change needs golden test coverage under `tests/golden/<resolver>/`, not just unit tests with hardcoded assertions. Read an existing golden file in the relevant resolver directory before writing a new one. Golden files are YAML with `in`, `metrics`, and `status` sections. Run `make golden_metrics` to regenerate `metrics.txt` (never edit it by hand).

### Resolver inheritance confusion

Resolvers inherit hierarchically: Store → Family → Metric. Set resolver at the highest applicable level.

```yaml
# BAD — setting resolver on every metric when store-level suffices
stores:
  - group: apps
    version: v1
    kind: Deployment
    families:
      - name: replicas
        metrics:
          - value: "resource.spec.replicas"
            resolver: cel          # redundant if all metrics use cel
          - value: "resource.status.readyReplicas"
            resolver: cel          # redundant
```

```yaml
# GOOD — set resolver at the store level, metrics inherit it
stores:
  - group: apps
    version: v1
    kind: Deployment
    resolver: cel
    families:
      - name: replicas
        metrics:
          - value: "resource.spec.replicas"
          - value: "resource.status.readyReplicas"
```

### Cardinality ignorance

Unbounded label expressions can explode cardinality. Always set limits.

```yaml
# BAD — no cardinality limit on a label that could have thousands of values
families:
  - name: pod_info
    metrics:
      - labels:
          - name: annotation
            value: "resource.metadata.annotations"   # unbounded
        value: "1"
```

```yaml
# GOOD — set cardinalityLimit at the family or store level
families:
  - name: pod_info
    cardinalityLimit: 500
    metrics:
      - labels:
          - name: annotation
            value: "resource.metadata.annotations"
        value: "1"
```

### Other common mistakes

- **Missing license headers** — All source files need Apache 2.0 boilerplate. Run `make lint` to catch, `make lint_fix` to auto-fix.
- **Dark work** — Starting or finishing work **without** a Jira ticket (MON project), or skipping **acceptance criteria** / agile hygiene.
- **Using `Co-Authored-By:` for AI** — Use `Assisted-by:` or `Generated-by:` trailers instead.
- **Forgetting `make manifests`** — After changing CRD types in `pkg/apis/`, manifests in `manifests/` must be regenerated.

## Hard boundaries

### `internal/`

- Use `sync.Map` for stores, `sync.Once` for scheme registration, `atomic.Bool` for sync state. Do not use bare maps or non-atomic booleans for shared state.
- Cardinality enforcement is three-tier (global → per-store → per-family) with warning ratio before hard cutoff. Never bypass — it prevents OOM in production.
- Never import from `pkg/generated/` directly — use interfaces and types from `pkg/apis/`.
- Never add a new HTTP endpoint without updating probe configuration.
- Never modify rate limiter constants without benchmarking under load.

### `pkg/apis/`

- Any change to `types.go` requires human review (L3 escalation) — this is the API contract.
- Never remove or rename a field without a deprecation plan — existing CRs in clusters depend on these fields.
- Never change `+kubebuilder` marker comments without understanding their effect on CRD generation.
- After modifying `types.go`: `make codegen` → `make manifests` → `make test`.

### `pkg/resolver/`

- Never return errors from `Resolve` — log and return an empty map. The caller handles absence.
- Never edit `resolver.go` to add resolver-specific logic — keep the interface clean.
- Never add external dependencies without human approval — resolvers run on every watch event and must be fast.
- Adding a new resolver: create `<name>.go`, add type to `ResolverType` in `pkg/apis/`, register in `internal/config.go`, add golden tests, run `make codegen && make manifests && make test`.

### `tests/golden/`

- Per-resolver subdirectories: `cel/`, `starlark/`, `unstructured/`. Place new tests in the matching subdirectory.
- Never delete a golden test without confirming the behavior it tested is intentionally removed.
- Always include `status.conditions` — they validate the controller's status reporting.

## Testing architecture

This project uses a two-layer testing approach:

- **Layer 1 — Development feedback** (`make test_unit`): Unit tests in `*_test.go` files alongside production code. Agents write these during implementation for fast feedback. Tests live in `internal/`, `pkg/resolver/`, `pkg/options/`, etc.
- **Layer 2 — Behavioral validation** (`make test_e2e`): Golden tests under `tests/golden/` define expected Prometheus output for each resolver independently of the implementation. E2e tests under `tests/` run against a live cluster. Golden files serve as the **information barrier** — they specify expected behavior without coupling to internal code.

Run `make test` for both layers. Run `make verify` for the full gate (lint + test + generated-code verification).

## Escalation guide

Operations in this repo map to escalation levels based on reversibility, blast radius, commitment, and visibility:

| Operation | Level | Rationale |
|-----------|-------|-----------|
| Edit tests, docs, golden files | **L0** — agent-autonomous | Fully reversible, internal only |
| Edit `internal/` or `pkg/resolver/` code | **L1** — informed | Reversible, low blast radius, human gets FYI |
| Change Makefile or CI configuration | **L2** — agent-decided | Moderate blast radius, clear options |
| Modify CRD types in `pkg/apis/` | **L3** — human-decided | API contract change, high commitment |
| Modify manifests/RBAC | **L3** — human-decided | Security-sensitive, affects deployed clusters |
| Push to `openshift-main` or `main` | **L4** — human-action | Externally visible, triggers CI/CD |
| Edit `pkg/generated/` | **Never** — run `make codegen` instead | Auto-generated, will be overwritten |

## Key links

- Upstream: [github.com/kubernetes-sigs/resource-state-metrics](https://github.com/kubernetes-sigs/resource-state-metrics)
