# Getting Started — Applying Chaos Experiments to Your Solo Network

This guide explains how to use `solo-chaos` to inject and observe failures against a running Hedera Solo network. You
bring the network; this repo provides the chaos tooling.

Two paths are supported:

- **Option A** — deploy a fresh network with `task deploy-network` (fastest way to get started)
- **Option B** — point the tooling at an existing Kubernetes deployment that carries the expected pod labels

---

## Prerequisites

Install these tools before running any experiments:

| Tool                                               | Purpose                                       |
|----------------------------------------------------|-----------------------------------------------|
| [kubectl](https://kubernetes.io/docs/tasks/tools/) | Apply chaos manifests and inspect pods        |
| [Task](https://taskfile.dev/) v3.x                 | Run experiments via `task chaos:...` commands |
| `envsubst` (via `gettext`)                         | Template substitution in chaos manifests      |
| [Helm](https://helm.sh/)                           | Install Chaos Mesh (one-time)                 |

---

## Step 1: Connect to Your Network

### Option A — Deploy a fresh Solo network

The quickest path. This deploys a 5-node Solo network on a local Kind cluster:

```bash
task setup          # install all prerequisites (macOS / Linux)
task deploy-network # create Kind cluster + deploy Solo network
task deploy-block-node # deploy block node with chaos labels
```

After this, `SOLO_NAMESPACE=solo` is the default — skip directly to Step 2.

### Option B — Use an existing network

Export the namespace your Solo pods are running in:

```bash
export SOLO_NAMESPACE=<your-namespace>   # default: solo
```

Then verify your pods carry the labels described in the next section. If they were deployed by the Solo CLI, the labels
are already present.

---

## Required Pod Labels

Chaos experiments target pods using Kubernetes label selectors. The table below lists what each component needs.

### Consensus Nodes

| Label                       | Required values       | Purpose                               |
|-----------------------------|-----------------------|---------------------------------------|
| `solo.hedera.com/type`      | `network-node`        | Identifies all consensus node pods    |
| `solo.hedera.com/region`    | `us`, `eu`, or `ap`   | Routes region-specific latency rules  |
| `solo.hedera.com/node-name` | e.g. `node1`, `node5` | Targets individual pods for pod chaos |

### Block Node

| Label                    | Required values     | Purpose                              |
|--------------------------|---------------------|--------------------------------------|
| `solo.hedera.com/type`   | `block-node`        | Identifies block node pods           |
| `solo.hedera.com/region` | `us`, `eu`, or `ap` | Routes region-specific latency rules |

### Verify labels

```bash
kubectl get pods -n $SOLO_NAMESPACE --show-labels
```

### Apply missing labels

**Consensus nodes** — if you deployed with `task deploy-network` or the Solo CLI, labels are already set. Otherwise, add
the required labels manually:

```bash
kubectl label pod <pod-name> -n $SOLO_NAMESPACE solo.hedera.com/region=us
```

**Block node** — use `task deploy-block-node`, which deploys the block node and applies the chaos labels automatically:

```bash
task deploy-block-node
```

If you already have a running block node and only need to add the labels, use the actual instance name (e.g.
`block-node-1`, `block-node-2` — check with `kubectl get pods -n $SOLO_NAMESPACE --show-labels`):

```bash
kubectl label pods -n $SOLO_NAMESPACE \
  -l app.kubernetes.io/instance=<block-node-instance> \
  solo.hedera.com/type=block-node \
  solo.hedera.com/region=ap \
  --overwrite
```

---

## Step 2: Install Chaos Mesh

Chaos Mesh must be running in your cluster before any experiment can be applied. The task below is idempotent — safe to
run multiple times:

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

> Fixed-name netem experiments are idempotent — re-running them replaces the active experiment.
Experiments whose tasks/manifests include `${UUID}` in `metadata.name` (for example, `pod-kill`, `pod-failure`,
`network-bandwidth`, `network-partition`) create a new Chaos Mesh resource on each run and will accumulate unless you
delete prior runs or let them expire.

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

### Verifying an Experiment is Active

```bash
# Show events for a specific experiment
task chaos:show-experiment-status NAME=solo-chaos-network-netem-us-to-eu TYPE=NetworkChaos

# List all active NetworkChaos resources
kubectl get networkchaos -n chaos-mesh

# List all active PodChaos resources
kubectl get podchaos -n chaos-mesh
```

---

### Measuring the Impact (Cluster Diagnostics Pod)

The diagnostics pod gives you a network-connected container with `ping`, `iperf3`, and `curl` pre-installed. Deploy it
before applying chaos to capture a baseline, then watch latency change in real time.

### Scenario A — Consensus Node Latency

**1. Deploy the diagnostics pod** (no chaos yet):

```bash
task chaos:deploy-cluster-diagnostics REGION=us
```

**2. Get the IP of an EU-region consensus node:**

```bash
kubectl get pods -n $SOLO_NAMESPACE \
  -l solo.hedera.com/type=network-node,solo.hedera.com/region=eu \
  -o jsonpath='{.items[0].status.podIP}'
```

**3. Exec in and measure baseline latency:**

```bash
task chaos:exec-cluster-diagnostics
# inside the pod:
ping <eu-node-pod-IP>
```

Expected baseline (no chaos active):

```
64 bytes from 10.244.0.12: icmp_seq=1 ttl=63 time=0.04 ms
64 bytes from 10.244.0.12: icmp_seq=2 ttl=63 time=0.03 ms
```

**4. From a second terminal, apply cross-region latency:**

```bash
task chaos:consensus-node:network-netem
```

**5. Latency in the diagnostics pod jumps to ~50ms** (us→eu rule, 50ms one-way):

```
64 bytes from 10.244.0.12: icmp_seq=50 ttl=63 time=49.4 ms
64 bytes from 10.244.0.12: icmp_seq=51 ttl=63 time=51.7 ms
```

**Cleanup:**

```bash
task chaos:cleanup-networkchaos
task chaos:cleanup-cluster-diagnostics
```

---

### Scenario B — Block Node Latency

**1. Deploy the diagnostics pod:**

```bash
task chaos:deploy-cluster-diagnostics REGION=us
```

**2. Get the block node pod IP:**

```bash
kubectl get pods -n $SOLO_NAMESPACE \
  -l solo.hedera.com/type=block-node \
  -o jsonpath='{.items[0].status.podIP}'
```

**3. Exec in and measure baseline latency:**

```bash
task chaos:exec-cluster-diagnostics
# inside the pod:
ping <block-node-pod-IP>
```

Expected baseline:

```
64 bytes from 10.244.0.44: icmp_seq=1 ttl=63 time=0.03 ms
64 bytes from 10.244.0.44: icmp_seq=2 ttl=63 time=0.04 ms
```

**4. From a second terminal, apply block node latency:**

```bash
task chaos:block-node:network-netem
```

**5. Latency jumps to ~100ms** (us→ap rule, 100ms one-way / 200ms RTT):

```
64 bytes from 10.244.0.44: icmp_seq=50 ttl=63 time=99.0 ms
64 bytes from 10.244.0.44: icmp_seq=51 ttl=63 time=103 ms
```

**Cleanup:**

```bash
task chaos:cleanup-networkchaos
task chaos:cleanup-cluster-diagnostics
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

| Guide                                           | What it covers                                                    |
|-------------------------------------------------|-------------------------------------------------------------------|
| [`docs/consensus-node.md`](consensus-node.md)   | Full walkthroughs: latency simulation and pod kill with live load |
| [`docs/block-node.md`](block-node.md)           | Block node latency verification with baseline measurements        |
| [`docs/developer-guide.md`](developer-guide.md) | How to contribute new experiments or add a new component          |
