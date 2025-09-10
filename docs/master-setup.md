# Master Node Setup

This document describes the process of preparing and configuring the **master (controller) node** for the Kubernetes cluster.

The steps cover system preparation, OS configuration, and installation of the Kubernetes distribution (**k3s**).

## 1. Base OS Installation

- Install **Ubuntu Server 24.04 LTS (Noble Numbat)**.  
- Use a minimal installation (no desktop environment).  
- Ensure SSH is enabled during installation.  
- Set a static hostname, e.g. `cluster-master`.
- Configure storage to 50G root and remaining to `/mnt/data`.

## 2. Networking

- Assign a static IP address to the ethernet MAC address via Pi-hole DHCP reservation.  
- Verify connectivity:

  ```bash
  ping 192.168.2.1   # pi-hole
  ping 192.168.2.254 # main network gateway
  ping google.com    # external connectivity
  ```

## 3. System Preparation

Update and install baseline tools:
```bash
sudo apt update && sudo apt upgrade -y
sudo apt install -y curl wget git htop net-tools vim nano
```

Optional (useful for Kubernetes administration):
```bash
sudo apt install -y tmux unzip
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

## 4. K3s Installation (Master Node)

Install k3s server (control plane):

```bash
curl -sfL https://get.k3s.io | \
  INSTALL_K3S_EXEC="server \
    --node-ip=192.168.2.11 \
    --flannel-iface=enp2s0" \
  sh -
```

This sets up the master node with the Kubernetes control plane and embedded etcd datastore (sufficient for small clusters).

**Note**: because the nodes have multiple network interfaces and are using DHCP we specify the IP address and interface during install

By default, kubectl is configured and available at:

```bash
sudo kubectl get nodes
```

We now want to enable kubectl for the non-root user:

```bash
mkdir -p $HOME/.kube
sudo cp /etc/rancher/k3s/k3s.yaml $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
echo 'export KUBECONFIG=$HOME/.kube/config' >> ~/.bashrc
source ~/.bashrc
```

This will copy the config to the current user's kube config and update your KUBECONFIG env to point to it.

Test you can now use kubectl without sudo:

```bash
kubectl get nodes
```

## 5. Verify Cluster Status

Check that the master node is up and running:

```bash
kubectl get nodes -o wide
kubectl get pods -A
```

You should see the master node in Ready status, along with system pods (CoreDNS, Traefik, metrics, etc.).

## 6. Install Helm for package management

Install Helm for package management:

```bash
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
```

## 7. Retrieve Cluster Join Token

Worker nodes will need a token to join the cluster. Retrieve it with:

```bash
sudo cat /var/lib/rancher/k3s/server/node-token
```

Keep this value secure — you will need it when configuring worker nodes.