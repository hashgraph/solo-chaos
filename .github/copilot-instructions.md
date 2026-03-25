# Copilot Instructions for solo-chaos Repository

## Repository Overview

**solo-chaos** is a reusable chaos testing framework for Solo test networks. It tests resilience and stability by introducing controlled failures and disruptions to Hedera Solo networks running on Kubernetes.

**Key Technologies:**
- **Language:** Go 1.24+ (requires Go 1.25+ for CI)
- **Build System:** Task (Taskfile.dev) - NOT Make
- **Container:** Docker with multi-architecture builds (linux/darwin, amd64/arm64)
- **Orchestration:** Kubernetes with Chaos Mesh for chaos experiments
- **Dependencies:** Hedera SDK, Cobra CLI framework, Viper configuration

**Project Size:** ~60 YAML files, Go source in `/cmd/hammer/` and `/internal/`

## Critical Build Requirements

### Prerequisites (ALWAYS install these first)
```bash
# Install Task (required - NOT optional)
wget -qO- https://github.com/go-task/task/releases/latest/download/task_linux_amd64.tar.gz | sudo tar -C /usr/local/bin -xzf - task

# Fix PATH for Go tools (CRITICAL for tests)
export PATH=$PATH:$(go env GOPATH)/bin
```

### Build Commands (Use Task, NOT go commands directly)

**ALWAYS run these commands in this exact order:**

1. **Build:** `task build` (takes ~30-60s)
   - Runs: vendor → clean → generate → build for all architectures
   - Generates binaries: `bin/hammer-{os}-{arch}`
   - ALWAYS run before any other operations

2. **Lint Check:** `task lint:check` 
   - Uses `go fmt` - will fail if any formatting issues exist
   - Must pass for CI/CD

3. **Unit Tests:** `task test:unit` (takes ~5s)
   - Requires PATH to include `$(go env GOPATH)/bin`
   - Generates `unit-test-report.md`
   - All 6 tests should pass

4. **List All Tasks:** `task --list`

### Known Issues & Workarounds

**CRITICAL:** The following commands have bugs:
- `task test:coverage` - FAILS due to non-existent `./pkg/...` directory reference
- Docker builds may fail in restricted environments due to `snapshot.debian.org` dependency

**Build Failures:** If Task commands fail:
1. Verify Task is installed: `task --version` (should be 3.39.2+)
2. Verify Go version: `go version` (should be 1.24+)
3. Clean and retry: `task clean && task build`

## GitHub CI/CD Pipeline

**Pull Request Checks** (`.github/workflows/`):
- **Code Compilation:** `task build && task lint:check`
- **Unit Tests:** `task test:unit`
- **Dependencies:** Task 3.39.2, Go 1.25+
- All workflows use `arduino/setup-task` action

**Required for PR approval:**
1. All builds must pass (`task build`)
2. Code formatting must be clean (`task lint:check`)  
3. All unit tests must pass (`task test:unit`)

## Project Architecture

### Directory Structure
```
/cmd/hammer/           # Main Go application (chaos testing CLI)
├── commands/          # Cobra CLI commands and tests
├── config/           # Configuration handling
└── main.go           # Application entry point

/chaos/               # Chaos experiment configurations
├── network/          # Network chaos (bandwidth, latency)
├── pod/             # Pod chaos (kill, failure)  
└── Taskfile.yml     # Chaos-specific tasks

/dev/                # Development tooling
├── taskfile/        # Build system configuration
│   ├── Taskfile.build.yml    # Core build tasks
│   ├── Taskfile.root.yml     # Infrastructure tasks
│   └── Taskfile.{os}.yml     # OS-specific tasks
└── k8s/            # Kubernetes manifests

/internal/version/   # Version generation
/.github/workflows/  # CI/CD pipeline
```

### Key Configuration Files
- `Taskfile.yml` - Main build configuration (includes OS and build taskfiles)
- `go.mod` - Go dependencies (Hedera SDK, Cobra, Viper)
- `.testcoverage.yml` - Coverage thresholds (80% total, 60% file)
- `Dockerfile` - Multi-stage build with deterministic timestamps

## Development Workflow

### Making Code Changes
1. **Always run build first:** `task build`
2. **Check formatting:** `task lint:check` 
3. **Run tests:** `task test:unit`
4. **For Kubernetes work:** Understand chaos experiments in `/chaos/`

### Adding New Features
- **CLI commands:** Add to `/cmd/hammer/commands/`
- **Chaos experiments:** Add YAML to `chaos/consensus-node/network/`, `chaos/consensus-node/pod/`, or `chaos/block-node/network/`
- **Build changes:** Modify `/dev/taskfile/Taskfile.build.yml`

### Testing Changes
```bash
# Basic validation
task build && task lint:check && task test:unit

# Run specific hammer commands
./bin/hammer-$(go env GOOS)-$(go env GOARCH) --help
./bin/hammer-$(go env GOOS)-$(go env GOARCH) tx --help
```

## Kubernetes Integration

**Chaos Mesh Experiments:**
- **Pod Chaos:** `task chaos:consensus-node:pod-kill NODE_NAMES=node5`
- **Network Chaos:** `task chaos:consensus-node:network-bandwidth NODE_NAMES=node1 RATE=1gbps`
- **Block Node Chaos:** `task chaos:block-node:network-netem`

**Infrastructure Tasks:**
- `task deploy-network` - Deploy 5-node Solo network
- `task install-chaos-mesh` - Install Chaos Mesh
- `task deploy-hammer-job` - Deploy chaos testing job

**Environment Variables:**
- `SOLO_CLUSTER_NAME=solo` 
- `SOLO_NAMESPACE=solo`
- `NODES=5` (configurable)

## Common Validation Steps

Before submitting changes, ALWAYS verify:
1. `task build` completes successfully
2. `task lint:check` passes (no formatting issues)
3. `task test:unit` passes (6/6 tests)
4. If modifying chaos configs, validate YAML syntax
5. New Go code includes appropriate tests

## Root Directory Contents
```
Taskfile.yml          # Main build entry point
go.mod/go.sum         # Go dependencies  
Dockerfile            # Container build
README.md             # Usage documentation
LICENSE               # Apache 2.0
.testcoverage.yml     # Coverage configuration
.releaserc            # Release configuration
repro-sources-list.sh # Debian snapshot script
```

## Trust These Instructions

These instructions are comprehensive and tested. Only search for additional information if:
- Build commands fail with errors not covered here
- New dependencies or tools are introduced
- Kubernetes chaos experiments behave unexpectedly

The build system and test suite are working correctly when following these exact procedures.