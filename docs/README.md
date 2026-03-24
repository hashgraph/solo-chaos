# solo-chaos Docs

Reference documentation for running chaos experiments and load testing against Solo networks.

## Scenario Guides

| Guide | Scenarios covered |
|---|---|
| [`docs/consensus-node.md`](consensus-node.md) | Network latency simulation · Pod kill with hammer load + node recovery |
| [`docs/block-node.md`](block-node.md) | Block-node network latency simulation |

## Contributor Docs

| Guide | Focus |
|---|---|
| [`docs/developer-guide.md`](developer-guide.md) | Prerequisites, setup, build/lint/test workflow, and contributor troubleshooting |

## Project Structure

```
solo-chaos/
├── chaos/                              # Chaos experiments and taskfiles
│   ├── Taskfile.yml                    # Main chaos taskfile (includes consensus-node + block-node)
│   ├── Taskfile.utils.yml              # Shared utility tasks (show-experiment-status)
│   ├── consensus-node/                 # Consensus node chaos experiments
│   │   ├── Taskfile.yml                # Consensus node tasks
│   │   ├── network/                    # Network chaos manifests (netem, bandwidth, partition)
│   │   └── pod/                        # Pod chaos manifests (kill, failure)
│   └── block-node/                     # Block node chaos experiments
│       ├── Taskfile.yml                # Block node tasks
│       └── network/                    # Block node network chaos manifests
├── dev/
│   ├── taskfile/                       # Root and build taskfiles
│   │   ├── Taskfile.build.yml          # Build, test, lint tasks
│   │   ├── Taskfile.root.yml           # Infrastructure tasks (deploy, chaos-mesh)
│   │   └── Taskfile.{os}.yml          # OS-specific setup
│   ├── config/                         # Example hammer config
│   └── k8s/                            # Kubernetes manifests (hammer job, diagnostics)
└── cmd/hammer/                         # Hammer CLI load generator
```

## Running Chaos Tasks

Tasks can be run from three different working directories — the prefix changes but the underlying operation is the same:

```bash
# From repo root
task chaos:consensus-node:network-netem
task chaos:consensus-node:pod-kill NODE_NAMES=node5
task chaos:block-node:network-netem

# From chaos/  (shorter prefix)
cd chaos
task consensus-node:network-netem
task consensus-node:pod-kill NODE_NAMES=node5
task block-node:network-netem

# From inside a component directory (no prefix)
cd chaos/consensus-node
task network-netem
task pod-kill NODE_NAMES=node5

cd chaos/block-node
task network-netem
```

## Task Catalog

### Core
| Task | Description |
|---|---|
| `task setup` | Install prerequisites |
| `task deploy-network` | Deploy a 5-node Solo network on Kind |
| `task destroy-network` | Tear down the network |
| `task install-chaos-mesh` | Install Chaos Mesh via Helm |
| `task uninstall-chaos-mesh` | Remove Chaos Mesh |
| `task refresh-node NODE=<name>` | Re-setup and restart a consensus node (e.g. after pod kill) |

### Build
| Task | Description |
|---|---|
| `task build` | Cross-compile hammer for linux/darwin × amd64/arm64 |
| `task build:image` | Build the Docker image |
| `task lint:check` | Verify Go formatting |
| `task test:unit` | Run unit tests + env validation tests |

### Chaos — Consensus Node
| Task | Description |
|---|---|
| `task chaos:consensus-node:network-netem` | Apply global cross-region latency rules |
| `task chaos:consensus-node:network-bandwidth NODE_NAMES=<nodes> RATE=<rate>` | Limit bandwidth |
| `task chaos:consensus-node:network-partition SOURCE_REGION=<r> TARGET_REGION=<r>` | Partition two regions |
| `task chaos:consensus-node:pod-kill NODE_NAMES=<nodes>` | Kill consensus pod(s) |
| `task chaos:consensus-node:pod-failure NODE_NAMES=<nodes> DURATION=<d>` | Trigger pod failure |

### Chaos — Block Node
| Task | Description |
|---|---|
| `task chaos:block-node:network-netem` | Apply US-to-AP latency rules for block node traffic |

### Chaos — Shared Utilities
| Task | Description |
|---|---|
| `task chaos:deploy-cluster-diagnostics REGION=<r>` | **Cleans up all existing NetworkChaos**, deploys diagnostics pod, then applies netem |
| `task chaos:exec-cluster-diagnostics` | Exec into the diagnostics pod |
| `task chaos:cleanup-networkchaos` | Delete all NetworkChaos resources |
| `task chaos:cleanup-cluster-diagnostics` | Remove the diagnostics pod |

### Cross-Region RTT Reference

Latency values applied by `consensus-network-netem` and `block-node-netem`:

| Source → Target | One-way | RTT |
|---|---|---|
| us → us | 10ms | 20ms |
| eu → eu | 10ms | 20ms |
| ap → ap | 10ms | 20ms |
| us ↔ eu | 50ms | 100ms |
| us ↔ ap | 100ms | 200ms |
| eu ↔ ap | 150ms | 300ms |

The block-node experiment (`chaos:block-node:network-netem`) applies the `us → ap` rule (100ms one-way / 200ms RTT) between US-region consensus nodes and the block node.

### Chaos — Utility
| Task | Description |
|---|---|
| `task chaos:show-experiment-status NAME=<n> TYPE=<t>` | Show experiment events |
| `task deploy-hammer-job` | Build image, load into Kind, and run the hammer job |
| `task destroy-hammer-job` | Remove the hammer job |
