# Home Assistant Integration Setup

This document describes how Home Assistant (on `dixie`, http://dixie.jinkies.net:8123) monitors and controls the cluster through the [tibuntu/homeassistant-kubernetes](https://github.com/tibuntu/homeassistant-kubernetes) custom integration (installed in HA via HACS).
Nothing runs in the cluster for this: HA talks to the API server at `https://192.168.2.31:6443` using a dedicated ServiceAccount token.

## 1. Prerequisites

- **metrics-server** (bundled with k3s) for live node and pod CPU/memory. Without it the integration shows capacity only.
- Upstream tests against Kubernetes 1.35–1.37. The cluster runs 1.33; the integration only uses GA APIs, so it works, but that version is outside upstream's tested range.

## 2. Deploy the RBAC

The manifests live in [manifests/homeassistant/](../manifests/homeassistant/). They are upstream's `manifests/full/` from release **v1.12.0**, vendored unchanged, plus a Namespace:

| Resource | Purpose |
|---|---|
| Namespace `homeassistant` | Holds the ServiceAccount and its token |
| ServiceAccount `homeassistant-kubernetes-integration` | Identity HA authenticates as |
| Secret `homeassistant-kubernetes-integration-token` | Long-lived token for that ServiceAccount (filled in by Kubernetes) |
| ClusterRole + ClusterRoleBinding `homeassistant-kubernetes-integration` | "Full" mode: read and watch pods, nodes, workloads, jobs, events, services, ingresses and metrics; patch and scale deployments/statefulsets/daemonsets; cordon nodes; delete pods and jobs; create jobs from CronJobs. No access to Secrets and no creating workloads. |

The files are numbered so the ServiceAccount exists before its token Secret: Kubernetes deletes a
`service-account-token` Secret whose ServiceAccount does not exist yet.

```bash
kubectl apply -f manifests/homeassistant/00-namespace.yaml
kubectl apply -f manifests/homeassistant/
```

Verify:

```bash
SA=system:serviceaccount:homeassistant:homeassistant-kubernetes-integration
kubectl auth can-i watch pods -A --as=$SA              # yes
kubectl auth can-i patch deployments/scale -A --as=$SA # yes
kubectl auth can-i get secrets -A --as=$SA             # no
```

## 3. Configure the integration in Home Assistant

Get the token (paste it straight into HA; don't store it anywhere else):

```bash
kubectl get secret homeassistant-kubernetes-integration-token -n homeassistant \
  -o jsonpath='{.data.token}' | base64 -d
```

In HA: **Settings → Devices & services → Add integration → Kubernetes**, with host `192.168.2.31`, port `6443`, the token, all namespaces, and SSL verification on using the k3s server CA (`kubectl config view --minify --raw -o jsonpath='{.clusters[0].cluster.certificate-authority-data}' | base64 -d`).

## 4. Revoking access

Cut HA off immediately without removing anything else:

```bash
kubectl delete clusterrolebinding homeassistant-kubernetes-integration
```

To rotate the token, delete the Secret and re-apply `02-serviceaccount-token-secret.yaml`, then update the token in HA's integration settings.

## 5. Upgrading

When updating the integration in HACS, check upstream's `manifests/full/clusterrole.yaml` at the new release tag for new permissions, update the vendored copy and the version in the file headers, then `kubectl diff` and `apply`.
