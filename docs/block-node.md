# Block Node Chaos Guide

This guide walks through testing network latency between a block node and a US-region cluster diagnostics pod — from baseline measurement through chaos injection and verification.

## What This Scenario Does

- Deploys a block node alongside the Solo network.
- Deploys a cluster diagnostics pod labelled as the `us` region to emulate a US-region consensus node.
- Applies a `NetworkChaos` experiment (`chaos/block-node/network/netem-us-ohio.yml`) that adds ~100ms **one-way** latency (200ms RTT) on traffic from US-region consensus nodes (`solo.hedera.com/region=us`) to the block node (`solo.hedera.com/type=block-node`), simulating the block node being in the AP region.
- Uses a fixed resource name (`solo-chaos-network-netem-block-node-us-to-ap`) so reruns replace the experiment rather than accumulate.

## Full Reproduction Steps

### 1. Bootstrap the environment

```bash
task setup
task deploy-network
task install-chaos-mesh
```

### 2. Deploy the block node

> Requires solo v0.63.0+ (the version pinned in this repo).

```bash
task deploy-block-node
```

This deploys the block node via the Solo CLI, waits for the pod to become Ready, and applies the required chaos labels (`solo.hedera.com/type=block-node`, `solo.hedera.com/region=ap`) automatically.

### 3. Deploy cluster diagnostics in the US region

This emulates a network node in the US region and gives you a pod to measure latency from:

```bash
task chaos:deploy-cluster-diagnostics REGION=us
```

### 4. Exec into the diagnostics pod

```bash
task chaos:exec-cluster-diagnostics
```

### 5. Measure baseline latency to the block node

Inside the diagnostics pod, ping the block node pod IP:

```bash
ping <block-node-pod-IP>
```

With no chaos active, latency will be near zero:

```
64 bytes from 10.244.0.44: icmp_seq=1659 ttl=63 time=0.03 ms
64 bytes from 10.244.0.44: icmp_seq=1660 ttl=63 time=0.04 ms
64 bytes from 10.244.0.44: icmp_seq=1661 ttl=63 time=0.05 ms
64 bytes from 10.244.0.44: icmp_seq=1662 ttl=63 time=0.02 ms
```

### 6. Introduce network latency

From a separate terminal (keep the ping running in the diagnostics pod):

```bash
task chaos:block-node:network-netem
```

### 7. Verify latency has increased

Back in the diagnostics pod, the ping output should show ~100ms latency:

```
64 bytes from 10.244.0.44: icmp_seq=2988 ttl=63 time=99.0 ms
64 bytes from 10.244.0.44: icmp_seq=2989 ttl=63 time=98.1 ms
64 bytes from 10.244.0.44: icmp_seq=2990 ttl=63 time=93.0 ms
64 bytes from 10.244.0.44: icmp_seq=2991 ttl=63 time=115 ms
64 bytes from 10.244.0.44: icmp_seq=2992 ttl=63 time=103 ms
```

This confirms the `us → ap` latency rule is in effect (100ms one-way, 200ms RTT between US and AP regions).

## Check Experiment Status

```bash
task chaos:show-experiment-status NAME=solo-chaos-network-netem-block-node-us-to-ap TYPE=NetworkChaos
```

Or inspect all active experiments:

```bash
kubectl get networkchaos -n chaos-mesh
```

## Cleanup

```bash
task chaos:cleanup-networkchaos
task chaos:cleanup-cluster-diagnostics
```

From inside `chaos/`:

```bash
cd chaos
task cleanup-networkchaos
task cleanup-cluster-diagnostics
```

## Related Docs

- Main operational guide: `docs/README.md`
- Block node task definitions: `chaos/block-node/Taskfile.yml`
- Block-node netem manifest: `chaos/block-node/network/netem-us-ohio.yml`
- Environment validation: `chaos/validate-env.sh`
