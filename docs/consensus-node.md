# Consensus Node Chaos Guide

This guide walks through two end-to-end testing scenarios for consensus nodes:

1. [Network latency simulation](#scenario-1-network-latency-simulation) — verify that cross-region latency rules are applied and observable
2. [Pod kill with live hammer load](#scenario-2-pod-kill-with-live-hammer-load) — verify the network continues processing transactions after a node pod is killed, and how to recover the node

---

## Prerequisites

Bootstrap the environment before running either scenario:

```bash
task setup
task deploy-network
task install-chaos-mesh
```

---

## Scenario 1: Network Latency Simulation

This scenario deploys a diagnostics pod in the `us` region, measures baseline latency to a consensus node in a different region (`eu`), then applies the global netem rules and confirms the expected latency increase.

Consensus node pods are labeled with `solo.hedera.com/region` (us / eu / ap). The netem experiment adds latency selectively based on those labels — for example, `us → eu` adds 50ms one-way (~100ms RTT).

### Step 1: Deploy the diagnostics pod (baseline only, no chaos yet)

Deploy the diagnostics pod without any chaos experiment running yet:

```bash
task chaos:deploy-cluster-diagnostics REGION=us
```

This only deploys the diagnostics pod — it does not start any chaos experiment, so baseline latency will be near zero.

### Step 2: Get the IP of a consensus node in a different region

```bash
kubectl get pods -n solo -l solo.hedera.com/type=network-node,solo.hedera.com/region=eu \
  -o jsonpath='{.items[0].status.podIP}'
```

### Step 3: Exec into the diagnostics pod

```bash
task chaos:exec-cluster-diagnostics
```

### Step 4: Measure baseline latency

Inside the diagnostics pod, ping the EU-region consensus node IP from the previous step:

```bash
ping <eu-consensus-node-pod-IP>
```

With no chaos active, latency will be near zero:

```
64 bytes from 10.244.0.12: icmp_seq=1 ttl=63 time=0.04 ms
64 bytes from 10.244.0.12: icmp_seq=2 ttl=63 time=0.03 ms
64 bytes from 10.244.0.12: icmp_seq=3 ttl=63 time=0.05 ms
64 bytes from 10.244.0.12: icmp_seq=4 ttl=63 time=0.04 ms
```

### Step 5: Apply global latency rules

From a **separate terminal** (keep the ping running):

```bash
task chaos:consensus-node:network-netem
```

This applies 10 fixed-name `NetworkChaos` resources covering all cross-region paths. The `us → eu` rule adds 50ms one-way latency.

### Step 6: Verify latency has increased

Back in the diagnostics pod, the ping output should now show ~50ms:

```
64 bytes from 10.244.0.12: icmp_seq=120 ttl=63 time=48.2 ms
64 bytes from 10.244.0.12: icmp_seq=121 ttl=63 time=51.7 ms
64 bytes from 10.244.0.12: icmp_seq=122 ttl=63 time=49.4 ms
64 bytes from 10.244.0.12: icmp_seq=123 ttl=63 time=53.1 ms
```

This confirms the `us → eu` latency rule is in effect (50ms one-way / 100ms RTT).

For `us → ap` paths you would expect ~100ms one-way. See the [RTT reference table](README.md#cross-region-rtt-reference) in `docs/README.md` for all region pairs.

### Check experiment status

```bash
task chaos:show-experiment-status NAME=solo-chaos-network-netem-us-to-eu TYPE=NetworkChaos
kubectl get networkchaos -n chaos-mesh
```

### Cleanup

```bash
task chaos:cleanup-networkchaos
task chaos:cleanup-cluster-diagnostics
```

---

## Scenario 2: Pod Kill with Live Hammer Load

This scenario runs the `hammer` load generator to create a continuous stream of transactions against nodes 1–4, then kills node5 to verify the network keeps processing transactions. It also shows how to recover the killed node with `task refresh-node`.

> The hammer job is pre-configured in `dev/k8s/solo-chaos-hammer-job.yml` to target `node1,node2,node3,node4`, deliberately leaving `node5` unused so killing it does not interrupt the load.

### Step 1: Build the hammer image

```bash
task build
task build:image
```

### Step 2: Deploy the hammer job

```bash
task deploy-hammer-job
```

### Step 3: Watch transactions flowing

```bash
kubectl logs -n solo -l app=solo-chaos-hammer -f
```

You should see steady transaction receipts and a live TPS count:

```
{"level":"info","total_tx":42,"total_time_sec":4.1,"tps":10,"message":"Received a transaction receipt"}
{"level":"info","total_tx":43,"total_time_sec":4.2,"tps":10,"message":"Received a transaction receipt"}
```

### Step 4: Kill node5

From a **separate terminal** (keep the log stream open):

```bash
task chaos:consensus-node:pod-kill NODE_NAMES=node5
```

### Step 5: Observe network resilience

In the log terminal, transactions should continue flowing uninterrupted — the hammer only targets nodes 1–4 and the remaining consensus nodes maintain the network:

```
{"level":"info","total_tx":105,"total_time_sec":10.3,"tps":10,"message":"Received a transaction receipt"}
{"level":"info","total_tx":106,"total_time_sec":10.4,"tps":10,"message":"Received a transaction receipt"}
```

Check pod status — node5 will show `Terminating` or disappear, while nodes 1–4 stay `Running`:

```bash
kubectl get pods -n solo
```

### Step 6: Recover the killed node

After a pod kill, node5 may not automatically recover to a healthy consensus state. Use `refresh-node` to run `solo node setup` and `solo node start` for that node:

```bash
task refresh-node NODE=node5
```

> Node recovery takes a few minutes. The node must re-sync state before it can participate in consensus again.

### Step 7: Verify node5 is back

```bash
kubectl get pods -n solo
```

All five nodes should be `Running`:

```
network-node1-0   1/1   Running   0   15m
network-node2-0   1/1   Running   0   15m
network-node3-0   1/1   Running   0   15m
network-node4-0   1/1   Running   0   15m
network-node5-0   1/1   Running   0   2m
```

### Cleanup

```bash
task destroy-hammer-job
```

---

## Related Docs

- Main docs index: `docs/README.md`
- Block-node latency scenario: `docs/block-node.md`
- Consensus node task definitions: `chaos/consensus-node/Taskfile.yml`
- Netem manifests: `chaos/consensus-node/network/netem-*.yml`
- Hammer job manifest: `dev/k8s/solo-chaos-hammer-job.yml`

