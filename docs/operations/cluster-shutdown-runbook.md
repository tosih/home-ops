# Cluster Shutdown / Power-On Runbook

Procedure for safely powering down the whole cluster (e.g. UPS test, hardware
maintenance, extended outage) and bringing it back up cleanly.

## Cluster topology

- Control planes: `control0` (10.0.50.1), `control1` (10.0.50.2)
- Workers: `worker0` (10.0.50.10), `worker1` (10.0.50.11), `worker2` (10.0.50.12)
- Stateful workloads to protect: Longhorn volumes, CloudNativePG, Dragonfly, VerneMQ

!!! warning "Only 2 control planes = zero etcd fault tolerance"
    etcd quorum on a 2-node cluster is 2-of-2. The instant either `control0`
    or `control1` goes down, the kube-apiserver stops responding for the
    **entire** cluster — the other control plane alone cannot serve requests.
    Because of this, both control planes must be drained *while both are still
    up*, then shut down back-to-back. You cannot drain the second one after
    the first is already off.

## Shutting down

### 1. Pre-flight checks

```bash
kubectl get nodes -o wide
kubectl get pods -A --field-selector=status.phase!=Running,status.phase!=Succeeded
kubectl -n storage get volumes.longhorn.io   # all should be Healthy, none Degraded/Rebuilding
```

Optional but recommended for a long outage, to stop Flux reconciling mid-drain:

```bash
flux suspend kustomization --all -n flux-system
for ns in $(kubectl get helmrelease -A -o jsonpath='{range .items[*]}{.metadata.namespace}{"\n"}{end}' | sort -u); do
  flux suspend helmrelease --all -n "$ns"
done
```

### 2. Cordon every node first, then stop workloads in place

!!! warning "Don't `kubectl drain` node-by-node for a full shutdown"
    `drain` evicts pods so they *reschedule onto other nodes* — that's the
    right tool when one node is leaving the pool and the rest of the cluster
    stays up to absorb the load. Here every node is going down, so there is
    no safe place to reschedule to. Draining sequentially just bounces pods
    from the node you're draining onto whichever nodes you haven't gotten to
    yet, causing pointless churn — and for CloudNativePG specifically it can
    bounce the primary across nodes and leave replicas `Pending` once every
    node is cordoned. Cordon everything up front instead, so the scheduler
    stops placing anything anywhere, then shut workloads down in place.

```bash
kubectl cordon control0 control1 worker0 worker1 worker2
```

Hibernate CloudNativePG clusters (clean Postgres shutdown with a checkpoint,
scales instances to 0, keeps PVCs) instead of letting it get evicted:

```bash
kubectl annotate cluster.postgresql.cnpg.io --all -n databases cnpg.io/hibernation=on
```

Scale down the other clustered/stateful apps directly rather than relying on
eviction (this also sidesteps any PodDisruptionBudget that would otherwise
block eviction with nowhere to go):

```bash
kubectl scale statefulset dragonfly -n databases --replicas=0
kubectl scale statefulset vernemq -n databases --replicas=0
```

Everything else (plain Deployments) can just be scaled to 0, or left alone —
since every node is cordoned, deleting/evicting them cannot cause
reschedule-churn:

```bash
kubectl get deployments -A -o json | \
  jq -r '.items[] | select(.spec.replicas > 0) | "\(.metadata.namespace) \(.metadata.name)"' | \
  while read -r ns name; do kubectl scale deployment "$name" -n "$ns" --replicas=0; done
```

### 3. Power off the workers via Talos

```bash
talosctl shutdown -n 10.0.50.10,10.0.50.11,10.0.50.12
```

### 4. Drain both control planes while the API is still up

```bash
kubectl drain control0 --ignore-daemonsets --delete-emptydir-data
kubectl drain control1 --ignore-daemonsets --delete-emptydir-data
```

### 5. Shut both control planes down together

Once the first one goes down the API is gone regardless, so there's no
benefit to sequencing these — issue both in the same command:

```bash
talosctl shutdown -n 10.0.50.1,10.0.50.2
```

## Powering back on (reverse order)

1. Power on `control0` and `control1` physically, then wait for etcd/apiserver:
   ```bash
   talosctl -n 10.0.50.1,10.0.50.2 health
   ```
2. Power on `worker0`, `worker1`, `worker2`.
3. Uncordon every node:
   ```bash
   kubectl get nodes -o name | xargs -n1 kubectl uncordon
   ```
4. Un-hibernate CloudNativePG and scale the manually-scaled StatefulSets back up:
   ```bash
   kubectl annotate cluster.postgresql.cnpg.io --all -n databases cnpg.io/hibernation-
   kubectl scale statefulset dragonfly -n databases --replicas=1
   kubectl scale statefulset vernemq -n databases --replicas=1
   ```
5. Resume Flux if it was suspended — this also brings the plain Deployments
   back to their declared replica counts, since Flux reconciles them back to
   what's in Git:
   ```bash
   flux resume kustomization --all -n flux-system
   for ns in $(kubectl get helmrelease -A -o jsonpath='{range .items[*]}{.metadata.namespace}{"\n"}{end}' | sort -u); do
     flux resume helmrelease --all -n "$ns"
   done
   ```
6. Confirm Longhorn volumes reattach cleanly:
   ```bash
   kubectl -n storage get volumes.longhorn.io
   ```
