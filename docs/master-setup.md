# Master Node Setup

This document describes the process of preparing and configuring the **master (controller) node** for the Kubernetes cluster.

The steps cover system preparation, OS configuration, and installation of the Kubernetes distribution (**k3s**).

## 1. Base OS Installation

- Install **Ubuntu Server 24.04 LTS (Noble Numbat)**.  
- Use a minimal installation (no desktop environment).  
- Ensure SSH is enabled during installation.  
- Set a static hostname, e.g. `cluster-master`.

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
sudo apt install -y curl wget git htop net-tools vim
```

Optional (useful for Kubernetes administration):
```bash
sudo apt install -y nano tmux unzip
```

## 4. K3s Installation (Master Node)

Install k3s server (control plane):

```bash
curl -sfL https://get.k3s.io | sh -
```

This sets up the master node with the Kubernetes control plane and embedded etcd datastore (sufficient for small clusters).

By default, kubectl is configured and available at:

```bash
sudo kubectl get nodes
```

## 5. Retrieve Cluster Join Token

Worker nodes will need a token to join the cluster. Retrieve it with:

```bash
sudo cat /var/lib/rancher/k3s/server/node-token
```

Keep this value secure — you will need it when configuring worker nodes.

## 6. Verify Cluster Status

Check that the master node is up and running:

```bash
sudo kubectl get nodes -o wide
sudo kubectl get pods -A
```

You should see the master node in Ready status, along with system pods (CoreDNS, Traefik, metrics, etc.).

## 7. Post-Setup (Optional)

Enable kubectl for non-root user:

```bash
mkdir -p $HOME/.kube
sudo cp /etc/rancher/k3s/k3s.yaml $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

Install Helm for package management:

```bash
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
```
