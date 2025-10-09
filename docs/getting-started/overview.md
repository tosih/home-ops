# Overview

## What is home-ops?

Home-ops is a production-grade Kubernetes cluster designed for home lab and self-hosting use cases. It combines modern cloud-native technologies with practical operational procedures.

## Architecture

The cluster consists of:

- **2 Control Plane Nodes** - High availability for the Kubernetes API
- **3 Worker Nodes** - Run your workloads with redundancy
- **GitOps Management** - Flux continuously deploys from this Git repository
- **Template-Driven Config** - Easy cluster configuration via YAML files

## Technology Stack

### Core Components

| Component | Purpose | Version |
|-----------|---------|---------|
| Talos Linux | Immutable Kubernetes OS | v1.11.1 |
| Kubernetes | Container orchestration | v1.34.1 |
| Flux | GitOps CD operator | v2.6.4 |
| Cilium | CNI and networking | Latest |

### Storage

| Component | Purpose |
|-----------|---------|
| Rook-Ceph | Distributed storage | v1.16.3 |
| ZFS Provisioner | Local ZFS storage | Latest |

### Networking

| Component | Purpose |
|-----------|---------|
| Gateway API | Ingress routing | Latest |
| Cloudflare Tunnel | External access | Latest |
| external-dns | DNS automation | Latest |
| cert-manager | TLS certificates | Latest |

### Security

| Component | Purpose |
|-----------|---------|
| SOPS | Secret encryption | v3.10.2 |
| age | Encryption keys | v1.2.1 |
| External Secrets | Secret management | Latest |

## Design Principles

### GitOps First

All cluster configuration lives in Git. Changes are applied by pushing commits, not by running `kubectl apply`.

### Immutable Infrastructure

Talos Linux provides an immutable, API-managed operating system. No SSH access needed for normal operations.

### High Availability

Critical components run with multiple replicas across different nodes for resilience.

### Template-Driven

The cluster uses [makejinja](https://github.com/mirkolenz/makejinja) to generate configurations from simple YAML files, reducing repetition and errors.

## Network Architecture

### IP Addressing

- **Node CIDR:** 10.0.0.0/16
- **Pod CIDR:** 10.42.0.0/16
- **Service CIDR:** 10.43.0.0/16
- **Control Plane VIP:** 10.0.50.50

### Load Balancer IPs

| Service | IP | Purpose |
|---------|-----|---------|
| k8s-gateway | 10.0.50.100 | Internal DNS |
| Internal Gateway | 10.0.50.101 | Internal ingress |
| External Gateway | 10.0.50.102 | External ingress via Cloudflare |

### Gateways

The cluster uses **Gateway API** (not Ingress) with two gateways:

- **Internal** - For services accessible only on your home network
- **External** - For services exposed via Cloudflare Tunnel

## Next Steps

Ready to get started?

1. [Check prerequisites](prerequisites.md) - Ensure you have required tools and accounts
2. [Initial setup](initial-setup.md) - Deploy your cluster from scratch

Already have a running cluster? Jump to:

- [Operations guides](../operations/worker-upgrade-procedure.md) - Maintenance procedures
- [Storage setup](../storage/rook-ceph-setup.md) - Configure persistent storage
