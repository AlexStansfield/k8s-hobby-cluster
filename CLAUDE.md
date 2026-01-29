# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Table of Contents

- [Repository Overview](#repository-overview)
- [Network Topology](#network-topology)
- [Cluster Components](#cluster-components)
- [Common Commands](#common-commands)
- [Repository Structure](#repository-structure)
- [Architecture Patterns](#architecture-patterns)
- [Key Requirements](#key-requirements)
- [Quick Start Guide](#quick-start-guide)
- [Adapting This Setup](#adapting-this-setup)
- [Design Decisions & Rationale](#design-decisions--rationale)
- [Common Issues and Troubleshooting](#common-issues-and-troubleshooting)
- [Known Limitations & Future Enhancements](#known-limitations--future-enhancements)
- [Documentation References](#documentation-references)

## Repository Overview

This repository contains configuration, manifests, and documentation for a self-hosted Kubernetes cluster running **K3s** (lightweight Kubernetes). It's designed as a home lab environment for containerized workloads, AI services, and experimentation.

The cluster consists of:
- 1 master/controller node (Radxa X4: Intel N100, 16 GB RAM, 512 GB NVMe)
- 2-3 worker nodes (identical hardware)
- All nodes run Ubuntu Server 24.04 LTS

**For others using this repo**: This documents a specific deployment but is designed to be adapted. See [Adapting This Setup](#adapting-this-setup) for guidance on customizing for your own environment.

## Network Topology

**Important**: All network configuration assumes this specific topology:

- **Cluster subnet**: `192.168.2.0/24`
- **Gateway**: `192.168.2.254`
- **DNS/DHCP**: Pi-hole at `192.168.2.1`
- **Cluster nodes**: `192.168.2.31-40` (static DHCP reservations)
  - `192.168.2.31` - cluster-master (master node)
  - `192.168.2.32+` - worker nodes (cluster-node1, cluster-node2, etc.)
- **MetalLB pool**: `192.168.2.41-60` (for LoadBalancer services)

When adding services that need external access, use MetalLB LoadBalancer IPs from the `192.168.2.41-60` range.

## Cluster Components

### K3s Configuration
- Master node runs with `--node-ip=192.168.2.31 --flannel-iface=enp2s0`
- Worker nodes join via token from `/var/lib/rancher/k3s/server/node-token`
- All nodes require swap disabled (`swapoff -a` and `/etc/fstab` modification)
- Network interface is typically `enp2s0` (specified during install)

### Storage: Longhorn
- Distributed block storage across all nodes
- Uses `/mnt/data` partition on each node (not `/var/lib/longhorn`)
- Default StorageClass: `longhorn`
- Requires `open-iscsi` package on all nodes
- UI accessible via LoadBalancer at `192.168.2.41` (see [manifests/longhorn/longhorn-frontend-lb.yaml](manifests/longhorn/longhorn-frontend-lb.yaml))

### Load Balancing: MetalLB
- Provides LoadBalancer support in Layer 2 mode
- IP pool: `192.168.2.41-192.168.2.60`
- Configuration in [manifests/metallb/](manifests/metallb/)

### Private Registry
- Deployed via Helm chart (twuni/docker-registry)
- Runs only on master node via nodeSelector
- Accessible at `cluster-master:31234` (NodePort)
- Uses Longhorn for 50Gi persistent storage
- **Insecure registry** - requires `/etc/rancher/k3s/registries.yaml` configuration on all nodes

## Common Commands

### Cluster Management
```bash
# View cluster status
kubectl get nodes -o wide
kubectl get pods -A -o wide

# Access kubectl from master node (non-root)
export KUBECONFIG=$HOME/.kube/config

# Get join token for new worker nodes
sudo cat /var/lib/rancher/k3s/server/node-token
```

### Deploying Services

**Via kubectl:**
```bash
kubectl apply -f manifests/<service>/
```

**Via Helm:**
```bash
helm install <release-name> <chart-name> \
  -f helm/<service>/<values-file>.yaml \
  --namespace <namespace> \
  --create-namespace
```

**Private registry workflow:**
```bash
# Build and tag
docker build -t <service>:latest .
docker tag <service>:latest cluster-master:31234/<service>:latest

# Push to cluster registry
docker push cluster-master:31234/<service>:latest

# Use in manifests with image: cluster-master:31234/<service>:latest
```

### Node Configuration

**After joining a new worker node**, configure registry access:
```bash
# On each node
sudo mkdir -p /etc/rancher/k3s
sudo nano /etc/rancher/k3s/registries.yaml
```

Add:
```yaml
mirrors:
  cluster-master:31234:
    endpoint:
      - "http://cluster-master:31234"
```

Then restart:
- Master: `sudo systemctl restart k3s`
- Workers: `sudo systemctl restart k3s-agent`

## Repository Structure

```
.
├── manifests/         # Kubernetes YAML manifests for infrastructure
│   ├── longhorn/      # Longhorn UI LoadBalancer config
│   └── metallb/       # MetalLB IP pool and L2Advertisement
├── helm/              # Helm values files for chart deployments
│   └── registry/      # Private registry Helm values
└── docs/              # Setup and configuration documentation
    ├── master-setup.md    # K3s master node installation
    ├── worker-setup.md    # K3s worker node installation
    ├── longhorn-setup.md  # Longhorn storage setup
    ├── metallb-setup.md   # MetalLB load balancer setup
    └── services/
        └── registry.md    # Private registry setup
```

## Architecture Patterns

### Service Deployment Pattern
1. Services requiring persistence use Longhorn StorageClass with PersistentVolumeClaims
2. Services needing external LAN access use `type: LoadBalancer` with MetalLB
3. Services specific to master node use `nodeSelector: {kubernetes.io/hostname: cluster-master}`
4. Private images are stored in `cluster-master:31234` registry

### Storage Configuration
- Root partition: 50GB
- Data partition: Remaining space mounted at `/mnt/data`
- Longhorn configured to use `/mnt/data` instead of default `/var/lib/longhorn`

### Network Interface Handling
Nodes have multiple network interfaces. K3s installation explicitly specifies:
- `--node-ip=<static-ip>` - The static IP assigned via Pi-hole DHCP
- `--flannel-iface=enp2s0` - The primary wired interface

## Key Requirements

### Prerequisites for All Nodes
- Ubuntu Server 24.04 LTS
- Swap disabled (`swapoff -a` + `/etc/fstab` modification)
- Packages: `curl wget git htop net-tools vim nano`
- For Longhorn: `open-iscsi` package + `iscsid` service enabled
- Static DHCP reservation via Pi-hole

### Prerequisites for Master Node
- Helm installed via `curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash`
- kubectl configured at `$HOME/.kube/config` (copied from `/etc/rancher/k3s/k3s.yaml`)

## Quick Start Guide

**For someone replicating this setup from scratch:**

1. **Prepare nodes** (master + workers)
   - Install Ubuntu Server 24.04 LTS
   - Set static hostnames and configure static IPs via DHCP
   - Follow [docs/master-setup.md](docs/master-setup.md) for master node
   - Follow [docs/worker-setup.md](docs/worker-setup.md) for each worker

2. **Install storage** - [docs/longhorn-setup.md](docs/longhorn-setup.md)
   - Install `open-iscsi` on all nodes
   - Deploy Longhorn via Helm
   - Configure `/mnt/data` as storage path
   - Set as default StorageClass

3. **Install networking** - [docs/metallb-setup.md](docs/metallb-setup.md)
   - Deploy MetalLB
   - Configure IP address pool
   - Set up L2 advertisement
   - Expose Longhorn UI via LoadBalancer

4. **Install private registry** - [docs/services/registry.md](docs/services/registry.md)
   - Deploy via Helm with Longhorn storage
   - Configure all nodes with `/etc/rancher/k3s/registries.yaml`
   - Restart K3s services

5. **Deploy workloads**
   - Use `kubectl apply -f manifests/<service>/` or Helm

## Adapting This Setup

**If you're forking this repo for your own cluster**, here are the key values to customize:

### Network Configuration
Replace these in manifests and docs:
- **Subnet**: `192.168.2.0/24` → your subnet
- **Gateway**: `192.168.2.254` → your router
- **Cluster node IPs**: `192.168.2.31-40` → your node IP range
- **MetalLB pool**: `192.168.2.41-60` → your LoadBalancer IP range
- **Hostnames**: `cluster-master`, `cluster-node1`, etc. → your hostnames

Files to update:
- [manifests/metallb/metallb-ipaddresspool.yaml](manifests/metallb/metallb-ipaddresspool.yaml) - MetalLB IP range
- [manifests/longhorn/longhorn-frontend-lb.yaml](manifests/longhorn/longhorn-frontend-lb.yaml) - Longhorn UI IP
- [docs/master-setup.md](docs/master-setup.md) - Master install command `--node-ip`
- [docs/worker-setup.md](docs/worker-setup.md) - Worker install commands `--node-ip` and `--server`
- [helm/registry/registry-values.yaml](helm/registry/registry-values.yaml) - Node selector if master has different hostname

### Hardware/Storage Configuration
- **Network interface**: `enp2s0` → your interface name (find with `ip link`)
- **Data mount**: `/mnt/data` → your data partition path
- **Registry node**: If you want registry on a different node, update `nodeSelector` in [helm/registry/registry-values.yaml](helm/registry/registry-values.yaml)

### Registry Configuration
If using different hostname or port:
- Update `cluster-master:31234` references in docs and manifests
- Update `/etc/rancher/k3s/registries.yaml` on all nodes

## Design Decisions & Rationale

Understanding why certain choices were made:

**Why K3s instead of full Kubernetes?**
- Lower resource footprint suitable for SBC hardware (Intel N100 with 16GB RAM)
- Simplified installation and management for home lab
- Includes Traefik ingress and CoreDNS by default

**Why Longhorn over other storage solutions?**
- Cloud-native distributed storage designed for Kubernetes
- Replication provides resilience against node failures
- Web UI for easy management
- Works well with small clusters (3+ nodes)

**Why MetalLB in L2 mode?**
- Simplest mode for home LAN environments
- No BGP configuration needed
- Works with standard home network switches
- Allows services to get real LAN IPs

**Why insecure private registry?**
- Simpler for home lab (no TLS cert management)
- Contained to local network
- Easy to configure on all nodes
- Can be upgraded to secure registry with TLS later if needed

**Why manifests/ AND helm/ directories?**
- `manifests/`: Raw Kubernetes YAML for infrastructure services that only need configuration (MetalLB, Longhorn UI)
- `helm/`: Values files for complex applications best managed via Helm charts (private registry)
- Simple infrastructure components use manifests, stateful applications use Helm

**Why `/mnt/data` for Longhorn instead of default?**
- Separates system storage (50GB root) from data storage (remaining ~450GB)
- Allows easy expansion by replacing/adding drives
- Prevents system partition from filling up with user data

## Common Issues and Troubleshooting

### K3s Installation Issues

**Swap not disabled:**
```bash
# Symptom: K3s fails to start
# Solution: Disable swap
sudo swapoff -a
sudo nano /etc/fstab  # Comment out swap line
```

**Wrong network interface:**
```bash
# Symptom: Nodes can't communicate, flannel errors
# Find correct interface
ip link
# Reinstall with correct interface: --flannel-iface=<interface-name>
```

**Worker can't join cluster:**
```bash
# Check token on master
sudo cat /var/lib/rancher/k3s/server/node-token

# Verify master API is accessible from worker
curl -k https://192.168.2.31:6443
```

### Longhorn Issues

**Pods stuck in Pending:**
```bash
# Check if iscsid is running on all nodes
systemctl status iscsid

# Check Longhorn has valid disks configured
kubectl -n longhorn-system get nodes.longhorn.io -o yaml
```

**Storage not using /mnt/data:**
- Must configure via Longhorn UI (Settings → Node → Disks)
- Remove default `/var/lib/longhorn` disk
- Add `/mnt/data` disk with appropriate size

### MetalLB Issues

**LoadBalancer stuck in Pending:**
```bash
# Check MetalLB pods are running
kubectl -n metallb-system get pods

# Verify IPAddressPool is configured
kubectl -n metallb-system get ipaddresspool

# Check L2Advertisement exists
kubectl -n metallb-system get l2advertisement
```

**IP conflict on LAN:**
- Ensure MetalLB IP range doesn't overlap with DHCP pool
- Reserve MetalLB range in your DHCP server (Pi-hole)

### Registry Issues

**Can't push to registry:**
```bash
# Ensure /etc/rancher/k3s/registries.yaml is configured on all nodes
cat /etc/rancher/k3s/registries.yaml

# Restart k3s services after editing
# Master: sudo systemctl restart k3s
# Workers: sudo systemctl restart k3s-agent

# Test registry access
curl http://cluster-master:31234/v2/_catalog
```

**ImagePullBackOff when using registry images:**
```bash
# Check if image exists in registry
curl http://cluster-master:31234/v2/<image-name>/tags/list

# Verify registries.yaml is correct on the node pulling the image
# Check image name format: cluster-master:31234/<image>:<tag>
```

### General Debugging

```bash
# Check node status
kubectl get nodes -o wide

# Check all pods across namespaces
kubectl get pods -A -o wide

# Describe a problematic pod
kubectl describe pod <pod-name> -n <namespace>

# Check pod logs
kubectl logs <pod-name> -n <namespace>

# Check system component logs (on master)
sudo journalctl -u k3s -f

# Check worker agent logs
sudo journalctl -u k3s-agent -f
```

## Known Limitations & Future Enhancements

**Current state:**
- Registry is **insecure** (HTTP only) - suitable for isolated home lab
- Traefik ingress is default (K3s bundled) but may be replaced with NGINX
- No backup/disaster recovery automated yet

**Planned improvements** (from [README.md](README.md)):
- Upgrade to 2.5 GbE PoE+ switch for full network speed and consolidated power
- Consider HA control plane with external etcd if cluster grows
- Implement GitOps workflow with ArgoCD or Flux
- Add monitoring stack (Prometheus + Grafana)
- Secure registry with TLS certificates

## Documentation References

All setup procedures are documented in `docs/`:
- Node provisioning: [docs/master-setup.md](docs/master-setup.md), [docs/worker-setup.md](docs/worker-setup.md)
- Storage: [docs/longhorn-setup.md](docs/longhorn-setup.md)
- Networking: [docs/metallb-setup.md](docs/metallb-setup.md)
- Services: [docs/services/](docs/services/)

When modifying cluster configuration, update the corresponding documentation in `docs/`.
