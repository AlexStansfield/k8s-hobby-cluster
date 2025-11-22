# Registry

Deploy a docker image registry to the cluster.

This will give us a private registry to store the images that will be deployed to the nodes

## Overview

- Is deployed via Helm
- Runs only on the master node (cluster-master)
- Uses Longhorn for persistent storage
- Exposes the registry on the LAN (e.g. via NodePort)
- Can be pushed to from the dev machine and pulled from by the cluster

## 1. Add Helm Chart

Use an existing minimal, production-ready registry chart.

```bash
helm repo add twuni https://helm.twun.io
helm repo update
```

## 2. Install the Chart

```bash
helm install private-registry twuni/docker-registry \
  -f registry-values.yaml \
  --namespace registry \
  --create-namespace
```

This will:

- Deploy 1 pod pinned to the master node
- Use Longhorn storage
- Expose the registry at `cluster-master:31234`

## 3. Test the Deployment

Verify:

```bash
kubectl get pods -n registry -o wide
kubectl get svc -n registry
```

Make sure:
- The pod is running
- It's on node `cluster-master`
- The `NodePort` is `31234`

Then test from the dev machine:

```bash
curl http://cluster-master:31234/v2/_catalog
```

You should get a response like:

```json
{"repositories":[]}
```

## 4. Update the k3s nodes to use the registry

We need to add a specific configuration to each node so that it will allow insecure connections to the registry

On each node:

```bash
sudo mkdir -p /etc/rancher/k3s
sudo nano /etc/rancher/k3s/registries.yaml
```

Update the file content

```yaml
mirrors:
  cluster-master:31234:
    endpoint:
      - "http://cluster-master:31234"
```

Restart k3s
- On master
  ```bash
  sudo systemctl restart k3s
  ```
- On worker nodes
  ```bash
  sudo systemctl restart k3s-agent
  ```
