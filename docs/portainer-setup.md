# Portainer Agent Setup

This document describes how to connect the cluster to **Portainer** as a Kubernetes environment.
The Portainer server itself runs on `dixie` (https://portainer.jinkies.net); the cluster only runs the **Portainer Agent**, which the server connects to over the LAN on port 9001.

## 1. Prerequisites

- A running **k3s cluster** (master + workers ready).
- **kubectl** configured on the master node.
- **MetalLB** installed ([metallb-setup.md](metallb-setup.md)) so the agent can get a LAN IP.
- A Portainer server somewhere on the same LAN.

## 2. Deploy the agent

The manifest lives in [manifests/portainer/portainer-agent.yaml](../manifests/portainer/portainer-agent.yaml). It is Portainer's generated `portainer-agent-k8s-lb.yaml` with the image pinned, and creates:

| Resource | Purpose |
|---|---|
| Namespace `portainer` | Holds everything below |
| ServiceAccount `portainer-sa-clusteradmin` + ClusterRoleBinding to `cluster-admin` | The agent manages the whole cluster on Portainer's behalf |
| Service `portainer-agent-headless` | Peer discovery for the agent's cluster mode |
| Service `portainer-agent` (LoadBalancer, port 9001) | LAN endpoint the Portainer server connects to |
| Deployment `portainer-agent` | One replica of `portainer/agent` |

Apply it:

```bash
kubectl apply -f manifests/portainer/portainer-agent.yaml
```

Verify:

```bash
kubectl -n portainer get pods,svc
```

The pod should be `Running` and the `portainer-agent` Service should show an `EXTERNAL-IP`.

## 3. Register the environment in Portainer

In the Portainer UI: **Environments → Add environment → Kubernetes → Agent**.

- **Name**: `cluster`
- **Environment address**: `<EXTERNAL-IP>:9001` from the step above

Portainer should show the environment as *up* within a few seconds.

## 4. Upgrading the agent

Keep the agent on the same minor version as the Portainer server (**Home → About** in the UI shows the server version). Portainer tolerates some skew, but a large gap eventually breaks the connection.

1. Edit `image:` in [manifests/portainer/portainer-agent.yaml](../manifests/portainer/portainer-agent.yaml).
2. `kubectl apply -f manifests/portainer/portainer-agent.yaml`
3. The Deployment rolls over (a few seconds of downtime for the environment in Portainer).
4. Portainer picks up the new agent version on its next environment snapshot, or immediately if you click **Refresh** on the environment.

Current versions: server **2.45.1**, agent **2.45.0**.

## 5. Note on the LoadBalancer IP

The live `portainer-agent` Service was first applied on 2025-09-08, **before** k3s's built-in ServiceLB was disabled for MetalLB. Its status still carries the node IPs (`192.168.2.31–33`) that ServiceLB assigned. MetalLB does not touch a Service that already has an external IP, and kube-proxy keeps routing those IPs, so the Portainer environment is registered as `192.168.2.31:9001` and works — but only by accident of history.

If the Service is ever deleted and recreated (including on a cluster rebuild), MetalLB will assign an IP from its pool (`192.168.2.41–60`) instead, and the Portainer environment address will need updating to match.

To make this deterministic, pin an address from the MetalLB pool in the manifest, as the other LoadBalancer services in this repo do:

```yaml
spec:
  type: LoadBalancer
  loadBalancerIP: 192.168.2.48   # pick a free one from the pool
```

then delete and re-apply the Service (`kubectl -n portainer delete svc portainer-agent`, then `kubectl apply -f …`) so MetalLB takes ownership, and update the environment address in Portainer.

## 6. Troubleshooting

**Environment shows as down in Portainer:**

```bash
kubectl -n portainer get pods
kubectl -n portainer logs deploy/portainer-agent
```

The agent logs at `DEBUG` level by default (set via `LOG_LEVEL` in the manifest); look for the server's connection attempts.

**Agent up but Portainer can't reach it:**

```bash
kubectl -n portainer get svc portainer-agent
curl -k https://<EXTERNAL-IP>:9001/ping
```

The IP shown must match the environment address configured in Portainer (see section 5).
