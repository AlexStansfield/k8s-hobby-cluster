# Portainer Agent Setup

This document describes how to connect the cluster to **Portainer** as a Kubernetes environment.
The Portainer server itself runs on `dixie` (https://portainer.jinkies.net); the cluster only runs the **Portainer Agent**, which the server connects to over the LAN on NodePort `30778`.

## 1. Prerequisites

- A running **k3s cluster** (master + workers ready).
- **kubectl** configured on the master node.
- A Portainer server somewhere on the same LAN.

## 2. Deploy the agent

The manifest lives in [manifests/portainer/portainer-agent.yaml](../manifests/portainer/portainer-agent.yaml). It is Portainer's generated `portainer-agent-k8s-nodeport.yaml` with the image pinned, and creates:

| Resource | Purpose |
|---|---|
| Namespace `portainer` | Holds everything below |
| ServiceAccount `portainer-sa-clusteradmin` + ClusterRoleBinding to `cluster-admin` | The agent manages the whole cluster on Portainer's behalf |
| Service `portainer-agent-headless` | Peer discovery for the agent's cluster mode |
| Service `portainer-agent` (NodePort `30778` → 9001) | LAN endpoint the Portainer server connects to |
| Deployment `portainer-agent` | One replica of `portainer/agent` |

Apply it:

```bash
kubectl apply -f manifests/portainer/portainer-agent.yaml
```

Verify:

```bash
kubectl -n portainer get pods,svc
```

The pod should be `Running` and the `portainer-agent` Service should show `9001:30778/TCP`.

## 3. Register the environment in Portainer

In the Portainer UI: **Environments → Add environment → Kubernetes → Agent**.

- **Name**: `cluster`
- **Environment address**: `192.168.2.31:30778` (any node IP works; the master is the natural choice since it is the control plane anyway)

Portainer should show the environment as *up* within a few seconds.

## 4. Upgrading the agent

Keep the agent on the same minor version as the Portainer server (**Home → About** in the UI shows the server version). Portainer tolerates some skew, but a large gap eventually breaks the connection.

1. Edit `image:` in [manifests/portainer/portainer-agent.yaml](../manifests/portainer/portainer-agent.yaml).
2. `kubectl apply -f manifests/portainer/portainer-agent.yaml`
3. The Deployment rolls over (a few seconds of downtime for the environment in Portainer).
4. Portainer picks up the new agent version on its next environment snapshot, or immediately if you click **Refresh** on the environment.

Current versions: server **2.45.1**, agent **2.45.0**.

## 5. Why NodePort and not MetalLB

The other LAN-facing services in this cluster get a MetalLB address. The agent deliberately does not:

- **It is the management plane.** If MetalLB is broken you still want to reach the cluster through Portainer to fix it, so the agent should not depend on it.
- **Failover buys nothing here.** MetalLB's L2 mode would move a floating IP to a surviving node, but with a single control-plane node, if `cluster-master` is down Portainer cannot manage the cluster regardless of how it reaches the agent.
- **It is deterministic.** No IP allocation happens on apply, so a rebuild comes up on exactly the same address and the Portainer environment entry keeps working.

This is also how the private registry is exposed (`cluster-master:31234`).

Historical note: until September 2026 the Service was `type: LoadBalancer` and was reached on `192.168.2.31:9001`. That only worked because it had been created before k3s ServiceLB was disabled and still carried the node IPs ServiceLB had assigned; MetalLB never took it over. It was switched to NodePort to make the setup intentional.

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
curl -k https://192.168.2.31:30778/ping
```

The node IP and port must match the environment address configured in Portainer.
