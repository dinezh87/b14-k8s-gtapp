# Helm Runbook for This Repository

This repository now includes a complete Helm chart at `gratitudeapp/`.

Use Helm here in two practical ways:
- Manage cluster dependencies (recommended now).
- Deploy this app stack with the included chart.

## 1) Prerequisites

Install:
- `kubectl`
- `helm`
- `aws` CLI and `eksctl` (if using EKS)

Point to your cluster:

```bash
kubectl config current-context
kubectl get nodes
```

## 2) Current Recommended Helm Usage (Dependencies)

This app requires ingress on EKS. Install NGINX Ingress with Helm:

```bash
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update
helm upgrade --install ingress-nginx ingress-nginx/ingress-nginx \
  --namespace ingress-nginx \
  --create-namespace \
  --set controller.service.type=LoadBalancer \
  --set controller.ingressClassResource.name=nginx \
  --set controller.ingressClassByName=true
```

Verify:

```bash
helm list -A
kubectl get pods -n ingress-nginx
kubectl get svc -n ingress-nginx
```

Then deploy this app with Helm:

```bash
helm lint ./gratitudeapp
helm upgrade --install gtapp ./gratitudeapp \
  --namespace default \
  --create-namespace
```

## 3) Chart Location and Overrides

Chart files:
- `gratitudeapp/Chart.yaml`
- `gratitudeapp/values.yaml`
- `gratitudeapp/templates/*`

Example overrides for secrets:

```bash
helm upgrade --install gtapp ./gratitudeapp \
  --namespace default \
  --set openai.secret.apiKey=\"<real-key>\" \
  --set database.secret.password=\"<postgres-password>\"
```

Validate generated manifests:

```bash
helm lint ./gratitudeapp
helm template gtapp ./gratitudeapp > /tmp/gtapp-rendered.yaml
kubectl apply --dry-run=client -f /tmp/gtapp-rendered.yaml
```

## 4) Lifecycle Commands

Rollback:

```bash
helm history gtapp -n default
helm rollback gtapp <REVISION> -n default
```

Uninstall:

```bash
helm uninstall gtapp -n default
```

## 5) Day-2 Helm Operations

See release values:

```bash
helm get values gtapp -n default
helm get manifest gtapp -n default
```

Preview changes before applying:

```bash
helm upgrade --install gtapp ./gratitudeapp \
  --namespace default \
  --dry-run --debug
```

## 6) Notes Specific to This Repo

- `postgres-init-config.yml` mounts SQL init scripts into Postgres; after schema changes, restart Postgres if needed:

```bash
kubectl rollout restart deployment postgres-deployment
kubectl get pods -w
```

- `database-secret.yml` and `openai-api-secret.yml` should be managed securely for real environments (for example via sealed secrets or external secret managers).
- If you use Argo CD, you can point the app path to `gratitudeapp/` and use Helm mode.
