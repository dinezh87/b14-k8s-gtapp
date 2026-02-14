# Argo CD Runbook on EKS for This Repository

This document explains how to run Argo CD on an AWS EKS cluster and deploy this app manifests to that EKS cluster.

## 1) Prerequisites

- AWS CLI, `eksctl`, `kubectl`, `helm`, and `argocd` CLI installed.
- An EKS cluster already created (example in `README.md` uses `b15-pk-eks` in `ap-south-1`).
- Your kubeconfig pointing to that EKS cluster.

Set kubeconfig to EKS and verify:

```bash
aws eks update-kubeconfig --region ap-south-1 --name b15-pk-eks
kubectl config current-context
kubectl get nodes
```

## 2) EKS Add-ons Required by This App

This repository uses:
- NGINX ingress (`ingress-service.yml`)
- EBS-backed PVC (`database-persistent-volume-claim.yml`)

Install/configure on EKS:

```bash
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update
helm upgrade --install ingress-nginx ingress-nginx/ingress-nginx \
  -n ingress-nginx --create-namespace \
  --set controller.service.type=LoadBalancer \
  --set controller.ingressClassResource.name=nginx \
  --set controller.ingressClassByName=true
```

```bash
eksctl create addon --cluster b15-pk-eks --region ap-south-1 --name aws-ebs-csi-driver --force
kubectl get csidrivers
```

## 3) Install Argo CD on EKS

```bash
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
kubectl get pods -n argocd -w
```

Wait until Argo CD pods are `Running`.

## 4) Access Argo CD UI/API

Use port-forward:

```bash
kubectl port-forward svc/argocd-server -n argocd 8080:443
```

Then open `https://localhost:8080`.

## 5) Login to Argo CD

Get the initial admin password:

```bash
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 --decode; echo
```

Login with CLI:

```bash
argocd login localhost:8080 --username admin --insecure
```

## 6) Prepare Repository Secrets

Before sync, update these files with real values:

- `openai-api-secret.yml` (`OPENAI_API_KEY`)
- `database-secret.yml` (`PGPASSWORD`)

Commit and push those changes to your Git repo branch used by Argo CD.

## 7) Create Argo CD Application (Target: EKS)

Run this from your machine (replace values in angle brackets):

```bash
argocd app create gtapp \
  --repo <YOUR_GIT_REPO_URL> \
  --path . \
  --revision <BRANCH_OR_TAG> \
  --dest-server https://kubernetes.default.svc \
  --dest-namespace default \
  --directory-recurse
```

Sync:

```bash
argocd app sync gtapp
argocd app wait gtapp --health --sync
```

## 8) Corresponding Kubernetes Files (This Repo)

Argo CD will apply all manifests under this folder. Logical order/dependency:

1. `storageclass-gp3-default.yml`
2. `database-persistent-volume-claim.yml`
3. `database-secret.yml`
4. `openai-api-secret.yml`
5. `postgres-init-config.yml`
6. `postgres-deployment.yml`
7. `postgres-cluster-ip-service.yml`
8. `entries-deployment.yml`
9. `entries-cluster-ip-service.yml`
10. `moods-service-deployment.yml`
11. `moods-service-cluster-ip-service.yml`
12. `stats-service-deployment.yml`
13. `stats-service-cluster-ip-service.yml`
14. `server-deployment.yml`
15. `server-cluster-ip-service.yml`
16. `files-service-service-account.yml`
17. `files-service-deployment.yml`
18. `files-service-cluster-ip-service.yml`
19. `moods-api-deployment.yml`
20. `moods-api-cluster-ip-service.yml`
21. `stats-api-deployment.yml`
22. `stats-api-cluster-ip-service.yml`
23. `api-gateway-deployment.yml`
24. `api-gateway-cluster-ip-service.yml`
25. `client-deployment.yml`
26. `client-cluster-ip-service.yml`
27. `client-service.yml`
28. `ingress-service.yml`

After first database init or schema updates, restart Postgres if needed:

```bash
kubectl rollout restart deployment postgres-deployment
kubectl get pods -w
```

## 9) Verify on EKS

```bash
kubectl get nodes
kubectl get deployments
kubectl get services
kubectl get ingress
argocd app get gtapp
```
