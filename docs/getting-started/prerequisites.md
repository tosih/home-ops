# Prerequisites

Before deploying the cluster, ensure you have the following requirements met.

## Hardware Requirements

### Minimum Configuration

For a working cluster, you'll need **at least 3 nodes** (can be physical or VMs):

| Role | Nodes | CPU | RAM | Storage |
|------|-------|-----|-----|---------|
| Control Plane | 2 | 4 cores | 16GB | 256GB SSD |
| Worker | 3 | 4 cores | 16GB | 256GB SSD |

!!! tip "Control Plane Nodes"
    In this setup, all nodes can run workloads. The control plane nodes are configured with `controlPlane: true` but still accept pods.

### Network Requirements

- **Static IP addresses** for all nodes
- **Same Layer 2 network** (or proper routing configured)
- **Internet connectivity** for pulling images
- **Unique MAC addresses** for each node

## Software Requirements

### Local Workstation

Your workstation needs these tools installed:

| Tool | Purpose | Install via mise? |
|------|---------|-------------------|
| mise | Tool version manager | [Manual install](https://mise.jdx.dev/getting-started.html) |
| task | Task automation | ✅ Yes |
| kubectl | Kubernetes CLI | ✅ Yes |
| talosctl | Talos CLI | ✅ Yes |
| talhelper | Talos config generator | ✅ Yes |
| flux | GitOps CLI | ✅ Yes |
| sops | Secret encryption | ✅ Yes |
| age | Encryption keys | ✅ Yes |
| cilium | Cilium CLI | ✅ Yes |
| helm | Helm CLI | ✅ Yes |
| kustomize | Kustomize CLI | ✅ Yes |
| yq | YAML processor | ✅ Yes |
| jq | JSON processor | ✅ Yes |

### Installing Tools

This repository uses [mise](https://mise.jdx.dev/) to manage tool versions:

```bash
# 1. Install mise (one-time setup)
curl https://mise.run | sh

# 2. Activate mise in your shell
# Follow instructions at: https://mise.jdx.dev/getting-started.html#activate-mise

# 3. Install all tools from .mise.toml
cd home-ops
mise trust
pip install pipx  # Required for Python tools
mise install
```

## Cloud Accounts

### Cloudflare (Required)

You need a Cloudflare account with:

- **A registered domain** under your account
- **API token** with these permissions:
  - Zone:DNS:Edit
  - Account:Cloudflare Tunnel:Read

See the [Cloudflare token creation docs](https://developers.cloudflare.com/fundamentals/api/get-started/create-token/) for details.

### GitHub (Required)

- **GitHub account** to host this repository
- **Repository** (public or private)
- For private repos: Deploy key configured (generated during setup)

## Operating System Images

### Talos Linux

You'll need to prepare Talos installation media:

1. Visit [Talos Image Factory](https://factory.talos.dev)
2. Select **minimal system extensions** needed for your hardware
3. Download ISO (for bare metal) or RAW image (for VMs/SBCs)
4. Note the **schematic ID** - you'll need this in `nodes.yaml`

!!! warning "System Extensions"
    Only select extensions you absolutely need. Some extensions require additional configuration and may prevent boot if misconfigured.

## Node Preparation

Before installing Talos:

1. **Flash the Talos image** to USB drives or configure PXE boot
2. **Boot each node** from the Talos media
3. **Verify network connectivity:**
   ```bash
   nmap -Pn -n -p 50000 10.0.0.0/16 -vv | grep 'Discovered'
   ```
4. **Record each node's:**
   - IP address
   - MAC address
   - Disk serial number or path

## DNS Configuration

For proper operation, you should configure split DNS:

1. **Internal DNS server** (Pi-hole, Adguard, router, etc.)
2. **Forward `*.yourdomain.com`** to the k8s-gateway IP (10.0.50.100)
3. **Keep other queries** going to your normal upstream DNS

This allows internal services to resolve correctly while external access goes through Cloudflare.

## Optional but Recommended

- **Backup solution** for your data
- **UPS** for power protection
- **Separate management network** for Talos nodes
- **Monitoring** (Prometheus/Grafana can be added later)

## Next Steps

Once you have all prerequisites ready:

→ [Initial Setup](initial-setup.md) - Deploy your cluster
