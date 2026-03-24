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
│   ├── Taskfile.yml                    # Main chaos taskfile
│   ├── Taskfile.chaos.network.yml      # Network chaos tasks
│   ├── Taskfile.chaos.pod.yml          # Pod chaos tasks
│   ├── network/                        # Network chaos manifests (netem, bandwidth, partition)
│   ├── pod/                            # Pod chaos manifests (kill, failure)
│   └── block-node/                     # Block-node specific manifests
├── dev/
│   ├── taskfile/                       # Root and build taskfiles
│   │   ├── Taskfile.build.yml          # Build, test, lint tasks
│   │   ├── Taskfile.root.yml           # Infrastructure tasks (deploy, chaos-mesh)
│   │   └── Taskfile.{os}.yml          # OS-specific setup
│   ├── config/                         # Example hammer config
│   └── k8s/                            # Kubernetes manifests (hammer job, diagnostics)
└── cmd/hammer/                         # Hammer CLI load generator
```

## Running Chaos Tasks from `chaos/`

Task names are shorter when run from inside the `chaos/` directory:

```bash
cd chaos
task --list
task pod:consensus-pod-kill NODE_NAMES=node5
task network:consensus-network-netem
task network:network-partition-by-region SOURCE_REGION=us TARGET_REGION=eu
task show-experiment-status NAME=<name> TYPE=<PodChaos|NetworkChaos>
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

### Chaos — Pod
| Task | Description |
|---|---|
| `task chaos:pod:consensus-pod-kill NODE_NAMES=<nodes>` | Kill consensus pod(s) |
| `task chaos:pod:consensus-pod-failure NODE_NAMES=<nodes> DURATION=<d>` | Trigger pod failure |

### Chaos — Network
| Task | Description |
|---|---|
| `task chaos:network:consensus-network-netem` | Apply global cross-region latency rules |
| `task chaos:network:consensus-network-bandwidth NODE_NAMES=<nodes> RATE=<rate>` | Limit bandwidth |
| `task chaos:network:network-partition-by-region SOURCE_REGION=<r> TARGET_REGION=<r>` | Partition two regions |
| `task chaos:network:block-node-netem` | Apply block-node latency rules |
| `task chaos:network:deploy-cluster-diagnostics REGION=<r>` | **Cleans up all existing NetworkChaos**, deploys diagnostics pod, then applies netem |
| `task chaos:network:exec-cluster-diagnostics` | Exec into the diagnostics pod |
| `task chaos:network:cleanup-cluster-diagnostics` | Remove the diagnostics pod |
| `task chaos:network:cleanup-networkchaos` | Delete all NetworkChaos resources |

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

The block-node experiment (`block-node-netem`) applies the `us → ap` rule (100ms one-way / 200ms RTT) between US-region consensus nodes and the block node.

### Chaos — Utility
| Task | Description |
|---|---|
| `task chaos:show-experiment-status NAME=<n> TYPE=<t>` | Show experiment events |
| `task deploy-hammer-job` | Build image, load into Kind, and run the hammer job |
| `task destroy-hammer-job` | Remove the hammer job |
