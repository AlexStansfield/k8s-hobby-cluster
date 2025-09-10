# Longhorn Setup

This document describes how to deploy and configure **Longhorn** as the distributed block storage solution for the cluster.  
Longhorn provides replicated, fault-tolerant volumes that can be used by Kubernetes workloads.

## 1. Prerequisites

- A running **k3s cluster** (master + workers ready).  
- **kubectl** configured and working from the master node.  
- Each node has:  
  - At least one dedicated disk (NVMe / SSD recommended).
  - Disk has parition mounted at `/mnt/data`.
  - Sufficient free space for storage (at least 250GB in `/mnt/data`).
  - Static IP within the cluster range (`192.168.2.31–40`).  

## 2. Install iSCSI Packages (All Nodes)

Longhorn requires **iSCSI initiator tools** to attach volumes.  
Run these steps on **every node** (master + workers):

```bash
sudo apt update
sudo apt install -y open-iscsi
```

Enable and start the service:

```bash
sudo systemctl enable iscsid
sudo systemctl start iscsid
```

Verify it is active:

```bash
systemctl status iscsid
```

## 3. Install Longhorn via Helm

Add the Longhorn Helm repo:

```bash
helm repo add longhorn https://charts.longhorn.io
helm repo update
```

## 4. Verify Installation

Check that all Longhorn pods are running:

```bash
kubectl -n longhorn-system get pods
```

You should see pods like longhorn-manager, longhorn-driver-deployer, and longhorn-ui in Running state.

## 5. Access Longhorn UI

Forward the Longhorn UI service to your local machine.

From your workstation, create an SSH tunnel to the cluster master:

```bash
ssh -L 8080:localhost:8080 user@cluster-master
```

Then, on the master node, run:

```bash
kubectl -n longhorn-system port-forward svc/longhorn-frontend 8080:80
```

Now open:
👉 http://localhost:8080
 in your local browser.

From here you can view nodes, disks, volumes, and replica status.

## 6. Configure Storage Disks

By default, Longhorn auto-adds /var/lib/longhorn as a storage path on each node.
In this cluster we want to make use of the partition mounted at `/mnt/data`.

Steps in the Longhorn UI (Settings → Node → Disks):

1. Remove the default /var/lib/longhorn storage entry.
2. Add a new disk with the path /mnt/data.
  - Example size: 250 GB per node.
  - Enable scheduling for this disk.

After applying, each node should show `/mnt/data` as the active Longhorn storage location.

## 7. StorageClass & Default Configuration

Longhorn installs its own StorageClass (longhorn) automatically.
To make it the default storage class:

```bash
kubectl patch storageclass longhorn \
  -p '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'
```

Verify:

```bash
kubectl get storageclass
```
