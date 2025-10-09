# Rook-Ceph Setup Guide

## Overview

This document describes the rook-ceph configuration that has been set up for this cluster. The setup uses custom disk partitioning on worker nodes to create dedicated storage for Ceph OSDs.

## Disk Configuration

### Worker Node Disk Layout

Each worker node (worker0, worker1, worker2) has been configured with the following partition layout on `/dev/nvme0n1`:

| Partition | Size | Mount Point | Purpose |
|-----------|------|-------------|---------|
| nvme0n1p1 | 2.2 GB | /boot/efi | EFI boot partition |
| nvme0n1p2 | 1.0 MB | - | Talos META partition |
| nvme0n1p3 | 110 MB | /var | Talos STATE partition |
| nvme0n1p4 | 150 GB | /var/lib/containerd | Container runtime storage (EPHEMERAL) |
| nvme0n1p5 | 100 GB | /var/lib/rook | **Dedicated Ceph storage** (NEW) |

The new partition (`nvme0n1p5`) is a dedicated 100GB partition for rook-ceph storage, separate from the EPHEMERAL partition.

## Important Notes

### ⚠️ Worker Node Reinstallation Required

**The disk partitioning changes require reinstallation of Talos on the worker nodes.** The existing nodes cannot be repartitioned in-place.

See [WORKER_UPGRADE_PROCEDURE.md](../operations/worker-upgrade-procedure.md) for safe, rolling upgrade instructions that won't disrupt your cluster.

### Configuration Files

The following files have been created/modified:

1. **Talos Configuration:**
   - `/talos/patches/worker/machine-disks.yaml` - Custom disk partitioning patch for worker nodes
   - `/talos/talconfig.yaml` - Updated to include worker patches

2. **Rook-Ceph Operator:**
   - `/kubernetes/apps/storage/rook-ceph/operator/ks.yaml`
   - `/kubernetes/apps/storage/rook-ceph/operator/app/namespace.yaml`
   - `/kubernetes/apps/storage/rook-ceph/operator/app/oci-repository.yaml`
   - `/kubernetes/apps/storage/rook-ceph/operator/app/helm-release.yaml`
   - `/kubernetes/apps/storage/rook-ceph/operator/app/kustomization.yaml`

3. **Rook-Ceph Cluster:**
   - `/kubernetes/apps/storage/rook-ceph/cluster/ks.yaml`
   - `/kubernetes/apps/storage/rook-ceph/cluster/app/ceph-cluster.yaml` - Uses `/dev/nvme0n1p5` on each worker
   - `/kubernetes/apps/storage/rook-ceph/cluster/app/ceph-block-pool.yaml` - 3-way replication
   - `/kubernetes/apps/storage/rook-ceph/cluster/app/storage-class.yaml` - RBD storage class
   - `/kubernetes/apps/storage/rook-ceph/cluster/app/ceph-filesystem.yaml` - CephFS configuration
   - `/kubernetes/apps/storage/rook-ceph/cluster/app/storage-class-filesystem.yaml` - CephFS storage class
   - `/kubernetes/apps/storage/rook-ceph/cluster/app/kustomization.yaml`

4. **Storage Namespace:**
   - `/kubernetes/apps/storage/kustomization.yaml` - Registers rook-ceph with Flux

## Ceph Configuration

### Cluster Specifications

- **Ceph Version:** v19.2.0 (Squid)
- **Monitors:** 3 replicas (1 per worker node)
- **Managers:** 2 replicas
- **OSDs:** 1 per worker node (using `/dev/nvme0n1p5`)
- **Data Replication:** 3 replicas (min 2 for safety)
- **Failure Domain:** host-level
- **Compression:** Aggressive mode enabled

### Storage Classes

Two storage classes are provided:

1. **ceph-block** - RBD (Block) storage
   - Provisioner: `rook-ceph.rbd.csi.ceph.com`
   - Format: ext4
   - Supports volume expansion
   - Reclaim policy: Delete

2. **ceph-filesystem** - CephFS (Shared filesystem) storage
   - Provisioner: `rook-ceph.cephfs.csi.ceph.com`
   - MDS: 1 active + 1 standby
   - Supports volume expansion
   - Reclaim policy: Delete

### Resource Allocation

- **MON:** 500m CPU, 1-2Gi memory
- **OSD:** 1000m CPU, 2-4Gi memory
- **MGR:** 500m CPU, 512Mi-1Gi memory
- **MDS:** 500m CPU, 1-2Gi memory

## Deployment Steps

### 1. Apply Talos Changes

Follow the [Worker Upgrade Procedure](../operations/worker-upgrade-procedure.md) to safely upgrade each worker node with the new disk layout.

### 2. Deploy Rook-Ceph

After all workers are upgraded, commit and push the configuration:

```bash
git add -A
git commit -m "feat: add rook-ceph with dedicated storage partitions"
git push

# Flux will automatically deploy rook-ceph from git
# Monitor the deployment:
flux get ks -A
flux get hr -A

# Check rook-ceph operator
kubectl -n rook-ceph get pods

# Check Ceph cluster status (after cluster is deployed)
kubectl -n rook-ceph get cephcluster
```

### 3. Verify Ceph Health

```bash
# Get a shell in the rook-ceph toolbox
kubectl -n rook-ceph exec -it deployment/rook-ceph-tools -- bash

# Inside the toolbox:
ceph status
ceph osd status
ceph df
```

### 4. Use Ceph Storage

Create a PVC using one of the storage classes:

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: test-pvc
  namespace: default
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: ceph-block
  resources:
    requests:
      storage: 10Gi
```

## Monitoring

The rook-ceph operator includes monitoring support. Metrics are exposed for Prometheus scraping.

## Dashboard

Ceph dashboard is enabled (HTTP, not HTTPS). To access it:

```bash
# Port-forward to the dashboard
kubectl -n rook-ceph port-forward svc/rook-ceph-mgr-dashboard 8443:8443

# Get dashboard password
kubectl -n rook-ceph get secret rook-ceph-dashboard-password \
  -o jsonpath="{['data']['password']}" | base64 --decode
```

## Troubleshooting

### OSDs not starting

```bash
# Check OSD pod logs
kubectl -n rook-ceph logs -l app=rook-ceph-osd

# Verify the partition exists and is accessible
talosctl --nodes 10.0.50.10 get discoveredvolumes | grep nvme0n1p5
```

### Ceph cluster stuck in HEALTH_WARN

```bash
# Get detailed status
kubectl -n rook-ceph exec -it deployment/rook-ceph-tools -- ceph -s

# Common issues:
# - Not enough OSDs (need at least 3)
# - Clock skew between nodes
# - Network issues between nodes
```

### Re-creating the cluster

If you need to completely wipe Ceph data:

```bash
# Delete the Ceph cluster (but not operator)
kubectl -n rook-ceph delete cephcluster rook-ceph

# On each worker node, wipe the Ceph partition:
talosctl --nodes 10.0.50.10 reset --graceful=false --reboot \
  --system-labels-to-wipe STATE --system-labels-to-wipe EPHEMERAL

# Then re-apply the Ceph cluster
flux reconcile kustomization rook-ceph-cluster
```

## References

- [Rook Documentation](https://rook.io/docs/rook/latest/)
- [Ceph Documentation](https://docs.ceph.com/)
- [Talos Disk Configuration](https://www.talos.dev/latest/reference/configuration/#machinedisk)
