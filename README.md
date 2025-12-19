# ⛵ Home-Ops Kubernetes Cluster

A production-grade home Kubernetes cluster running on Talos Linux with GitOps management via Flux.

## 📖 Documentation

**[View Full Documentation →](https://tosih.github.io/home-ops)**

Complete guides for deployment, operations, and maintenance are available in the documentation site.

## ✨ Highlights

- **OS**: [Talos Linux](https://www.talos.dev/) v1.11.1 - Immutable Kubernetes OS
- **Kubernetes**: v1.34.1 - Latest stable version
- **GitOps**: [Flux](https://fluxcd.io/) - Continuous delivery from Git
- **CNI**: [Cilium](https://cilium.io/) - eBPF-based networking with Gateway API
- **Storage**: [Rook-Ceph](https://rook.io/) - Distributed block and filesystem storage
- **Ingress**: Cilium Gateway API (internal & external gateways)
- **Secrets**: SOPS with age encryption + External Secrets with 1Password
- **Authentication**: [Pocket ID](https://github.com/stonith404/pocket-id) - OIDC provider for SSO

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

| Component | Count | Details |
|-----------|-------|---------|
| Control Plane | 2 | Talos v1.11.1, Kubernetes v1.34.1 |
| Workers | 3 | High availability workload distribution |
| Storage | Multiple | Rook-Ceph (block, filesystem), ZFS NFS |
| Applications | 16+ | Media automation, photos, home automation |

## 🚀 Deployed Applications

### Media & Entertainment
- **[Plex](https://plex.tosih.org)** - Media streaming server
- **[Jellyseerr](https://requests.tosih.org)** - Media request management
- **[Sonarr](https://sonarr.tosih.org)** - TV show automation
- **[Radarr](https://radarr.tosih.org)** - Movie automation
- **[Prowlarr](https://prowlarr.tosih.org)** - Indexer management
- **[qBittorrent](https://qbittorrent.tosih.org)** - Torrent client
- **[NZBGet](https://nzbget.tosih.org)** - Usenet downloader

### Cloud Services
- **[Immich](https://photos.tosih.org)** - Self-hosted photo and video backup with OIDC

### Home Automation
- **[Home Assistant](https://home.tosih.org)** - Home automation platform
- **[Homebridge](https://homebridge.tosih.org)** - HomeKit bridge

### Security & Infrastructure
- **[Pocket ID](https://pid.tosih.org)** - OIDC identity provider (SSO)
- **[Homepage](https://dashboard.tosih.org)** - Application dashboard
- **[Rook-Ceph Dashboard](https://rook.tosih.org)** - Storage cluster management

## 📚 Key Documentation

- **[Prerequisites](https://tosih.github.io/home-ops/getting-started/prerequisites/)** - Required tools and accounts
- **[Architecture](https://tosih.github.io/home-ops/reference/architecture/)** - System design and structure
- **[Authentication](https://tosih.github.io/home-ops/reference/authentication/)** - OIDC and SSO setup
- **[Post Installation](https://tosih.github.io/home-ops/getting-started/post-installation/)** - Maintenance and upgrades

## 🔧 Built With

This cluster is based on [@onedr0p's cluster-template](https://github.com/onedr0p/cluster-template) and uses [makejinja](https://github.com/mirkolenz/makejinja) for template-driven configuration.

## 📝 License

See [LICENSE](./LICENSE)

---

⭐ Star this repo if you find it helpful!
