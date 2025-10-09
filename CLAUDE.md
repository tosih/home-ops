# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Overview

This is a Talos Linux-based Kubernetes cluster deployment using GitOps principles with Flux. The repository is based on the onedr0p/cluster-template and uses a template-driven approach to generate cluster configurations from YAML files.

## Core Architecture

### Configuration Generation System

The repository uses **makejinja** to generate configurations from templates. This is a crucial workflow:

1. **Source files**: `cluster.yaml` and `nodes.yaml` contain all cluster and node-specific configuration
2. **Templates**: Located in `templates/` directory with `.j2` extension
3. **Generation**: `task configure` renders templates into the target directories (`kubernetes/`, `talos/`, `bootstrap/`)
4. **Template syntax**: Uses custom Jinja2 delimiters:
   - Block: `#% ... %#`
   - Variable: `#{ ... }#`
   - Comment: `#| ... |#`

**Important**: After modifying `cluster.yaml` or `nodes.yaml`, you MUST run `task configure` to regenerate the cluster configs. Do not manually edit generated files in `kubernetes/`, `talos/`, or `bootstrap/` - edit the templates or source YAML files instead.

### Directory Structure

- **`talos/`**: Talos Linux configuration files generated from templates
  - `talconfig.yaml`: Main talhelper configuration (generated)
  - `patches/`: Talos machine configuration patches
  - `clusterconfig/`: Generated per-node configurations
- **`kubernetes/`**: All Kubernetes manifests organized by category
  - `apps/`: Application deployments organized by namespace
  - `flux/`: Flux system configuration and cluster-wide Kustomizations
- **`bootstrap/`**: Helmfile configurations for initial cluster bootstrap
- **`templates/`**: Jinja2 templates that generate the above directories
- **`.taskfiles/`**: Task definitions for various operations (bootstrap, talos, template)
- **`scripts/`**: Automation scripts (e.g., bootstrap-apps.sh)

### GitOps Workflow

The cluster uses **Flux** for continuous deployment:

1. Flux monitors this Git repository (defined in `cluster.yaml` as `repository_name`)
2. Two main Kustomizations in `kubernetes/flux/cluster/ks.yaml`:
   - `cluster-meta`: Deploys Flux repositories and dependencies first
   - `cluster-apps`: Deploys all applications from `kubernetes/apps/`
3. Each app namespace has its own `ks.yaml` that Flux reconciles
4. Uses SOPS with age for secret encryption (all `*.sops.yaml` files are encrypted)

### Secret Management

- **SOPS** encrypts secrets using **age** encryption
- Age key location: `age.key` (set in env as `SOPS_AGE_KEY_FILE`)
- SOPS config: `.sops.yaml` defines encryption rules
- Pattern: Files matching `*.sops.yaml` or `*.sops.yml` are encrypted
- **Critical**: Never commit unencrypted secrets. The `task configure` command automatically encrypts secrets.

## Common Development Commands

### Initial Setup & Configuration

```bash
# Install all required tools via mise
mise trust
pip install pipx
mise install

# Initialize config files (only needed once)
task init

# Generate cluster configurations from templates
# Run this after ANY changes to cluster.yaml, nodes.yaml, or templates/
task configure
```

### Talos Operations

```bash
# Bootstrap Talos cluster (first-time only)
task bootstrap:talos

# Generate Talos configs without applying
task talos:generate-config

# Apply config to specific node (MODE: auto, no-reboot, reboot, staged)
task talos:apply-node IP=10.0.50.1 MODE=auto

# Upgrade Talos on a node
task talos:upgrade-node IP=10.0.50.1

# Upgrade Kubernetes version
task talos:upgrade-k8s

# Reset cluster to maintenance mode (destructive!)
task talos:reset
```

### Flux & Kubernetes Operations

```bash
# Bootstrap apps into cluster (after Talos bootstrap)
task bootstrap:apps

# Force Flux to reconcile from Git
task reconcile

# Debug cluster resources
task template:debug
```

### Template Management

```bash
# After initial setup, archive template files to clean up repo
task template:tidy

# Remove all generated configs (destructive!)
task template:reset
```

## Application Deployment Pattern

Apps follow a consistent structure in `kubernetes/apps/`:

```
apps/
├── <namespace>/
│   ├── kustomization.yaml          # Lists all apps in this namespace
│   ├── <app-name>/
│   │   ├── ks.yaml                 # Flux Kustomization for this app
│   │   └── app/
│   │       ├── kustomization.yaml
│   │       ├── helmrelease.yaml    # or deployment.yaml for plain manifests
│   │       ├── ocirepository.yaml  # OCI source for Helm charts
│   │       └── *.sops.yaml         # Encrypted secrets (if needed)
```

### Adding New Applications

1. Create directory structure under `kubernetes/apps/<namespace>/<app-name>/`
2. Add `ks.yaml` to define Flux Kustomization
3. Create `app/` directory with manifests
4. For Helm: create `ocirepository.yaml` and `helmrelease.yaml`
5. Reference the app in parent `kustomization.yaml`
6. Commit and push - Flux will deploy automatically

## Important Networking Details

- **Cluster API VIP**: 10.0.50.50 (shared by control plane nodes)
- **Load Balancer IPs**:
  - k8s_gateway (DNS): 10.0.50.100
  - Internal gateway: 10.0.50.101
  - External gateway: 10.0.50.102
- **CNI**: Cilium (built-in Talos CNI is disabled)
- **Ingress**: Gateway API with Cilium (not Ingress resources)
- **DNS**: CoreDNS + k8s_gateway for internal, external-dns for Cloudflare

## Critical Environment Variables

Set automatically by Taskfile but important to know:

- `KUBECONFIG`: `./kubeconfig`
- `SOPS_AGE_KEY_FILE`: `./age.key`
- `TALOSCONFIG`: `./talos/clusterconfig/talosconfig`

## Cluster Configuration Files

- **`cluster.yaml`**: Cluster-wide settings (network CIDRs, domain, Cloudflare config)
- **`nodes.yaml`**: Node-specific settings (hostname, IP, MAC, disk serial, schematic ID)
- **`talenv.yaml`** (in talos/): Talos and Kubernetes versions

## Key Tools & Their Roles

- **mise**: Developer environment and tool version management
- **task**: Task automation (replaces make)
- **talhelper**: Generates Talos configs from talconfig.yaml
- **makejinja**: Template rendering engine
- **flux**: GitOps CD operator
- **sops**: Secret encryption
- **cilium**: CNI and networking
- **helmfile**: Used during bootstrap phase only

## Testing & Validation

The `task configure` command validates:

1. Schema validation with CUE (cluster.yaml and nodes.yaml)
2. Kubernetes manifest validation with kubeconform
3. Talos config validation with talhelper
4. Automatic SOPS encryption of secrets

## Troubleshooting Tips

- If Flux isn't syncing: `task reconcile`
- Check Flux status: `flux get ks -A` and `flux get hr -A`
- Pod not starting: `kubectl -n <namespace> describe pod <pod-name>`
- Check Cilium: `cilium status`
- Talos issues: `talosctl -n <node-ip> dmesg` or `talosctl -n <node-ip> logs`
- View namespace events: `kubectl -n <namespace> get events --sort-by='.metadata.creationTimestamp'`

## Repository Workflow

1. Modify `cluster.yaml`, `nodes.yaml`, or templates
2. Run `task configure` to regenerate configs
3. Review generated changes
4. Commit ONLY if `*.sops.*` files are encrypted
5. Push to Git
6. Flux automatically reconciles changes to cluster

## Security Notes

- All sensitive values in `cluster.yaml` should be replaced with SOPS-encrypted secrets after initial setup
- The `github-deploy.key.pub` public key must be added to GitHub deploy keys for private repos
- Cloudflare tunnel credentials are stored in `cloudflare-tunnel.json` (not committed in template)
