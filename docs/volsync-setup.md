# Volsync Setup Guide

Volsync is installed in your cluster and provides PVC backup and replication capabilities. This guide explains how to configure backups for your applications.

## Overview

**What is Volsync?**
Volsync backs up Kubernetes PersistentVolumeClaims (PVCs) to various destinations using different methods.

**Backup Destination:**
Your Volsync backups go to **Rook-Ceph object storage** (S3-compatible) that's already running in your cluster.

## Quick Start

### 1. Setup (One-time)

After deploying the new manifests, verify the S3 bucket is created:

```bash
# Wait for the ObjectBucketClaim to be bound
kubectl -n rook-ceph wait --for=jsonpath='{.status.phase}'=Bound objectbucketclaim/volsync-backups --timeout=300s

# Verify the bucket exists
kubectl -n rook-ceph get objectbucketclaim volsync-backups

# The bucket 'volsync-backups' is automatically created by the ObjectBucketClaim
```

### 2. Back Up an Application

To back up an application's PVC (e.g., Memos), follow these steps:

#### a. Create a Restic Secret

Each application needs a secret with:
- S3 endpoint and credentials
- Restic encryption password
- Bucket path for that app

**Example for Memos** (already created in `kubernetes/apps/cloud/memos/backup/`):

```bash
# Generate a secure restic password
RESTIC_PASSWORD=$(openssl rand -base64 32)

# Create the secret
kubectl -n cloud create secret generic memos-restic-secret \
  --from-literal=RESTIC_REPOSITORY="s3:http://rook-ceph-rgw-ceph-objectstore.rook-ceph.svc:80/volsync-backups/memos" \
  --from-literal=RESTIC_PASSWORD="${RESTIC_PASSWORD}" \
  --from-literal=AWS_ACCESS_KEY_ID="$(kubectl -n rook-ceph get secret rook-ceph-object-user-ceph-objectstore-volsync-backup -o jsonpath='{.data.AccessKey}' | base64 -d)" \
  --from-literal=AWS_SECRET_ACCESS_KEY="$(kubectl -n rook-ceph get secret rook-ceph-object-user-ceph-objectstore-volsync-backup -o jsonpath='{.data.SecretKey}' | base64 -d)"

# Save the restic password somewhere safe (for restore operations)
echo "Memos Restic Password: ${RESTIC_PASSWORD}" >> ~/memos-backup-password.txt
```

#### b. Enable Backup in the App Kustomization

Update the app's kustomization to include the backup resources:

```yaml
# kubernetes/apps/cloud/memos/app/kustomization.yaml
---
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - ./database.yaml
  - ./helm-release.yaml
  - ./grpcroute.yaml
  - ./oci-repository.yaml
  - ../backup  # Add this line
```

#### c. Commit and Push

```bash
git add kubernetes/apps/cloud/memos/
git commit -m "Enable Volsync backup for Memos"
git push
```

Flux will deploy the ReplicationSource and start backing up on schedule.

### 3. Monitor Backups

```bash
# Check ReplicationSource status
kubectl -n cloud get replicationsource memos-backup

# View backup details
kubectl -n cloud describe replicationsource memos-backup

# Check recent backup jobs
kubectl -n cloud get jobs -l volsync.backube/replication-source=memos-backup

# View backup logs
kubectl -n cloud logs -l volsync.backube/replication-source=memos-backup
```

### 4. Trigger Manual Backup

```bash
# Trigger an immediate backup (without waiting for schedule)
kubectl -n cloud patch replicationsource memos-backup \
  --type merge \
  -p '{"spec":{"trigger":{"manual":"backup-'$(date +%Y%m%d-%H%M%S)'"}}}'
```

## Backup Configuration Options

### Schedule Format
Uses standard cron format:
- `"0 3 * * *"` - Daily at 3 AM
- `"0 */6 * * *"` - Every 6 hours
- `"0 2 * * 0"` - Weekly on Sunday at 2 AM

### Retention Policy
Configure how many backups to keep:
```yaml
retain:
  hourly: 6      # Last 6 hourly backups
  daily: 7       # Last 7 daily backups
  weekly: 4      # Last 4 weekly backups
  monthly: 6     # Last 6 monthly backups
  yearly: 1      # Last 1 yearly backup
```

### Copy Method
- **Snapshot** (recommended): Uses CSI snapshots for consistency
- **Clone**: Creates a temporary PVC clone
- **Direct**: Backs up while app is running (may be inconsistent)

## Applications to Back Up

Here are the PVCs you should consider backing up:

| Application | PVC Name | Size | Priority | Why |
|-------------|----------|------|----------|-----|
| Memos | `memos` | 5Gi | **High** | Notes and content |
| Immich | `immich-media` | 10Ti | **Critical** | Photos (already on NFS, but extra safety) |
| Syncthing | `syncthing-data` | 100Gi | Medium | Synced files |
| Romm | `romm-config` | 1Gi | Medium | Game library metadata |
| Home Assistant | `home-assistant-config` | ? | High | Home automation config |
| Linkding | `linkding-data` | 5Gi | Medium | Bookmarks |

**Note:** PostgreSQL is already backed up to S3 via CloudNativePG, so you don't need Volsync for database backups.

## Restoring from Backup

To restore a PVC from a Volsync backup:

### 1. Create a ReplicationDestination

```yaml
---
apiVersion: volsync.backube/v1alpha1
kind: ReplicationDestination
metadata:
  name: memos-restore
  namespace: cloud
spec:
  trigger:
    manual: restore-$(date +%Y%m%d-%H%M%S)
  
  restic:
    # Same repository secret used for backup
    repository: memos-restic-secret
    
    # Destination PVC (can be new or existing)
    destinationPVC: memos-restored
    
    # Storage class and size
    storageClassName: ceph-block
    capacity: 5Gi
    accessModes:
      - ReadWriteOnce
    
    # Optional: restore to a specific snapshot
    # restoreAsOf: "2025-01-01T12:00:00Z"
```

### 2. Apply and Wait

```bash
kubectl apply -f restore.yaml

# Monitor restore progress
kubectl -n cloud get replicationdestination memos-restore --watch
```

### 3. Use the Restored PVC

Once complete, you can:
- Mount it to a pod to verify data
- Replace the original PVC (requires downtime)
- Copy data from restored PVC to original

## Example: Adding Backup to Another App

To add Volsync backup to any app (e.g., Syncthing):

```bash
# 1. Create backup directory
mkdir -p kubernetes/apps/cloud/syncthing/backup

# 2. Generate restic password
RESTIC_PASSWORD=$(openssl rand -base64 32)
echo "Syncthing Restic Password: ${RESTIC_PASSWORD}" >> ~/syncthing-backup-password.txt

# 3. Create secret
kubectl -n cloud create secret generic syncthing-restic-secret \
  --from-literal=RESTIC_REPOSITORY="s3:http://rook-ceph-rgw-ceph-objectstore.rook-ceph.svc:80/volsync-backups/syncthing-data" \
  --from-literal=RESTIC_PASSWORD="${RESTIC_PASSWORD}" \
  --from-literal=AWS_ACCESS_KEY_ID="$(kubectl -n rook-ceph get secret rook-ceph-object-user-ceph-objectstore-volsync-backup -o jsonpath='{.data.AccessKey}' | base64 -d)" \
  --from-literal=AWS_SECRET_ACCESS_KEY="$(kubectl -n rook-ceph get secret rook-ceph-object-user-ceph-objectstore-volsync-backup -o jsonpath='{.data.SecretKey}' | base64 -d)" \
  --dry-run=client -o yaml | kubectl apply -f -

# 4. Create ReplicationSource
cat > kubernetes/apps/cloud/syncthing/backup/replication-source.yaml <<EOF
---
apiVersion: volsync.backube/v1alpha1
kind: ReplicationSource
metadata:
  name: syncthing-data-backup
  namespace: cloud
spec:
  sourcePVC: syncthing-data
  trigger:
    schedule: "0 4 * * *"  # 4 AM daily
  restic:
    repository: syncthing-restic-secret
    retain:
      daily: 7
      weekly: 4
      monthly: 3
    copyMethod: Snapshot
    storageClassName: ceph-block
    volumeSnapshotClassName: csi-ceph-blockpool
EOF

# 5. Update kustomization
# Add ../backup to resources in kubernetes/apps/cloud/syncthing/app/kustomization.yaml

# 6. Commit and push
git add kubernetes/apps/cloud/syncthing/backup/
git commit -m "Add Volsync backup for Syncthing"
git push
```

## Monitoring and Alerting

### Check Backup Health

```bash
# List all ReplicationSources
kubectl get replicationsource -A

# Check for failed backups
kubectl get replicationsource -A -o jsonpath='{range .items[?(@.status.lastManualSync=="error")]}{.metadata.namespace}/{.metadata.name}{"\n"}{end}'

# View backup metrics (if Prometheus is installed)
kubectl -n kube-system port-forward svc/volsync-metrics 8080:8080
# Then visit http://localhost:8080/metrics
```

### Common Issues

**Backup fails with "repository does not exist"**
- Initialize the repository manually:
```bash
kubectl -n <namespace> exec -it <pod-with-restic-secret> -- \
  restic -r $RESTIC_REPOSITORY init
```

**Snapshot fails**
- Check if VolumeSnapshotClass exists: `kubectl get volumesnapshotclass`
- Verify CSI driver supports snapshots
- Try `copyMethod: Clone` instead

**Slow backups**
- Increase cache size: `cacheCapacity: 5Gi`
- Use `copyMethod: Snapshot` for consistency without performance impact

## Security Best Practices

1. **Restic passwords**: Store them securely (password manager, 1Password, etc.)
2. **Rotate credentials**: Periodically rotate S3 access keys
3. **Encrypt at rest**: Restic encrypts all backup data
4. **Access control**: Limit who can access backup secrets
5. **Test restores**: Regularly test restoration procedures

## Cost Considerations

Volsync stores backups in your **Rook-Ceph cluster** (not external cloud):
- **Storage**: Uses your Ceph OSDs (same disks as your cluster)
- **Network**: Internal traffic only (no egress costs)
- **Retention**: Adjust retention policies to manage storage usage

Monitor Ceph storage usage:
```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- ceph df
```

## How It Works

When you deploy the manifests, Rook-Ceph automatically:
1. Creates the `CephObjectStoreUser` (volsync-backup) with S3 credentials
2. Creates the `ObjectBucketClaim` which provisions the `volsync-backups` bucket
3. Generates a secret with bucket connection details (not needed for our setup)

The bucket and credentials are managed declaratively via GitOps!

## Next Steps

1. ✅ Deploy the manifests (ObjectBucketClaim and CephObjectStoreUser)
2. ✅ Wait for bucket to be automatically created
3. Configure backup for critical apps:
   - Start with Memos (example already provided)
   - Add Immich, Home Assistant, Linkding
4. Set up monitoring/alerting for backup failures
5. Document and test restore procedures
6. Schedule regular restore tests

## Additional Resources

- [Volsync Documentation](https://volsync.readthedocs.io/)
- [Restic Documentation](https://restic.readthedocs.io/)
- [Your cluster's Ceph dashboard](https://rook.tosih.org)
