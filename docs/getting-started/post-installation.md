# Post Installation

After your cluster is up and running, there are several important configuration steps and optional enhancements to consider.

## Essential Post-Installation Tasks

### Verify Cluster Health

```bash
# Check all nodes are Ready
kubectl get nodes

# Check Cilium status
cilium status

# Verify Flux is syncing
flux check
flux get sources git flux-system
flux get ks -A
flux get hr -A
```

### Test Network Connectivity

```bash
# Check gateway connectivity (replace with your actual IPs)
nmap -Pn -n -p 443 10.0.50.101 10.0.50.102 -vv

# Test DNS resolution (replace with your domain)
dig @10.0.50.100 echo.tosih.org
```

### Verify TLS Certificates

```bash
kubectl -n kube-system describe certificates
```

## Configure GitHub Webhook

By default, Flux checks Git every hour. For instant updates on `git push`, configure a webhook:

**1. Get the webhook path:**

```bash
kubectl -n flux-system get receiver github-webhook \
  --output=jsonpath='{.status.webhookPath}'
```

The path will look like: `/hook/12ebd1e363c641dc3c2e430ecf3cee2b3c7a5ac9e1234506f6f5f3ce1230e123`

**2. Construct the full webhook URL:**

```
https://flux-webhook.tosih.org/hook/12ebd1e363c641dc3c2e430ecf3cee2b3c7a5ac9e1234506f6f5f3ce1230e123
```

**3. Add to GitHub:**

1. Go to your repository settings: `https://github.com/tosih/home-ops/settings/hooks`
2. Click **Add webhook**
3. **Payload URL**: Enter the URL from step 2
4. **Content type**: `application/json`
5. **Secret**: Paste content from `github-push-token.txt`
6. **Events**: Select "Just the push event"
7. Click **Add webhook**

!!! tip "Test the webhook"
    After adding, push a commit to test. The webhook should trigger immediately.

## Enable Renovate

[Renovate](https://www.mend.io/renovate) automatically creates PRs for dependency updates (Helm charts, container images, etc.).

**Setup:**

1. Visit https://github.com/apps/renovate
2. Click **Configure**
3. Select your `home-ops` repository
4. Renovate will create a "Dependency Dashboard" issue

**Configuration:**

The base config is in [.renovaterc.json5](https://github.com/tosih/home-ops/blob/main/.renovaterc.json5):

- Runs on weekends by default
- Groups related updates
- Auto-merges minor/patch updates (configurable)

!!! info "Dependency Dashboard"
    Renovate creates an issue that shows all pending updates with interactive checkboxes for manual control.

## Configure DNS Resolution

### Public DNS (External Access)

The `external-dns` application handles public DNS records for services using the `external` gateway.

**Default public services:**
- `echo.tosih.org` - Test service
- `flux-webhook.tosih.org` - GitHub webhook endpoint

**To make additional apps public:**

Use the `external` gateway in your HTTPRoute:

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: my-app
spec:
  parentRefs:
    - name: external  # Use external gateway
      namespace: kube-system
  hostnames:
    - "my-app.tosih.org"
  rules:
    - backendRefs:
        - name: my-app
          port: 80
```

### Home DNS (Internal Access)

For services accessible only on your home network, configure split DNS:

**1. Configure your home DNS server** (Pi-hole, AdGuard, router, etc.)

Forward `*.tosih.org` queries to the k8s-gateway IP: `10.0.50.100`

**Example for Pi-hole:**

```
# In /etc/dnsmasq.d/02-k8s-gateway.conf
server=/tosih.org/10.0.50.100
```

**Example for pfSense/OPNsense:**

- Services → DNS Resolver → General Settings
- Add Domain Override: `tosih.org` → `10.0.50.100`

**2. Use the `internal` gateway** in your HTTPRoute:

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: my-internal-app
spec:
  parentRefs:
    - name: internal  # Internal only
      namespace: kube-system
  hostnames:
    - "my-app.tosih.org"
  rules:
    - backendRefs:
        - name: my-app
          port: 80
```

!!! warning "DNS troubleshooting"
    If DNS isn't working, check [this discussion](https://github.com/onedr0p/cluster-template/discussions/719) for common issues.

## Clean Up Template Files

Once you're comfortable with your setup and no longer need to run `task configure`, clean up template-related files:

```bash
task template:tidy
```

This archives:
- `templates/` directory
- `makejinja.toml`
- Template-related taskfiles

And removes "duplicate registry" warnings from Renovate.

```bash
git add -A
git commit -m "chore: tidy up template files"
git push
```

## Optional Enhancements

### Storage Solutions

If you need persistent storage beyond Ceph, consider:

- **[Longhorn](https://longhorn.io/)** - Cloud-native distributed storage
- **[OpenEBS](https://openebs.io/)** - Container-native storage
- **[Democratic CSI](https://github.com/democratic-csi/democratic-csi)** - NFS/iSCSI support
- **[Synology CSI](https://github.com/SynologyOpenSource/synology-csi)** - For Synology NAS

### Alternative DNS Providers

Instead of k8s_gateway, you can use external-dns with:

- **[Pi-hole](https://github.com/kubernetes-sigs/external-dns/blob/master/docs/tutorials/pihole.md)**
- **[UniFi](https://github.com/kashalls/external-dns-unifi-webhook)**
- **[AdGuard Home](https://github.com/muhlba91/external-dns-provider-adguard)**
- **[Bind](https://github.com/kubernetes-sigs/external-dns/blob/master/docs/tutorials/rfc2136.md)**

### Secret Management Alternatives

While SOPS works well, consider [External Secrets](https://external-secrets.io/) for:

- Centralized secret storage
- Integration with cloud providers (AWS Secrets Manager, etc.)
- Self-hosted Vault support
- Easier secret rotation

### Monitoring & Observability

Add monitoring with:

- **Prometheus** - Metrics collection
- **Grafana** - Dashboards and visualization
- **Loki** - Log aggregation
- **Hubble** - Cilium network observability

### Backup Solutions

Implement backups with:

- **[Velero](https://velero.io/)** - Kubernetes backup and restore
- **[K8up](https://k8up.io/)** - Kubernetes backup operator
- **Restic** - For Ceph volume backups

## Maintenance Tasks

### Update Talos Configuration

```bash
# Edit talconfig.yaml or patches
vim talos/talconfig.yaml

# Regenerate configs
task talos:generate-config

# Apply to specific node
task talos:apply-node IP=10.0.50.10 MODE=auto
```

### Upgrade Talos Version

```bash
# Edit version in talenv.yaml
vim talos/talenv.yaml

# Upgrade node (one at a time)
task talos:upgrade-node IP=10.0.50.10
```

!!! tip "Safe Rolling Upgrades"
    Always upgrade one node at a time and verify cluster health between upgrades. For worker nodes, ensure workloads are rescheduled properly before proceeding to the next node.

### Upgrade Kubernetes

```bash
# Edit version in talenv.yaml
vim talos/talenv.yaml

# Upgrade cluster
task talos:upgrade-k8s
```

### Force Flux Sync

```bash
# Quick sync
task reconcile

# Or manually
flux reconcile source git flux-system
flux reconcile kustomization flux-system --with-source
```

## Debugging Common Issues

### Pods Not Starting

```bash
# Check node resources
kubectl describe node <node-name>

# Check pod events
kubectl describe pod <pod-name> -n <namespace>

# Check logs
kubectl logs <pod-name> -n <namespace> -f
```

### Flux Not Syncing

```bash
# Check Flux status
flux check

# View logs
kubectl logs -n flux-system -l app=source-controller --tail=50
kubectl logs -n flux-system -l app=kustomize-controller --tail=50
kubectl logs -n flux-system -l app=helm-controller --tail=50

# Force reconcile
flux reconcile kustomization flux-system --with-source
```

### Certificate Issues

```bash
# Check cert-manager
kubectl get certificates -A
kubectl describe certificate <name> -n <namespace>

# Check cert-manager logs
kubectl logs -n cert-manager deploy/cert-manager
```

### Network Issues

```bash
# Check Cilium
cilium status
cilium connectivity test

# Check endpoints
kubectl get endpoints -n <namespace>

# Check services
kubectl get svc -A
```

## Getting Help

### Community Support

- **GitHub Discussions**: [onedr0p/cluster-template](https://github.com/onedr0p/cluster-template/discussions)
- **Discord**: [Home Operations](https://discord.gg/home-operations) - `#support` or `#cluster-template` channels

### Related Projects

If you're looking for alternatives or inspiration:

- [khuedoan/homelab](https://github.com/khuedoan/homelab) - Fully automated homelab
- [ricsanfre/pi-cluster](https://github.com/ricsanfre/pi-cluster) - Pi-based cluster
- [techno-tim/k3s-ansible](https://github.com/techno-tim/k3s-ansible) - k3s with Ansible

## Next Steps

Now that your cluster is configured:

1. ✅ Deploy your first application
2. ✅ Set up monitoring and alerting
3. ✅ Configure automated backups
4. ✅ Explore the community repos at [Kubesearch](https://kubesearch.dev)

!!! tip "Learn by doing"
    The best way to learn is to experiment. Don't be afraid to break things in your homelab - that's what it's for!
