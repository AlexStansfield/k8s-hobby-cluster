# Kubertnetes Hobby Cluster

This repository documents the architecture, configuration, and custom manifests for my self-hosted Kubernetes cluster.

It serves as both a reference for day-to-day operations and a record for rebuilding the cluster if needed.

## Overview

The cluster is designed as a lightweight but flexible environment for experimenting with containerized workloads, AI services, and home-lab projects.

It prioritizes simplicity, resilience, and reproducibility.

## Hardware

### Controller / Master Node  
- SBC: Radxa X4 (Intel N100, 16 GB RAM)  
- Storage: NVMe M.2 SSD (512 GB)  
- Networking: 2.5 GbE

### Worker Nodes  
- Radxa X4 (same spec as master) × 2–3  
- Each with NVMe M.2 SSD (512 GB)
- 2.5 GbE networking

### Networking & Power

#### Current
- Gigabit switch
- Cat6 cabling across all nodes
- 3x 65W USB chargers

#### Planned
- 2.5 GbE PoE+ switch (120 W total power budget)  
- Cat6 cabling across all nodes  
- Nodes powered via PoE

## Operating System

- **Ubuntu Server 24.04 LTS (Noble Numbat)** on all nodes  
- Minimal server installation with SSH access  
- Consistent user, hostname, and package baseline across the cluster

## Kubernetes Distribution

Using **K3s** (lightweight Kubernetes) - https://k3s.io/
- Chosen for low resource footprint and simplified management  
- Installed in HA-ready mode with an external datastore if needed  
- kubectl and helm used for application deployment

## Storage Solution

Using **Longhorn** - https://longhorn.io/
- Provides distributed block storage across all nodes  
- Replication ensures resilience against node failure  
- Used for stateful workloads such as databases and media services

## Networking

- **Cluster Network**:  
  - Single subnet (`192.168.2.0/24`) shared with other home-lab services  
  - Static DHCP assignments via Pi-hole DHCP server  
  - Master and worker nodes assigned IPs in reserved cluster nodes range (`192.168.2.31–40`)

- **Ingress / Load Balancing**:  
  - K3s built-in Traefik ingress controller (may replace with NGINX ingress later)  
  - Cluster accessible via LAN DNS and reverse proxy  
  - MetalLB planned for future external LoadBalancer support

## Additional Components

- **Helm** for package management  
- **Kubectl** as main management CLI  
- **Portainer** for basic GUI-based management  
- **GitOps** approach with YAML manifests and Helm charts tracked in this repo  

## Repository Contents

- `manifests/` → Custom Kubernetes YAML definitions  
- `helm/` → Helm charts for services and tools  
- `docs/` → Setup guides for master and worker nodes, storage, and networking  
- `README.md` → High-level summary (this file)

## Purpose

This cluster is intended for:  
- Running personal projects and AI workloads  
- Experimenting with Kubernetes, Helm, and GitOps  
- Serving as a reproducible blueprint for rebuilding or expanding the cluster

