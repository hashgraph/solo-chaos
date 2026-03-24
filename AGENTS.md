# AGENTS Guide for solo-chaos

## Big Picture
- This repo combines two layers: a Go CLI load generator (`cmd/hammer`) and Kubernetes chaos orchestration (`chaos/` + root Taskfiles).
- Typical flow: deploy Solo network (`task deploy-network`) -> run hammer job (`task deploy-hammer-job`) -> inject chaos (`task chaos:...`) -> inspect experiment events.
- `hammer` sends Hedera transactions to consensus nodes using config-defined node/account mappings (`cmd/hammer/config/config.go`).

## Architecture Map
- CLI entrypoint: `cmd/hammer/main.go` -> `commands.Execute()`.
- Command wiring: `cmd/hammer/commands/root.go` registers global flags and mandatory `--config`, then adds `tx` subcommand.
- Transaction engine: `cmd/hammer/commands/tx.go` creates a shared Hedera client, spawns bot goroutines, and computes live TPS from receipt channel events.
- Config source: `dev/config/hammer-local.yml` is the canonical example schema (`consensusNodes`, `mirrorNodes`, `operator`).
- Chaos orchestration: `chaos/Taskfile.yml` includes `consensus-node/Taskfile.yml` and `block-node/Taskfile.yml`; tasks render YAML via `envsubst` and apply with `kubectl`. Shared utilities (diagnostics, cleanup) are in `chaos/Taskfile.yml`; shared task helpers are in `chaos/Taskfile.utils.yml`.

## Build/Test Workflow (Do This Order)
- Use Task, not ad-hoc `go build`/`go test` commands.
- Run in order: `task build` -> `task lint:check` -> `task test:unit`.
- `task build` runs `vendor`, `clean`, `generate`, and cross-builds `bin/hammer-{linux,darwin}-{amd64,arm64}`.
- `task test:unit` first runs `chaos/test-validations.sh` (env validation coverage), then Go tests for `./cmd/...` with race + coverage.
- If `go-testreport` is not on PATH, ensure `$(go env GOPATH)/bin` is added before tests.
- Known issue: `task test:coverage` references `./pkg/...` and is currently unreliable.

## Project-Specific Conventions
- Root Taskfile is only a dispatcher (`Taskfile.yml` includes `dev/taskfile/Taskfile.{os}.yml` + `Taskfile.build.yml`). Put automation changes in `dev/taskfile/*`.
- Chaos tasks always validate inputs via `chaos/validate-env.sh` before applying manifests.
- Chaos task names differ by working directory:
  - repo root: `task chaos:consensus-node:pod-kill NODE_NAMES=node5`
  - inside `chaos/`: `task consensus-node:pod-kill NODE_NAMES=node5`
  - inside `chaos/consensus-node/`: `task pod-kill NODE_NAMES=node5`
- Network netem experiments intentionally use fixed resource names (for replace-not-accumulate behavior), see `chaos/consensus-node/network/netem-*.yml` and `chaos/block-node/network/netem-us-ohio.yml`.

## Integration Points
- External CLIs used heavily in Taskfiles: `solo`, `kind`, `kubectl`, `helm`, `docker`, `envsubst`, `jq`.
- Chaos Mesh namespace and CRDs are assumed (`install-chaos-mesh` in `dev/taskfile/Taskfile.root.yml`).
- Runtime behavior depends on env vars such as `SOLO_NAMESPACE`, `SOLO_CLUSTER_NAME`, `NODES`, and chaos task args (`NODE_NAMES`, `REGION`, `RATE`).

## Fast Verification Commands
- `task --list`
- `./bin/hammer-$(go env GOOS)-$(go env GOARCH) --help`
- `./bin/hammer-$(go env GOOS)-$(go env GOARCH) tx --help`
- `task chaos:consensus-node:network-netem`

