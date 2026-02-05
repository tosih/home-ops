# New Applications Deployment Steps

This document outlines the deployment steps for the newly added applications:
- **Volsync** - PVC backup and replication operator
- **PostgreSQL S3 Backups** - Automated backups to Rook-Ceph object storage
- **Linkding** - Bookmark manager application

## Overview of Changes

### 1. Volsync Operator
- **Location**: `kubernetes/apps/kube-system/volsync/`
- **Purpose**: Enables PVC backup and replication using various backends
- **Features**: S3, Restic, Rsync backup methods with scheduling

### 2. PostgreSQL S3 Backups
- **Location**: Updates to `kubernetes/apps/databases/cloudnative-pg/clusters/v17/postgres-cluster.yaml`
- **Backend**: Rook-Ceph object storage (already running in your cluster)
- **Features**: 
  - Daily automated backups at 2 AM
  - 30-day retention policy
  - WAL archiving with gzip compression
  - Parallel backup jobs for performance

### 3. Linkding
- **Location**: `kubernetes/apps/cloud/linkding/`
- **Purpose**: Self-hosted bookmark manager
- **Features**: PostgreSQL backend, URL validation, snapshots
- **Access**: Will be available at `https://bookmarks.tosih.org`

## Deployment Steps

### Step 1: Commit and Push Changes

First, commit all the new manifests to your Git repository:

```bash
git add kubernetes/apps/kube-system/volsync/
git add kubernetes/apps/cloud/linkding/
git add kubernetes/apps/rook-ceph/rook-ceph/cluster/ceph-objectstore-user-postgres.yaml
git add kubernetes/apps/databases/cloudnative-pg/clusters/v17/postgres-cluster.yaml
git commit -m "Add Volsync, PostgreSQL S3 backups, and Linkding"
git push
```

### Step 2: Wait for Flux to Deploy

Flux will automatically deploy the changes. Monitor the deployment:

```bash
# Watch all Flux kustomizations
flux get kustomizations -A --watch

# Check specifically for new apps
kubectl -n kube-system get helmrelease volsync
kubectl -n cloud get helmrelease linkding
```

### Step 3: Configure PostgreSQL S3 Backup Credentials

The PostgreSQL backup requires S3 credentials from the Ceph object store user. After the CephObjectStoreUser is created, run this command:

```bash
# Wait for the CephObjectStoreUser to be ready
kubectl -n rook-ceph wait --for=condition=Ready cephobjectstoreuser/postgres-backup --timeout=300s

# Create the S3 credentials secret in the databases namespace
kubectl -n databases create secret generic postgres-s3-backup \
  --from-literal=ACCESS_KEY_ID=$(kubectl -n rook-ceph get secret rook-ceph-object-user-ceph-objectstore-postgres-backup -o jsonpath='{.data.AccessKey}' | base64 -d) \
  --from-literal=ACCESS_SECRET_KEY=$(kubectl -n rook-ceph get secret rook-ceph-object-user-ceph-objectstore-postgres-backup -o jsonpath='{.data.SecretKey}' | base64 -d)
```

### Step 4: Verify S3 Buckets are Created

The ObjectBucketClaims will automatically create the S3 buckets. Verify they're ready:

```bash
# Check ObjectBucketClaims
kubectl -n rook-ceph get objectbucketclaim

# You should see:
# NAME               STORAGE-CLASS   PHASE   AGE
# postgres-backups   ceph-bucket     Bound   1m
# volsync-backups    ceph-bucket     Bound   1m
```

The buckets `postgres-backups` and `volsync-backups` will be automatically created.

### Step 5: Trigger First PostgreSQL Backup

After the credentials are configured, trigger a manual backup to test:

```bash
# Create a backup
kubectl -n databases create -f - <<EOF
apiVersion: postgresql.cnpg.io/v1
kind: Backup
metadata:
  name: postgres17-manual-backup-$(date +%Y%m%d-%H%M%S)
spec:
  cluster:
    name: postgres17
  method: barmanObjectStore
  target: primary
EOF

# Watch the backup progress
kubectl -n databases get backup --watch
```

### Step 6: Verify Deployments

Check that all applications are running:

```bash
# Check Volsync
kubectl -n kube-system get pods -l app.kubernetes.io/name=volsync

# Check Linkding
kubectl -n cloud get pods -l app.kubernetes.io/name=linkding

# Check PostgreSQL backup job (should show completed)
kubectl -n databases get jobs
```

### Step 7: Access Linkding

1. Linkding will be available at: `https://bookmarks.tosih.org`
2. On first access, you'll need to create a superuser:

```bash
# Create admin user
kubectl -n cloud exec -it deployment/linkding-linkding -- python manage.py createsuperuser
```

Follow the prompts to create your admin account.

## Verification Commands

### Volsync Status
```bash
kubectl -n kube-system logs -l app.kubernetes.io/name=volsync
```

### PostgreSQL Backup Status
```bash
# List all backups
kubectl -n databases get backups

# Check backup details
kubectl -n databases describe backup <backup-name>

# View backup logs
kubectl -n databases logs -l job-name=<backup-job-name>
```

### Linkding Status
```bash
# Check pods
kubectl -n cloud get pods -l app.kubernetes.io/name=linkding

# Check logs
kubectl -n cloud logs -l app.kubernetes.io/name=linkding

# Check database connection
kubectl -n cloud get database linkding
```

## Using Volsync for Application Backups

Volsync is now available for backing up PVCs to your **Rook-Ceph object storage**.

**Where do backups go?**
- **Destination**: Your Rook-Ceph S3-compatible object storage (internal to cluster)
- **Method**: Restic with encryption
- **Bucket**: `volsync-backups` (created during setup)

**Complete setup guide**: See `docs/volsync-setup.md` for:
- Initial setup (one-time)
- How to configure backups for each app
- Backup monitoring
- Restore procedures
- Example configurations

**Quick example**: A Memos backup configuration is already provided in `kubernetes/apps/cloud/memos/backup/` as a reference.

## Troubleshooting

### PostgreSQL Backup Not Starting
1. Check if the secret exists: `kubectl -n databases get secret postgres-s3-backup`
2. Verify Ceph object store is accessible: `kubectl -n rook-ceph get cephobjectstore`
3. Check PostgreSQL cluster logs: `kubectl -n databases logs -l cnpg.io/cluster=postgres17`

### Linkding Not Starting
1. Check database is ready: `kubectl -n cloud get database linkding`
2. Check database secret exists: `kubectl -n cloud get secret database-linkding-user-linkding-user`
3. Check pod logs: `kubectl -n cloud logs -l app.kubernetes.io/name=linkding`

### Volsync Issues
1. Check operator logs: `kubectl -n kube-system logs -l app.kubernetes.io/name=volsync`
2. Verify CRDs are installed: `kubectl get crd | grep volsync`

## Next Steps

1. **Set up Volsync backups** for critical applications (Immich, Memos, etc.)
2. **Monitor PostgreSQL backups** - ensure they're completing successfully
3. **Configure Linkding** - import your existing bookmarks
4. **Add monitoring** - Consider adding Prometheus/Grafana to monitor backup health

## Backup Restoration

### PostgreSQL Restore
To restore from a backup:

```bash
# List available backups
kubectl -n databases get backups

# Restore to a new cluster (recommended)
kubectl apply -f - <<EOF
apiVersion: postgresql.cnpg.io/v1
kind: Cluster
metadata:
  name: postgres17-restored
  namespace: databases
spec:
  instances: 3
  bootstrap:
    recovery:
      source: postgres17
      recoveryTarget:
        targetTime: "2025-01-01 12:00:00"
  # ... rest of cluster spec
EOF
```

Refer to CloudNativePG documentation for detailed recovery procedures.
