# Safe Worker Node Upgrade Procedure

## Overview

This procedure allows you to upgrade worker nodes one at a time to apply the new disk partitioning for rook-ceph **without disrupting your running cluster**. The control plane nodes remain untouched.

## Prerequisites

- Kubernetes cluster is healthy
- Control plane nodes (control0, control1) are running normally
- You have physical or remote access to each worker node
- Workloads can tolerate one node being down at a time

## Important Notes

⚠️ **This is a destructive operation for each worker node** - all data on the worker will be wiped
⚠️ **Only affects worker nodes** - control plane remains running
⚠️ **Process one worker at a time** - ensures cluster availability

## Step-by-Step Procedure

### Pre-Flight Checks

```bash
# Verify cluster health
kubectl get nodes
kubectl get pods -A | grep -v Running | grep -v Completed

# Verify you have 3 healthy workers
talosctl --nodes 10.0.50.10,10.0.50.11,10.0.50.12 version

# Check what's running on each worker
kubectl get pods -A -o wide | grep worker0
kubectl get pods -A -o wide | grep worker1
kubectl get pods -A -o wide | grep worker2
```

### For Each Worker Node (Repeat 3 times)

#### Worker 0 (10.0.50.10)

**Step 1: Cordon the node**
```bash
kubectl cordon worker0
```

**Step 2: Drain the node** (gracefully evict all pods)
```bash
kubectl drain worker0 \
  --ignore-daemonsets \
  --delete-emptydir-data \
  --force \
  --grace-period=300 \
  --timeout=600s
```

**Step 3: Verify pods migrated**
```bash
# Should show no pods (except DaemonSets)
kubectl get pods -A -o wide | grep worker0
```

**Step 4: Reset ONLY this worker node**
```bash
talosctl --nodes 10.0.50.10 reset \
  --graceful=false \
  --reboot \
  --system-labels-to-wipe STATE \
  --system-labels-to-wipe EPHEMERAL
```

**Step 5: Wait for node to reboot into maintenance mode**
```bash
# This may take 2-5 minutes
# Check when it's responding to ping
ping 10.0.50.10
```

**Step 6: Apply the new configuration with disk partitioning**
```bash
cd talos
talhelper gencommand apply \
  --node 10.0.50.10 \
  --extra-flags="--insecure" \
  | bash
```

**Step 7: Wait for node to join cluster**
```bash
# Watch node status (wait for Ready)
watch kubectl get nodes

# Alternative: use this to wait
kubectl wait --for=condition=Ready node/worker0 --timeout=600s
```

**Step 8: Verify the new disk layout**
```bash
talosctl --nodes 10.0.50.10 get discoveredvolumes | grep nvme0n1
# You should see nvme0n1p5 now (the new 100GB partition)

# Verify mount
talosctl --nodes 10.0.50.10 df | grep rook
```

**Step 9: Uncordon the node**
```bash
kubectl uncordon worker0
```

**Step 10: Verify cluster health**
```bash
kubectl get nodes
kubectl get pods -A | grep -v Running | grep -v Completed

# Wait for pods to redistribute
sleep 60
```

#### Worker 1 (10.0.50.11)

Repeat the same steps for worker1:

```bash
# Step 1: Cordon
kubectl cordon worker1

# Step 2: Drain
kubectl drain worker1 \
  --ignore-daemonsets \
  --delete-emptydir-data \
  --force \
  --grace-period=300 \
  --timeout=600s

# Step 3: Verify
kubectl get pods -A -o wide | grep worker1

# Step 4: Reset
talosctl --nodes 10.0.50.11 reset \
  --graceful=false \
  --reboot \
  --system-labels-to-wipe STATE \
  --system-labels-to-wipe EPHEMERAL

# Step 5: Wait for reboot
ping 10.0.50.11

# Step 6: Apply config
cd talos
talhelper gencommand apply \
  --node 10.0.50.11 \
  --extra-flags="--insecure" \
  | bash

# Step 7: Wait for Ready
kubectl wait --for=condition=Ready node/worker1 --timeout=600s

# Step 8: Verify disk
talosctl --nodes 10.0.50.11 get discoveredvolumes | grep nvme0n1

# Step 9: Uncordon
kubectl uncordon worker1

# Step 10: Health check
kubectl get nodes
sleep 60
```

#### Worker 2 (10.0.50.12)

Repeat the same steps for worker2:

```bash
# Step 1: Cordon
kubectl cordon worker2

# Step 2: Drain
kubectl drain worker2 \
  --ignore-daemonsets \
  --delete-emptydir-data \
  --force \
  --grace-period=300 \
  --timeout=600s

# Step 3: Verify
kubectl get pods -A -o wide | grep worker2

# Step 4: Reset
talosctl --nodes 10.0.50.12 reset \
  --graceful=false \
  --reboot \
  --system-labels-to-wipe STATE \
  --system-labels-to-wipe EPHEMERAL

# Step 5: Wait for reboot
ping 10.0.50.12

# Step 6: Apply config
cd talos
talhelper gencommand apply \
  --node 10.0.50.12 \
  --extra-flags="--insecure" \
  | bash

# Step 7: Wait for Ready
kubectl wait --for=condition=Ready node/worker2 --timeout=600s

# Step 8: Verify disk
talosctl --nodes 10.0.50.12 get discoveredvolumes | grep nvme0n1

# Step 9: Uncordon
kubectl uncordon worker2

# Step 10: Health check
kubectl get nodes
```

### Post-Upgrade Verification

**Verify all workers have the new disk layout:**
```bash
for node in 10.0.50.10 10.0.50.11 10.0.50.12; do
  echo "=== Node $node ==="
  talosctl --nodes $node get discoveredvolumes | grep nvme0n1p5
done
```

**Verify all nodes are Ready:**
```bash
kubectl get nodes
# All 5 nodes (2 control + 3 worker) should be Ready
```

**Commit the configuration:**
```bash
git add -A
git commit -m "feat: add rook-ceph with dedicated storage partitions on workers"
git push
```

## Deploy Rook-Ceph

After all workers are upgraded, Flux will automatically deploy rook-ceph:

```bash
# Force Flux reconciliation
task reconcile

# Watch rook-ceph deployment
watch kubectl get pods -n rook-ceph

# Monitor Flux HelmReleases
flux get hr -A

# Check Ceph cluster formation (may take 5-10 minutes)
kubectl -n rook-ceph get cephcluster
kubectl -n rook-ceph get cephblockpool
```

## Troubleshooting

### Node stuck in "NotReady" after config apply

```bash
# Check node logs
talosctl --nodes <NODE_IP> dmesg | tail -50

# Check kubelet status
talosctl --nodes <NODE_IP> service kubelet status

# If needed, reboot manually
talosctl --nodes <NODE_IP> reboot
```

### Drain hangs or times out

```bash
# Identify stuck pods
kubectl get pods -A -o wide | grep <NODE_NAME>

# Force delete stuck pods (last resort)
kubectl delete pod <POD_NAME> -n <NAMESPACE> --grace-period=0 --force

# Then re-run drain
kubectl drain <NODE_NAME> --ignore-daemonsets --delete-emptydir-data --force
```

### Node can't rejoin cluster

```bash
# Check if node can reach control plane
talosctl --nodes <NODE_IP> get members

# Verify network connectivity
talosctl --nodes <NODE_IP> get links

# Check certificates
talosctl --nodes <NODE_IP> get certificates

# Re-apply config if needed
cd talos
talhelper gencommand apply --node <NODE_IP> --extra-flags="--insecure" | bash
```

### Partition not created correctly

```bash
# Check actual disk layout
talosctl --nodes <NODE_IP> disks

# Check discovered volumes
talosctl --nodes <NODE_IP> get discoveredvolumes

# If partition is missing, the node needs to be reset again:
talosctl --nodes <NODE_IP> reset --graceful=false --reboot \
  --system-labels-to-wipe STATE --system-labels-to-wipe EPHEMERAL
```

## Rollback Procedure

If you need to rollback a worker before completing all three:

1. **Remove the worker patch from talconfig.yaml:**
```bash
# Edit talos/talconfig.yaml and remove:
# worker:
#   patches:
#     - "@./patches/worker/machine-disks.yaml"
```

2. **Regenerate configs:**
```bash
task talos:generate-config
```

3. **Reset and re-apply the node:**
```bash
talosctl --nodes <NODE_IP> reset --graceful=false --reboot
# Wait for maintenance mode
talhelper gencommand apply --node <NODE_IP> --extra-flags="--insecure" | bash
```

## Timeline Estimate

- **Per worker node:** ~10-15 minutes
  - Drain: 2-5 minutes
  - Reset & reboot: 2-3 minutes
  - Config apply: 1-2 minutes
  - Node Ready: 2-3 minutes
  - Pod rescheduling: 2-3 minutes

- **Total for all 3 workers:** ~30-45 minutes

## Safety Considerations

✅ **Safe:**
- Control plane remains fully operational
- 2 out of 3 workers available during each upgrade
- Workloads automatically reschedule to healthy nodes
- Can pause between workers to monitor stability

⚠️ **Risks:**
- Workloads without multi-replica deployments may experience downtime
- Pods using local storage (emptyDir, hostPath) will be terminated
- StatefulSets may be disrupted if they don't have pod anti-affinity

🔧 **Recommendations:**
- Perform during maintenance window if possible
- Ensure critical workloads have multiple replicas
- Monitor cluster closely during each worker upgrade
- Wait and verify stability before proceeding to next worker
