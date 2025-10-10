# Initial Setup

This guide walks through deploying the cluster from scratch.

!!! warning "Starting from scratch"
    This guide assumes you're deploying a new cluster. If you already have a running cluster and want to make changes, see the [Operations](../operations/worker-upgrade-procedure.md) guides instead.

## Stage 1: Repository Setup

### Clone and Configure Repository

```bash
# Clone your repository
git clone https://github.com/tosih/home-ops.git
cd home-ops

# Install tools via mise
mise trust
pip install pipx
mise install
```

### Initialize Configuration Files

```bash
task init
```

This creates:

- `cluster.yaml` - Cluster-wide settings
- `nodes.yaml` - Node-specific configuration
- `age.key` - Encryption key for secrets
- `github-deploy.key` - Git repository deploy key
- `github-push-token.txt` - Webhook token

## Stage 2: Cloudflare Configuration

### Create API Token

1. Go to [Cloudflare API Tokens](https://dash.cloudflare.com/profile/api-tokens)
2. Click **Create Token**
3. Use the **Edit zone DNS** template
4. Add permissions:
   - Zone → DNS → Edit
   - Account → Cloudflare Tunnel → Read
5. Limit to your specific domain
6. Save the token somewhere safe

### Create Cloudflare Tunnel

```bash
cloudflared tunnel login
cloudflared tunnel create --credentials-file cloudflare-tunnel.json kubernetes
```

The `cloudflare-tunnel.json` file will be used during configuration.

## Stage 3: Configure Cluster

### Edit cluster.yaml

Open `cluster.yaml` and configure:

```yaml
# Network configuration
node_cidr: "10.0.0.0/16"
node_default_gateway: "10.0.0.1"
cluster_api_addr: "10.0.50.50"

# DNS and gateway IPs (choose unused IPs in your network)
cluster_dns_gateway_addr: "10.0.50.100"
cluster_gateway_addr: "10.0.50.101"
cloudflare_gateway_addr: "10.0.50.102"

# GitHub repository
repository_name: "tosih/home-ops"

# Cloudflare domain and token
cloudflare_domain: "tosih.org"
cloudflare_token: "your-api-token-here"
```

### Edit nodes.yaml

Configure each node:

```yaml
nodes:
  - name: "control0"
    address: "10.0.50.1"
    controller: true
    disk: "DISK-SERIAL-NUMBER"  # or /dev/sda
    mac_addr: "aa:bb:cc:dd:ee:ff"
    schematic_id: "your-talos-schematic-id"
    mtu: 1500
    secureboot: false
    encrypt_disk: false
```

Repeat for all nodes (control0, control1, worker0, worker1, worker2).

!!! tip "Finding disk serial numbers"
    Boot the node with Talos and run:
    ```bash
    talosctl --nodes <IP> disks
    ```

### Generate Configurations

```bash
task configure
```

This will:

1. ✅ Validate your YAML files
2. ✅ Generate Talos configurations
3. ✅ Generate Kubernetes manifests
4. ✅ Encrypt secrets with SOPS

### Commit to Git

```bash
# Verify all secrets are encrypted
ls -R kubernetes/ | grep sops

git add -A
git commit -m "chore: initial commit :rocket:"
git push
```

!!! warning "Private repository?"
    If using a private repo, add the public key from `github-deploy.key.pub` to your repository's deploy keys in Settings → Deploy keys.

## Stage 4: Bootstrap Talos

### Install Talos on All Nodes

```bash
task bootstrap:talos
```

This will:

1. Generate secrets if not present
2. Apply configuration to all nodes
3. Bootstrap the cluster
4. Generate kubeconfig

!!! info "This may take 10+ minutes"
    You'll see various errors during bootstrap - this is normal while the CNI initializes.

### Verify Talos Installation

```bash
# Check all nodes are responding
talosctl --nodes 10.0.50.1,10.0.50.2,10.0.50.10,10.0.50.11,10.0.50.12 version

# Check cluster health
talosctl health

# Verify nodes (will show NotReady until CNI is installed)
kubectl get nodes
```

### Commit Generated Secrets

```bash
git add talos/talsecret.sops.yaml
git commit -m "chore: add talhelper encrypted secret :lock:"
git push
```

## Stage 5: Bootstrap Applications

### Install Core Components

```bash
task bootstrap:apps
```

This installs:

- ✅ Cilium (CNI)
- ✅ CoreDNS
- ✅ Spegel (image mirror)
- ✅ Flux (GitOps)

### Monitor Deployment

```bash
# Watch all pods come up
kubectl get pods --all-namespaces --watch

# Check Cilium status
cilium status

# Check Flux
flux check
flux get sources git
flux get ks -A
```

## Stage 6: Verification

### Check Cluster Health

```bash
# All nodes should be Ready
kubectl get nodes

# Check Cilium
cilium status

# Verify Flux is syncing
flux get ks -A
flux get hr -A
```

### Test DNS Resolution

```bash
# Test internal DNS (replace with your domain)
dig @10.0.50.100 echo.tosih.org

# Should resolve to your cloudflare gateway IP
```

### Test Ingress

```bash
# Port-forward to echo service
kubectl -n default port-forward svc/echo 8080:8080

# In another terminal
curl http://localhost:8080
```

## Next Steps

Your cluster is now running! Here's what you can do next:

### Essential Setup

1. **Configure GitHub Webhook** for instant Flux sync on push
   ```bash
   # Get webhook path
   kubectl -n flux-system get receiver github-webhook \
     -o jsonpath='{.status.webhookPath}'

   # Configure at: https://github.com/tosih/home-ops/settings/hooks
   ```

2. **Enable Renovate** for automated dependency updates
   - Visit https://github.com/apps/renovate
   - Click Configure and select your repository

### Optional Enhancements

- **Add Monitoring**: Deploy Prometheus & Grafana
- **Add Applications**: Create new apps in `kubernetes/apps/`

### Clean Up Template Files

Once you're comfortable with the setup:

```bash
task template:tidy
```

This archives template-related files you no longer need.

## Troubleshooting

### Pods stuck in Pending

```bash
# Check node resources
kubectl describe nodes

# Check pod events
kubectl describe pod <pod-name> -n <namespace>
```

### Cilium not installing

```bash
# Check Flux HelmRelease
flux get hr -n kube-system

# Check logs
kubectl -n kube-system logs -l app.kubernetes.io/name=cilium
```

### Can't access cluster

```bash
# Verify kubeconfig
cat $KUBECONFIG

# Test connection
kubectl cluster-info

# Check if nodes are reachable
ping 10.0.50.50
```

For more help, see the [cluster-template discussions](https://github.com/onedr0p/cluster-template/discussions).
