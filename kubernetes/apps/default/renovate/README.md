# Renovate Self-Hosted

Self-hosted Renovate bot running as a Kubernetes CronJob.

## Setup Instructions

### 1. Create GitHub Personal Access Token

1. Go to https://github.com/settings/tokens/new
2. Create a token with these permissions:
   - `repo` (all scopes)
   - `workflow`
3. Copy the token

### 2. Encrypt the Secret

```bash
# Edit the secret file and replace the placeholder with your token
nano kubernetes/apps/default/renovate/app/secret.sops.yaml

# Encrypt it with SOPS
sops --encrypt --in-place kubernetes/apps/default/renovate/app/secret.sops.yaml
```

### 3. Commit and Push

```bash
git add kubernetes/apps/default/renovate
git commit -m "feat: add self-hosted Renovate"
git push
```

### 4. Verify Deployment

```bash
# Check if the CronJob was created
kubectl get cronjob -n default

# Check job history
kubectl get jobs -n default -l app.kubernetes.io/name=renovate

# View logs from the latest job
kubectl logs -n default -l app.kubernetes.io/name=renovate --tail=100
```

## Configuration

The Renovate configuration is read from your existing `.renovaterc.json5` in the repository root. The CronJob runs every hour by default.

### Adjust Schedule

Edit `helm-release.yaml` and change the `cronjob.schedule` value:

```yaml
cronjob:
  schedule: "0 */6 * * *"  # Every 6 hours
```

### Manual Trigger

```bash
kubectl create job -n default --from=cronjob/renovate renovate-manual-$(date +%s)
```

## Troubleshooting

### Check logs
```bash
kubectl logs -n default -l app.kubernetes.io/name=renovate --tail=200 -f
```

### Verify secret
```bash
kubectl get secret -n default renovate-secret -o yaml
```

### Debug with dry-run

Edit the HelmRelease and add to renovate config:
```yaml
renovate:
  config: |
    {
      "dryRun": "full"
    }
```
