# Worker Node Setup

This document describes the process of preparing and configuring **worker nodes** to join the Kubernetes cluster.  
The steps cover system preparation, networking, and connecting to the master node running **k3s**.

## 1. Base OS Installation

- Install **Ubuntu Server 24.04 LTS (Noble Numbat)** on each worker node.  
- Use a minimal installation (no desktop environment).  
- Ensure SSH is enabled during installation.  
- Set a unique static hostname, e.g. `cluster-node1`, `cluster-node2`, etc.
- Configure storage to 50G root and remaining to `/mnt/data`.


## 2. Networking

Assign each worker node a static IP to it's ethernet MAC address via Pi-hole DHCP reservation.  
- Reserved range: **192.168.2.31 → 192.168.2.40**  
- Example:  
  - `192.168.2.32` → `cluster-node1`  
  - `192.168.2.33` → `cluster-node2`  

Verify connectivity:

```bash
ping 192.168.2.1     # pi-hole
ping 192.168.2.254   # main network gateway
ping google.com      # external connectivity
```

## 3. System Preparation

Update and install baseline tools:

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install -y curl wget git htop net-tools vim nano
```

### Disable Swap

Kubernetes requires swap to be disabled.  

1. Turn off swap immediately:

```bash
sudo swapoff -a
```

2. Remove or comment out any swap entries in /etc/fstab:

```bash
sudo nano /etc/fstab
```

Look for a line referencing swap (e.g., /swap.img or a swap partition) and comment it out with #.

3. Verify swap is disabled:

```bash
free -h
```

The Swap row should show 0B.

## 4. K3s Installation (Worker Node)

You will need:

The master node’s IP (e.g. 192.168.2.31)

The cluster join token, retrieved on the master with:

```bash
sudo cat /var/lib/rancher/k3s/server/node-token
```

Install k3s agent on the worker:

```bash
curl -sfL https://get.k3s.io | \
  INSTALL_K3S_EXEC="agent \
    --server https://192.168.2.31:6443 \
    --token <MASTER_NODE_TOKEN> \
    --node-ip=192.168.2.32 \
    --flannel-iface=enp2s0" \
  sh -
```

- `--server` → API endpoint of the master node
- `--token` → secure join token from master
- `--node-ip` → static IP of this worker node
- `--flannel-iface` → interface to use (wired NIC, typically enp2s0)

Repeat this for each worker node, changing the `--node-ip` and `--flannel-iface` accordingly.

## 5. Verify Cluster Membership

From the master node:

```bash
kubectl get nodes -o wide
```

You should see the worker nodes listed alongside the master, each in Ready status.

Confirm system pods are distributed across nodes:

```bash
kubectl get pods -A -o wide
```

## 6. Post Setup (Optional)

Label nodes if you want to schedule workloads to specific workers:

```bash
kubectl label node cluster-node1 node-role.kubernetes.io/worker=worker
```