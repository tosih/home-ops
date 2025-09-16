Credit to: 

# Home-Ops

# Nodes

All nodes are part of the control plane (no workers)
* 3x HP T740
* 1x HP T640
* 1x ZFS Storage host (original home server)

| x | Model | OS | CPU | Memory | Storage |
| --- | --- | --- | --- | --- | --- |
| 3 | HP T740 | Talos 1.11.1 | AMD Ryzen™ V1756B | 8GB | 256GB NVME |
| 1 | HP T640 | Talos 1.11.1 | AMD Ryzen R1505G | 8GB | 64GB EMMC |
| 1 | DIY | Ubuntu Server | i3 3225 | 16GB | 6x 6TB HDD ZFS |


# Storage

Storage is served from separate server hosting ZFS datasets over NFS.
Using a [fork](https://github.com/tosih/kubernetes-zfs-provisioner) of `kubernetes-zfs-provisioner`

# Credits
* [cluster-template](https://github.com/onedr0p/cluster-template)
* [home-ops](https://github.com/ahinko/home-ops)
* [kubernetes-zfs-provisioner](https://github.com/ccremer/kubernetes-zfs-provisioner)
