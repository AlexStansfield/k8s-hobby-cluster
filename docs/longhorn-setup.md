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

## 8. Snapshot Policy

A recurring job `c-89s9j4` (created in the Longhorn UI, not in this repo) takes a **snapshot of every volume in the `default` group daily at 00:00 and keeps 7**. Volumes with no recurring-job labels fall into `default` automatically. These are **local snapshots only**: no backup target is configured, so they protect against accidental deletion or corruption, but not against losing the data on all replicas.

**Telemetry volumes are excluded.** Prometheus, Loki and Tempo rewrite their data constantly (compaction, retention), so each daily snapshot pinned several GiB of already-deleted blocks. By 2026-10-08 the 50 GiB `prometheus-server` volume held 83 GiB per replica for 11 GiB of real data. These three volumes are moved to a group with no jobs:

```bash
for V in <prometheus-server pv> <storage-loki-0 pv> <storage-tempo-0 pv>; do
  kubectl label volumes.longhorn.io -n longhorn-system $V \
    recurring-job-group.longhorn.io/default- \
    recurring-job-group.longhorn.io/no-snapshots=enabled --overwrite
done
```

They also have `unmapMarkSnapChainRemoved: enabled` (the volume's *Remove Snapshots During Filesystem Trim*), so a filesystem trim releases the last removed snapshot as well. Reclaiming the space was: delete the volume's snapshots (`kubectl delete snapshots.longhorn.io -l longhornvolume=<pv>`), wait for the purge to finish (`engines.longhorn.io` `status.purgeStatus`), then run **Trim Filesystem** on the volume. Result: prometheus-server 83.4 → 12.6 GiB, loki 15.8 → 2.0 GiB, tempo 1.6 → 0.3 GiB, about 225 GiB freed across the three nodes.

> These labels live on the Longhorn `Volume` objects, not in a manifest. If one of these PVCs is ever recreated, the new volume lands back in `default` and needs relabelling. The durable fix is to set the labels on the PVCs through each chart's persistence settings with `recurring-job.longhorn.io/source: enabled`. That needs the Prometheus Helm values in this repo reconciled with the deployed release first (they have drifted).
