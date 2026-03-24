# Developer Guide

This guide is for engineers who want to contribute to `solo-chaos`.
It focuses on local setup, the expected dev workflow, and project-specific patterns.

## Prerequisites

Install these tools before running tasks:

- Docker Desktop (macOS: allocate enough CPU/RAM for Kind + Solo)
- [Task](https://taskfile.dev/)
- [Kind](https://kind.sigs.k8s.io/)
- [kubectl](https://kubernetes.io/docs/tasks/tools/)
- [Helm](https://helm.sh/)
- [jq](https://stedolan.github.io/jq/)
- `envsubst` (via `gettext`)
- Node.js + npm (required for `solo` CLI)
- [solo CLI](https://github.com/hashgraph/solo)
- Go 1.24+

## Environment Setup

Use repo tasks instead of manual setup when possible:

```bash
# On macOS
task setup

# Validate local environment
task doctor
```

For Linux, `task setup` and `task doctor` are also available via `dev/taskfile/Taskfile.linux.yml`.

## Build/Test Workflow (Run In Order)

Use this order before opening a PR:

```bash
task build
task lint:check
task test:unit
```

Notes:
- `task build` vendors modules, generates version data, and cross-builds `bin/hammer-{linux,darwin}-{amd64,arm64}`.
- `task lint:check` enforces Go formatting (`go fmt`).
- `task test:unit` runs `chaos/test-validations.sh` and Go tests under `./cmd/...` with race + coverage output.
- If `go-testreport` is not found, add your Go bin path:

```bash
export PATH="$PATH:$(go env GOPATH)/bin"
```

Known caveat from current Taskfiles:
- `task test:coverage` references `./pkg/...` and may fail.

## Codebase Orientation

- `Taskfile.yml`: root dispatcher; includes OS and build taskfiles.
- `dev/taskfile/Taskfile.root.yml`: infrastructure tasks (`deploy-network`, `install-chaos-mesh`, `refresh-node`, job deploy/destroy).
- `dev/taskfile/Taskfile.build.yml`: build, lint, test, image tasks.
- `cmd/hammer/`: Go CLI app and transaction generator.
- `chaos/`: chaos taskfiles and manifests (`pod/`, `network/`, `block-node/`).
- `dev/k8s/`: Kubernetes manifests (`solo-chaos-hammer-job.yml`, `cluster-diagnostics.yaml`).

## Common Contributor Flows

### Run hammer locally

```bash
task build
./bin/hammer-$(go env GOOS)-$(go env GOARCH) tx --help
```

### Deploy hammer job to cluster

```bash
task build:image
task deploy-hammer-job
kubectl logs -n solo -l app=solo-chaos-hammer -f
```

### Run chaos from repo root

```bash
task chaos:pod:consensus-pod-kill NODE_NAMES=node5
task chaos:network:consensus-network-netem
```

### Run chaos directly from `chaos/`

```bash
cd chaos
task --list
task pod:consensus-pod-kill NODE_NAMES=node5
task network:consensus-network-netem
```

## Troubleshooting

- If setup-related commands fail, run:

```bash
task doctor
```

- If a killed node does not recover cleanly, refresh it:

```bash
task refresh-node NODE=node5
```

- To clear network chaos experiments:

```bash
task chaos:network:cleanup-networkchaos
```
