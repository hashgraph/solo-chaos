# solo-chaos

Chaos testing toolkit for [Hedera Solo](https://github.com/hiero-ledger/solo) networks on Kubernetes.

`solo-chaos` combines two layers:
- **Chaos experiments** — Kubernetes-native network and pod fault injection via [Chaos Mesh](https://chaos-mesh.org/), covering cross-region latency simulation, bandwidth throttling, network partitions, and pod kill/failure.
- **Hammer load generator** — Go CLI (`cmd/hammer`) that sends a continuous stream of Hedera transactions to consensus nodes, used to verify network resilience under chaos.

## Quick Start

```bash
# 1. Install prerequisites and deploy a 5-node Solo network on Kind
task setup
task deploy-network
task install-chaos-mesh

# 2. Run a chaos experiment
task chaos:consensus-node:network-netem

# 3. Or kill a node while hammer is running
task deploy-hammer-job
task chaos:consensus-node:pod-kill NODE_NAMES=node5
```

## Documentation

| Guide | What it covers |
|---|---|
| [`docs/getting-started.md`](docs/getting-started.md) | **Start here** — apply experiments to any Solo network (fresh or existing) |
| [`docs/consensus-node.md`](docs/consensus-node.md) | Cross-region latency simulation · Pod kill with live hammer load + node recovery |
| [`docs/block-node.md`](docs/block-node.md) | Block node network latency simulation |
| [`docs/README.md`](docs/README.md) | Full task catalog, project structure, contributor guide |
