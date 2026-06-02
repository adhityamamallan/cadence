# Plan: Frontend `GetServerVersion` Endpoint

A proposal for adding an endpoint to the **frontend** service that returns the
running server version.

## TL;DR

Most of the frontend's cross-cutting layers (rate limiting, metrics, auth,
cluster redirection, Thrift/gRPC protocol handlers) are **auto-generated** from a
single interface method, so the manual surface is smaller than it looks. The real
cost is the IDL change, which lives in an upstream repo.

## Key constraint: the IDL submodule

The frontend's public API is defined in IDL (Thrift + Protobuf) that lives in the
**`idls/` git submodule** (`github.com/uber/cadence-idl`). A genuinely new RPC
must land in that upstream repo first, then we bump the submodule pointer here.
This is the gating dependency for the whole effort.

### Two options

| Option | What it means | Effort |
|--------|---------------|--------|
| **A. New `GetServerVersion` RPC** | Dedicated public endpoint. Cleanest API. | Higher — upstream `cadence-idl` change + submodule bump |
| **B. Extend `GetClusterInfo`** | Add a `serverVersion` field to the existing `ClusterInfo` response. | Lower — IDL field add, no new RPC |

`GetClusterInfo` already exists as the natural home for cluster/server metadata
(it returns `SupportedClientVersions` today), so Option B is the pragmatic path if
the new-RPC friction isn't justified. The plan below describes **Option A**;
Option B follows the same Phases 2–6 with a smaller IDL change.

## Version source

Cadence does **not** currently bake in a build-time server version — the closest
existing values are the hardcoded SDK-compatibility constants in
`common/client/versionChecker.go`. A hardcoded version constant would drift from
reality, so the recommendation is the standard Go build-time pattern:

```go
// common/version/version.go
package version

var (
    Version   = "unknown" // set via -ldflags "-X .../version.Version=$(git describe)"
    GitCommit = "unknown"
    BuildDate = "unknown"
)
```

…with the `-X` linker flags added to the `make bins` link step. ~15 lines, and it
makes the endpoint actually useful. Can be deferred (return the constants, swap in
ldflags later) without changing the endpoint shape.

## Implementation phases

### Phase 1 — IDL (gating dependency)

In the `cadence-idl` repo, add the RPC to **both** protocols:

- `idls/proto/uber/cadence/api/v1/service_workflow.proto`: `GetServerVersionRequest{}`,
  `GetServerVersionResponse{}`, and `rpc GetServerVersion(...)` on `WorkflowAPI`.
- `idls/thrift/cadence.thrift`: matching `GetServerVersion()` RPC (next to
  `GetClusterInfo`, ~line 701).

Merge upstream, then bump the submodule pointer in this repo.

> **Local testing shortcut** (from `CLAUDE.md`): add
> `replace github.com/uber/cadence-idl => ./idls` to the bottom of `go.mod` to test
> local IDL changes before the upstream merge lands. Remove before committing.

### Phase 2 — Version source

- Add `common/version/version.go` (vars above).
- Add `-X` ldflags to the `make bins` link step in the `Makefile`.

### Phase 3 — Internal types + mappers

- `common/types/shared.go`: add the `GetServerVersionResponse` struct.
- `common/types/mapper/proto/api.go` + the thrift mapper: add
  `To/FromGetServerVersionResponse`.
- Add round-trip / fuzz tests (`ToX(FromX(x)) == x`) following
  `common/types/mapper/proto/api_test.go`. The repo requires this for all mappers.

### Phase 4 — Handler (triggers codegen)

- `service/frontend/api/interface.go`: add
  `GetServerVersion(context.Context) (*types.GetServerVersionResponse, error)` to
  the `Handler` interface. **This is what drives regeneration of all wrappers.**
- `service/frontend/api/handler.go`: implement it (model on `GetClusterInfo`,
  ~line 3254) — read from `common/version` and return.

### Phase 5 — Metrics + codegen

- `common/metrics/defs.go`: add `FrontendGetServerVersionScope`.
- `service/frontend/templates/metered.tmpl`: add the method to the
  `$nonDomainSpecificAPIs` list (it has no domain).
- Run `make go-generate` (or `make pr GEN_DIR=service/frontend`) to regenerate the
  versioncheck, ratelimited, metered, clusterredirection, accesscontrolled,
  **thrift**, and **grpc** wrappers. Never edit `*_generated.go` by hand.

### Phase 6 — Finish

- `make build` to compile-check.
- `make test` for the mapper/handler tests.
- `make pr` (tidy → generate → fmt → lint) before opening the PR.

## What you do *not* have to touch

`service/frontend/service.go` wiring, dispatcher registration, and all six wrapper
layers — all generated/driven off the interface method.

## Risks / open questions

- **Phase 1 is the real cost**: a separate upstream repo + submodule bump blocks
  everything else. If that friction isn't worth it for a version endpoint, prefer
  Option B (extend `GetClusterInfo`).
- **Version source decision** still open: build-time ldflags (recommended) vs.
  deferred constant.
- Decide whether the same value should also be exposed on the **admin** API.
