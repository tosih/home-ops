# Home-Ops Documentation

Welcome to the home-ops cluster documentation! This site provides comprehensive guides for deploying and operating a Talos Linux-based Kubernetes cluster using GitOps principles.

## About This Cluster

This is a home Kubernetes cluster built with:

- **[Talos Linux](https://www.talos.dev/)** - Immutable Kubernetes OS
- **[Flux](https://fluxcd.io/)** - GitOps continuous delivery
- **[Cilium](https://cilium.io/)** - Advanced networking and security

## Quick Links

<div class="grid cards" markdown>

-   :material-rocket-launch:{ .lg .middle } __Getting Started__

    ---

    New to this cluster? Start here to understand the architecture and initial setup.

    [:octicons-arrow-right-24: Get started](getting-started/overview.md)

-   :material-book-open-variant:{ .lg .middle } __Reference__

    ---

    Detailed reference documentation for cluster architecture and tools.

    [:octicons-arrow-right-24: Reference docs](reference/architecture.md)

</div>

## Cluster Overview

### Infrastructure

- **Control Plane Nodes:** 2 nodes (control0, control1)
- **Worker Nodes:** 3 nodes (worker0, worker1, worker2)
- **Network:** 10.0.0.0/16 (nodes on 10.0.50.0/24)
- **CNI:** Cilium with Gateway API
- **Storage:** Local ZFS storage provisioner

### Key Features

- ✅ **GitOps-driven** - All configuration stored in Git
- ✅ **Highly Available** - Multi-node control plane
- ✅ **Encrypted Secrets** - SOPS with age encryption
- ✅ **External Access** - Cloudflare Tunnel integration
- ✅ **Persistent Storage** - Local ZFS storage
- ✅ **Automated Updates** - Renovate for dependency management

## Contributing

This cluster follows a template-driven approach using [makejinja](https://github.com/mirkolenz/makejinja). When making changes:

1. Edit `cluster.yaml` or `nodes.yaml` for configuration
2. Run `task configure` to regenerate manifests
3. Commit and push - Flux handles deployment

See the [architecture reference](reference/architecture.md) for more details.

## Support

- **GitHub Issues:** [Report issues](https://github.com/tosih/home-ops/issues)
- **Discussions:** [Ask questions](https://github.com/tosih/home-ops/discussions)

---

!!! tip "Based on cluster-template"
    This cluster is based on [@onedr0p's cluster-template](https://github.com/onedr0p/cluster-template) - an excellent starting point for home Kubernetes clusters!
