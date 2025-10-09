# Tools Reference

This page provides quick reference for the main tools used in this cluster.

## Task - Task Automation

Task replaces Makefiles with a modern YAML-based task runner.

### Common Commands

```bash
# List all available tasks
task --list

# Initialize new cluster config
task init

# Generate configs from templates
task configure

# Bootstrap Talos
task bootstrap:talos

# Bootstrap apps
task bootstrap:apps

# Force Flux reconciliation
task reconcile

# Generate Talos configs
task talos:generate-config

# Apply config to specific node
task talos:apply-node IP=10.0.50.10 MODE=auto

# Upgrade Talos on node
task talos:upgrade-node IP=10.0.50.10

# Upgrade Kubernetes version
task talos:upgrade-k8s

# Archive template files (after setup)
task template:tidy

# Debug cluster resources
task template:debug
```

## Talosctl - Talos Management

### Node Operations

```bash
# Check node version
talosctl --nodes 10.0.50.10 version

# Check node health
talosctl health

# View node logs
talosctl --nodes 10.0.50.10 dmesg

# Check service status
talosctl --nodes 10.0.50.10 service kubelet status

# Reboot node
talosctl --nodes 10.0.50.10 reboot

# Shutdown node
talosctl --nodes 10.0.50.10 shutdown

# Get node configuration
talosctl --nodes 10.0.50.10 get machineconfig
```

### Resource Inspection

```bash
# List disks
talosctl --nodes 10.0.50.10 get disks

# List discovered volumes
talosctl --nodes 10.0.50.10 get discoveredvolumes

# Check network interfaces
talosctl --nodes 10.0.50.10 get links

# Check cluster members
talosctl --nodes 10.0.50.10 get members

# View disk usage
talosctl --nodes 10.0.50.10 df
```

### Configuration Management

```bash
# Apply configuration in auto mode
talosctl --nodes 10.0.50.10 apply-config --file worker0.yaml --mode=auto

# Apply in no-reboot mode
talosctl --nodes 10.0.50.10 apply-config --file worker0.yaml --mode=no-reboot

# Apply in staged mode (reboot later)
talosctl --nodes 10.0.50.10 apply-config --file worker0.yaml --mode=staged

# Upgrade Talos version
talosctl --nodes 10.0.50.10 upgrade \
  --image factory.talos.dev/installer/schematic-id:v1.11.1
```

## Kubectl - Kubernetes Management

### Cluster Info

```bash
# Cluster information
kubectl cluster-info

# Node status
kubectl get nodes
kubectl describe node worker0

# All resources
kubectl get all -A
```

### Pod Management

```bash
# List pods in namespace
kubectl get pods -n kube-system

# Wide output with node info
kubectl get pods -A -o wide

# Watch pod status
kubectl get pods -A --watch

# Pod details
kubectl describe pod <pod-name> -n <namespace>

# Pod logs
kubectl logs <pod-name> -n <namespace>
kubectl logs <pod-name> -n <namespace> --follow
kubectl logs <pod-name> -n <namespace> --previous

# Execute command in pod
kubectl exec -it <pod-name> -n <namespace> -- /bin/sh
```

### Resource Management

```bash
# Apply manifest
kubectl apply -f manifest.yaml

# Delete resource
kubectl delete -f manifest.yaml
kubectl delete pod <pod-name> -n <namespace>

# Edit resource
kubectl edit deployment <name> -n <namespace>

# Scale deployment
kubectl scale deployment <name> -n <namespace> --replicas=3
```

### Node Maintenance

```bash
# Cordon node (mark unschedulable)
kubectl cordon worker0

# Drain node (evict pods)
kubectl drain worker0 \
  --ignore-daemonsets \
  --delete-emptydir-data \
  --force

# Uncordon node
kubectl uncordon worker0
```

### Debugging

```bash
# Check events
kubectl get events -n <namespace> --sort-by='.metadata.creationTimestamp'

# Check resource usage
kubectl top nodes
kubectl top pods -A

# Port forward
kubectl port-forward svc/<service> -n <namespace> 8080:80

# Run debug pod
kubectl run debug --rm -it --image=alpine -- /bin/sh
```

## Flux - GitOps Management

### Status Checks

```bash
# Check Flux components
flux check

# Get Git sources
flux get sources git -A

# Get Kustomizations
flux get ks -A

# Get HelmReleases
flux get hr -A

# Get HelmRepositories
flux get sources helm -A
```

### Reconciliation

```bash
# Force reconcile Git source
flux reconcile source git flux-system

# Force reconcile Kustomization
flux reconcile ks <name> -n <namespace>

# Force reconcile HelmRelease
flux reconcile hr <name> -n <namespace>

# Suspend reconciliation
flux suspend ks <name> -n <namespace>

# Resume reconciliation
flux resume ks <name> -n <namespace>
```

### Debugging

```bash
# Check Kustomization build
flux build kustomization <name> --path ./kubernetes/apps

# Check HelmRelease values
flux get hr <name> -n <namespace> -o yaml

# View logs
kubectl logs -n flux-system -l app=source-controller --follow
kubectl logs -n flux-system -l app=kustomize-controller --follow
kubectl logs -n flux-system -l app=helm-controller --follow
```

## Cilium - Network Management

### Status and Health

```bash
# Check Cilium status
cilium status

# Connectivity test
cilium connectivity test

# Monitor network flows (Hubble)
cilium hubble ui

# Check network policies
cilium policy get
```

### Debugging

```bash
# Check endpoint status
cilium endpoint list

# Check service load balancing
cilium service list

# Monitor network events
cilium monitor

# Check BPF maps
cilium bpf lb list
cilium bpf ct list global
```

## SOPS - Secret Management

### Encryption

```bash
# Encrypt file
sops --encrypt --in-place secret.yaml

# Decrypt file (view only)
sops --decrypt secret.yaml

# Edit encrypted file
sops secret.yaml

# Rotate keys
sops --rotate --in-place secret.yaml
```

### File Status

```bash
# Check if file is encrypted
sops filestatus secret.yaml

# View file metadata
sops --decrypt --extract '["sops"]' secret.yaml
```

## Helm - Package Management

### Chart Management

```bash
# Add repository
helm repo add rook-release https://charts.rook.io/release
helm repo update

# Search charts
helm search repo rook

# Show chart values
helm show values rook-release/rook-ceph

# Install chart
helm install rook-ceph rook-release/rook-ceph \
  --namespace rook-ceph \
  --create-namespace

# Upgrade chart
helm upgrade rook-ceph rook-release/rook-ceph \
  --namespace rook-ceph

# Uninstall chart
helm uninstall rook-ceph --namespace rook-ceph
```

### Release Management

```bash
# List releases
helm list -A

# Release status
helm status <release> -n <namespace>

# Release history
helm history <release> -n <namespace>

# Rollback release
helm rollback <release> <revision> -n <namespace>

# Get values
helm get values <release> -n <namespace>
```

## Talhelper - Talos Config Generator

### Configuration Generation

```bash
# Generate all configs
talhelper genconfig

# Validate config
talhelper validate talconfig talconfig.yaml

# Generate apply command
talhelper gencommand apply --node 10.0.50.10 --extra-flags="--insecure"

# Generate upgrade command
talhelper gencommand upgrade --node 10.0.50.10

# Generate bootstrap command
talhelper gencommand bootstrap
```

## Quick Reference Table

| Task | Command |
|------|---------|
| Node status | `kubectl get nodes` |
| Pod status | `kubectl get pods -A` |
| Flux status | `flux get ks -A` |
| Cilium status | `cilium status` |
| Apply config changes | `task configure && git commit && git push` |
| Force Flux sync | `task reconcile` |
| Check node health | `talosctl health` |
| View pod logs | `kubectl logs <pod> -n <ns>` |
| Restart deployment | `kubectl rollout restart deploy/<name> -n <ns>` |
| Drain node | `kubectl drain <node> --ignore-daemonsets` |

## Environment Variables

Key environment variables set by Taskfile:

```bash
KUBECONFIG=$ROOT_DIR/kubeconfig
SOPS_AGE_KEY_FILE=$ROOT_DIR/age.key
TALOSCONFIG=$ROOT_DIR/talos/clusterconfig/talosconfig
```

You can override these by exporting them in your shell.
