# Backstage Deployment Guide

This document covers the full process to deploy StackLayer Backstage to a Kubernetes cluster using Helm and ArgoCD.

## Prerequisites

- A running Kubernetes cluster with:
  - [NGINX Ingress Controller](https://kubernetes.github.io/ingress-nginx/)
  - [cert-manager](https://cert-manager.io/) with a `selfsigned-cluster-issuer` ClusterIssuer
  - [ArgoCD](https://argo-cd.readthedocs.io/)
  - [local-path StorageClass](https://github.com/rancher/local-path-provisioner) (for PostgreSQL PVC)
- `kubectl` configured against the cluster
- A GitHub Personal Access Token (PAT) with:
  - `read:packages` scope (to pull the image from ghcr.io)
  - `repo` scope (for catalog GitHub integration)
- DNS or `/etc/hosts` entry pointing `backstage.stacklayer.local` to your ingress IP

---

## Step 1 — Create the Backstage namespace

ArgoCD will create the namespace automatically via `CreateNamespace=true`, but you can pre-create it if needed:

```bash
kubectl create namespace backstage
```

---

## Step 2 — Create Kubernetes secrets

These secrets are **not stored in the repo** and must be created manually before deploying.

### App secrets (GitHub token + backend signing secret)

```bash
kubectl create secret generic backstage-secrets \
  --from-literal=GITHUB_TOKEN=<your-github-pat> \
  --from-literal=BACKEND_SECRET=<random-secret-string> \
  -n backstage
```

> `BACKEND_SECRET` can be any long random string. Generate one with:
> ```bash
> openssl rand -base64 32
> ```
> On PowerShell:
> ```powershell
> [Convert]::ToBase64String((1..32 | ForEach-Object { Get-Random -Maximum 256 }) -as [byte[]])
> ```

### Image pull secret (to pull from ghcr.io)

```bash
kubectl create secret docker-registry ghcr-pull-secret \
  --docker-server=ghcr.io \
  --docker-username=liorabitbol \
  --docker-password=<your-github-pat> \
  -n backstage
```

> On PowerShell, replace `\` line continuations with a backtick `` ` ``.

---

## Step 3 — Deploy via ArgoCD

Apply the ArgoCD Application manifest. ArgoCD will pull the Helm chart and values from the repo and deploy everything automatically:

```bash
kubectl apply -f k8s/argocd-app.yaml
```

ArgoCD syncs automatically (`selfHeal: true`, `prune: true`). You can monitor sync status in the ArgoCD UI or via:

```bash
kubectl get application backstage -n argocd
```

---

## Step 4 — Monitor the deployment

Watch the pods come up:

```bash
kubectl get pods -n backstage -w
```

Expected pods:
- `backstage-*` — the Backstage app (should reach `1/1 Running`)
- `backstage-postgresql-*` — PostgreSQL database (should reach `1/1 Running`)

### Common issues

| Symptom | Cause | Fix |
|---|---|---|
| `ImagePullBackOff` | Cluster can't authenticate to ghcr.io | Ensure `ghcr-pull-secret` exists and is referenced in Helm values |
| `CrashLoopBackOff` | Missing secret or bad config | Check logs: `kubectl logs -n backstage <pod>` |
| Pod stuck `Pending` | PVC not bound | Check StorageClass `local-path` is installed |
| Guest login fails | `backend.auth.keys` missing from config | Ensure `app-config.production.yaml` has the `auth.keys` section |

---

## Step 5 — Verify

Once all pods are `Running`, browse to:

```
https://backstage.stacklayer.local
```

Accept the self-signed certificate warning and sign in as Guest.

---

## Rebuilding and pushing the Docker image

If you've made code changes and need to publish a new image:

```bash
# From the repo root
yarn install
yarn build:backend

# Build the Docker image
docker build -t ghcr.io/liorabitbol/backstage-stacklayer:latest packages/backend

# Push to ghcr.io (requires docker login)
docker login ghcr.io -u liorabitbol -p <your-github-pat>
docker push ghcr.io/liorabitbol/backstage-stacklayer:latest
```

ArgoCD will automatically pull the new image on the next sync (or force a sync from the ArgoCD UI).

---

## Key files reference

| File | Purpose |
|---|---|
| `k8s/argocd-app.yaml` | ArgoCD Application resource — triggers the full deployment |
| `helm-values/backstage-values.yaml` | Helm values (image, env vars, ingress, PostgreSQL) |
| `app-config.production.yaml` | Backstage runtime config for production |
| `packages/backend/Dockerfile` | Docker image build definition |
