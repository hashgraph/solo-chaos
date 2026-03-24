# Getting Started — Applying Chaos Experiments to Your Solo Network

This guide explains how to use `solo-chaos` to inject and observe failures against a running Hedera Solo network. You bring the network; this repo provides the chaos tooling.

Two paths are supported:
- **Option A** — deploy a fresh network with `task deploy-network` (fastest way to get started)
- **Option B** — point the tooling at an existing Kubernetes deployment that carries the expected pod labels

---

## Prerequisites

Install these tools before running any experiments:

| Tool | Purpose |
|---|---|
| [kubectl](https://kubernetes.io/docs/tasks/tools/) | Apply chaos manifests and inspect pods |
| [Task](https://taskfile.dev/) v3.x | Run experiments via `task chaos:...` commands |
| `envsubst` (via `gettext`) | Template substitution in chaos manifests |
| [Helm](https://helm.sh/) | Install Chaos Mesh (one-time) |

---

## Step 1: Connect to Your Network

### Option A — Deploy a fresh Solo network

The quickest path. This deploys a 5-node Solo network on a local Kind cluster:

```bash
task setup          # install all prerequisites (macOS / Linux)
task deploy-network # create Kind cluster + deploy Solo network
```

After this, `SOLO_NAMESPACE=solo` is the default — skip directly to Step 3.

### Option B — Use an existing network

Export the namespace your Solo pods are running in:

```bash
export SOLO_NAMESPACE=<your-namespace>   # default: solo
```

Then verify your pods carry the labels described in the next section. If they were deployed by the Solo CLI, the labels are already present.

---

## Required Pod Labels

Chaos experiments target pods using Kubernetes label selectors. The table below lists what each component needs.

### Consensus Nodes

| Label | Required values | Purpose |
|---|---|---|
| `solo.hedera.com/type` | `network-node` | Identifies all consensus node pods |
| `solo.hedera.com/region` | `us`, `eu`, or `ap` | Routes region-specific latency rules |
| `solo.hedera.com/node-name` | e.g. `node1`, `node5` | Targets individual pods for pod chaos |

### Block Node

| Label | Required values | Purpose |
|---|---|---|
| `solo.hedera.com/type` | `block-node` | Identifies block node pods |
| `solo.hedera.com/region` | `us`, `eu`, or `ap` | Routes region-specific latency rules |

### Verify labels

```bash
kubectl get pods -n $SOLO_NAMESPACE --show-labels
```

### Apply missing labels

**Consensus nodes** — if you deployed with `task deploy-network` or the Solo CLI, labels are already set. Otherwise, add the required labels manually:

```bash
kubectl label pod <pod-name> -n $SOLO_NAMESPACE solo.hedera.com/region=us
```

**Block node** — use `task deploy-block-node`, which deploys the block node and applies the chaos labels automatically:

```bash
task deploy-block-node
```

If you already have a running block node and only need to add the labels, use the actual instance name (e.g. `block-node-1`, `block-node-2` — check with `kubectl get pods -n $SOLO_NAMESPACE --show-labels`):

```bash
kubectl label pods -n $SOLO_NAMESPACE \
  -l app.kubernetes.io/instance=<block-node-instance> \
  solo.hedera.com/type=block-node \
  solo.hedera.com/region=ap \
  --overwrite
```


---

## Step 2: Install Chaos Mesh

Chaos Mesh must be running in your cluster before any experiment can be applied. The task below is idempotent — safe to run multiple times:

```bash
task install-chaos-mesh
```

If you prefer to install manually without Task:

```bash
helm repo add chaos-mesh https://charts.chaos-mesh.org
kubectl create ns chaos-mesh
helm install chaos-mesh chaos-mesh/chaos-mesh -n chaos-mesh \
  --version 2.7.2 \
  --set chaosDaemon.runtime=containerd \
  --set chaosDaemon.socketPath=/run/containerd/containerd.sock \
  --set chaosDaemon.hostPID=true \
  --set chaosDaemon.privileged=true \
  --set chaosDaemon.mountHostLibModules=true
kubectl wait --namespace chaos-mesh \
  --for=condition=Ready pod \
  --selector=app.kubernetes.io/instance=chaos-mesh \
  --timeout=180s
```

---


## Step 3: Run Chaos Experiments

All experiments are idempotent — re-running replaces the active experiment rather than accumulating duplicates.

### Consensus Node Experiments

```bash
# Cross-region latency simulation (all 10 region pairs)
task chaos:consensus-node:network-netem

# Bandwidth throttle on specific nodes
task chaos:consensus-node:network-bandwidth NODE_NAMES=node1,node2 RATE=100mbps

# Network partition between two regions
task chaos:consensus-node:network-partition SOURCE_REGION=us TARGET_REGION=eu

# Kill a node pod
task chaos:consensus-node:pod-kill NODE_NAMES=node5

# Inject pod failure for a fixed duration
task chaos:consensus-node:pod-failure NODE_NAMES=node5 DURATION=30s
```

### Block Node Experiments

```bash
# US-to-AP latency on block node traffic (100ms one-way / 200ms RTT)
task chaos:block-node:network-netem
```

### Shorter task names from inside the `chaos/` directory

```bash
cd chaos
task consensus-node:network-netem
task consensus-node:pod-kill NODE_NAMES=node5
task block-node:network-netem
```

Or from inside a component directory (no prefix needed):

```bash
cd chaos/consensus-node
task pod-kill NODE_NAMES=node5
task network-netem
```

---

## Verifying an Experiment is Active

```bash
# Show events for a specific experiment
task chaos:show-experiment-status NAME=solo-chaos-network-netem-us-to-eu TYPE=NetworkChaos

# List all active NetworkChaos resources
kubectl get networkchaos -n chaos-mesh

# List all active PodChaos resources
kubectl get podchaos -n chaos-mesh
```

---

## Measuring the Impact (Cluster Diagnostics Pod)

To observe latency changes before and after chaos injection, deploy a diagnostics pod with `ping`, `iperf3`, and `curl` pre-installed:

```bash
# Deploy a diagnostics pod labelled as the 'us' region
task chaos:deploy-cluster-diagnostics REGION=us

# Exec into it
task chaos:exec-cluster-diagnostics
```

Inside the pod, ping a consensus node to measure latency:

```bash
ping <pod-IP>
```

Run your chaos experiment from a second terminal, then observe the latency change in the ping output. See [`docs/consensus-node.md`](consensus-node.md) and [`docs/block-node.md`](block-node.md) for full step-by-step walkthroughs with expected output.

Now, delete the experiment and watch latency return to normal:

```bash 
kubectl delete networkchaos solo-chaos-network-netem-block-node-us-to-ap -n chaos-mesh  
```

---

## Cleanup

```bash
# Full teardown — removes all chaos experiments and the diagnostics pod
task chaos:cleanup-all

# Selective cleanup
task chaos:cleanup-networkchaos        # remove NetworkChaos resources only
task chaos:cleanup-podchaos            # remove PodChaos resources only
task chaos:cleanup-cluster-diagnostics # remove diagnostics pod only
```

---

## What's Next

| Guide | What it covers |
|---|---|
| [`docs/consensus-node.md`](consensus-node.md) | Full walkthroughs: latency simulation and pod kill with live load |
| [`docs/block-node.md`](block-node.md) | Block node latency verification with baseline measurements |
| [`docs/developer-guide.md`](developer-guide.md) | How to contribute new experiments or add a new component |
