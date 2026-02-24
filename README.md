# ⛵ Home-Ops Kubernetes Cluster

Home Kubernetes cluster running on Talos Linux with GitOps management via Flux.

## 📖 Documentation

**[View Full Documentation →](https://tosih.github.io/home-ops)**

Complete guides for deployment, operations, and maintenance are available in the documentation site.

## ✨ Highlights

- **OS**: [Talos Linux](https://www.talos.dev/) v1.11.1 - Immutable Kubernetes OS
- **Kubernetes**: v1.34.1 - Container orchestration platform
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
| Storage | Multiple | Rook-Ceph (block, filesystem, object), ZFS NFS |
| Applications | 35+ | Media automation, photos, home automation, cloud services |

## 🚀 Deployed Applications

| Application | Namespace | Purpose | URL |
|------------|-----------|---------|-----|
| **Media & Entertainment** | | | |
| Plex | media | Media streaming server | [plex.tosih.org](https://plex.tosih.org) |
| Jellyseerr | media | Media request management | [requests.tosih.org](https://requests.tosih.org) |
| Sonarr | media | TV show automation | [sonarr.tosih.org](https://sonarr.tosih.org) |
| Radarr | media | Movie automation | [radarr.tosih.org](https://radarr.tosih.org) |
| Lidarr | media | Music automation | [lidarr.tosih.org](https://lidarr.tosih.org) |
| Readarr | media | eBook & audiobook automation | [readarr.tosih.org](https://readarr.tosih.org) |
| Prowlarr | media | Indexer management | [prowlarr.tosih.org](https://prowlarr.tosih.org) |
| Recyclarr | media | TRaSH guide automation | - |
| qBittorrent | media | Torrent download client | [qbittorrent.tosih.org](https://qbittorrent.tosih.org) |
| NZBGet | media | Usenet download client | [nzbget.tosih.org](https://nzbget.tosih.org) |
| Audiobookshelf | media | Audiobook & podcast server | [audiobooks.tosih.org](https://audiobooks.tosih.org) |
| Beets | media | Music library manager | - |
| **Cloud Services** | | | |
| Immich | cloud | Photo & video backup (OIDC) | [photos.tosih.org](https://photos.tosih.org) |
| ImmichFrame | cloud | Digital photo frame for Immich | [frame.tosih.org](https://frame.tosih.org) |
| Memos | cloud | Note-taking service | [memos.tosih.org](https://memos.tosih.org) |
| Romm | cloud | ROM manager for retro gaming | [romm.tosih.org](https://romm.tosih.org) |
| Syncthing | cloud | Continuous file synchronization | [sync.tosih.org](https://sync.tosih.org) |
| **Home Automation** | | | |
| Home Assistant | home | Home automation platform | [home.tosih.org](https://home.tosih.org) |
| Homebridge | home | HomeKit bridge | [homebridge.tosih.org](https://homebridge.tosih.org) |
| AirConnect | home | AirPlay to UPnP/Sonos bridge | - |
| Eufy Security WS | home | Eufy camera integration | - |
| **Infrastructure** | | | |
| Homepage | default | Application dashboard | [dashboard.tosih.org](https://dashboard.tosih.org) |
| Uptime Kuma | default | Uptime monitoring | [uptime.tosih.org](https://uptime.tosih.org) |
| Echo | default | HTTP echo server | - |
| **Network Services** | | | |
| AdGuard Home | network | DNS server & ad blocking | [dns.tosih.org](https://dns.tosih.org) |
| k8s-gateway | network | Internal DNS for *.tosih.org | 10.0.50.100 |
| Cloudflare Tunnel | network | Secure external access | - |
| Cloudflare DNS | network | DNS record automation | - |
| **Security & Authentication** | | | |
| Pocket ID | security | OIDC identity provider (SSO) | [pid.tosih.org](https://pid.tosih.org) |
| External Secrets | security | 1Password secret integration | - |
| OnePassword Connect | security | 1Password API server | - |
| **Storage & Databases** | | | |
| Rook-Ceph | rook-ceph | Distributed storage (block, filesystem, object) | [rook.tosih.org](https://rook.tosih.org) |
| ZFS Provisioner | kubernetes-zfs-provisioner | Local ZFS storage provisioning | - |
| CloudNativePG | databases | PostgreSQL operator | - |
| Dragonfly | databases | Redis-compatible in-memory store | - |
| External Postgres Operator | databases | External DB management | - |
| VerneMQ | databases | MQTT message broker | - |
| **Platform Services** | | | |
| Flux | flux-system | GitOps continuous delivery | - |
| Cilium | kube-system | CNI & Gateway API | - |
| Cert-Manager | cert-manager | TLS certificate management | - |
| CoreDNS | kube-system | Cluster DNS service | - |
| Metrics Server | kube-system | Resource metrics API | - |
| Reloader | kube-system | Auto-reload on config changes | - |
| Spegel | kube-system | Distributed image cache | - |
| Descheduler | kube-system | Pod rescheduling optimization | - |
| Snapshot Controller | kube-system | Volume snapshot support | - |

**Total: 50+ applications** across 10 namespaces

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
