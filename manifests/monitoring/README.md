# Monitoring Namespace

This directory contains infrastructure manifests for the `monitoring` namespace.

## Contents

- `namespace.yaml` - Creates the monitoring namespace
- `grafana-admin-secret.yaml` - Grafana admin password (gitignored)

## Grafana Admin Secret

The `grafana-admin-secret.yaml` file is **not committed to git** for security.

To create it manually on the cluster:

```bash
kubectl create secret generic grafana-admin-secret \
  --from-literal=admin-password='YOUR_PASSWORD_HERE' \
  --namespace monitoring
```

Or apply the template file after setting your password:

```bash
# Edit grafana-admin-secret.yaml to set your password
kubectl apply -f grafana-admin-secret.yaml
```

## Deployment

Apply the namespace first:

```bash
kubectl apply -f namespace.yaml
```

Then create the secret:

```bash
kubectl apply -f grafana-admin-secret.yaml
```

## Changing Grafana Password

**Option 1: Via Grafana UI** (recommended)
1. Login to Grafana at `http://192.168.2.43:3000`
2. Go to Profile → Change Password
3. Enter current password and new password

**Option 2: Update the Secret**
```bash
# Update the secret
kubectl create secret generic grafana-admin-secret \
  --from-literal=admin-password='NEW_PASSWORD' \
  --namespace monitoring \
  --dry-run=client -o yaml | kubectl apply -f -

# Restart Grafana
kubectl rollout restart deployment grafana -n monitoring
```
