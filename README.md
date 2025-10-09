# ⛵ Home-Ops Kubernetes Cluster

A production-grade home Kubernetes cluster running on Talos Linux with GitOps management via Flux.

## 📖 Documentation

**[View Full Documentation →](https://tosih.github.io/home-ops)**

Complete guides for deployment, operations, and maintenance are available in the documentation site.

## ✨ Highlights

- **OS**: [Talos Linux](https://www.talos.dev/) - Immutable Kubernetes OS
- **GitOps**: [Flux](https://fluxcd.io/) - Continuous delivery from Git
- **CNI**: [Cilium](https://cilium.io/) - eBPF-based networking
- **Storage**: [Rook-Ceph](https://rook.io/) - Distributed storage
- **Ingress**: Gateway API with Cloudflare Tunnel
- **Secrets**: SOPS with age encryption

## 🚀 Quick Start

```bash
# Install tools
mise trust && pip install pipx && mise install

# Initialize configuration
task init

# Configure cluster
task configure

# Deploy
task bootstrap:talos
task bootstrap:apps
```

See the [Getting Started](https://tosih.github.io/home-ops/getting-started/initial-setup/) guide for detailed instructions.

## 🏗️ Infrastructure

| Component | Count | Specs |
|-----------|-------|-------|
| Control Plane | 2 | 4 CPU, 16GB RAM |
| Workers | 3 | 4 CPU, 16GB RAM |
| Storage | 300GB | Ceph distributed storage |

## 📚 Key Documentation

- **[Prerequisites](https://tosih.github.io/home-ops/getting-started/prerequisites/)** - Required tools and accounts
- **[Architecture](https://tosih.github.io/home-ops/reference/architecture/)** - System design and structure
- **[Operations](https://tosih.github.io/home-ops/operations/worker-upgrade-procedure/)** - Maintenance procedures
- **[Storage](https://tosih.github.io/home-ops/storage/rook-ceph-setup/)** - Persistent storage setup

## 🔧 Built With

This cluster is based on [@onedr0p's cluster-template](https://github.com/onedr0p/cluster-template) and uses [makejinja](https://github.com/mirkolenz/makejinja) for template-driven configuration.

## 📝 License

See [LICENSE](./LICENSE)

---

⭐ Star this repo if you find it helpful!
