# Infrastructure Deployment Guide

This guide explains how to deploy your **dev** infrastructure from scratch on a fresh k3s node.
Secrets are managed via `my-secrets.yaml` — a local file never committed to Git.

---

## Phase 0: Node Setup

### 1. Install k3s
Run this on your server node. We keep the built-in Traefik (required for `Middleware` CRDs).

```bash
curl -sfL https://get.k3s.io | sh -s - \
  --write-kubeconfig-mode 644
```

Copy the kubeconfig to your local machine so `kubectl` and `helm` can talk to the cluster:

```bash
# On your LOCAL machine
mkdir -p ~/.kube
scp user@<NODE_IP>:/etc/rancher/k3s/k3s.yaml ~/.kube/config
# Replace 127.0.0.1 with the real node IP
sed -i '' "s/127.0.0.1/<NODE_IP>/g" ~/.kube/config
# Verify
kubectl get nodes
```

### 2. Install tooling (macOS)
```bash
brew install helm kubectl argocd
```

---

## Phase 0.5: Install Core Operators

These are one-time cluster-wide installs. Run them from your local machine.

```bash
# Register Helm repos
helm repo add traefik https://helm.traefik.io/traefik
helm repo add cert-manager https://charts.jetstack.io
helm repo add external-dns https://kubernetes-sigs.github.io/external-dns/
helm repo add argo https://argoproj.github.io/argo-helm
helm repo update
```

#### cert-manager
```bash
helm install cert-manager cert-manager/cert-manager \
  -n cert-manager --create-namespace \
  --set crds.enabled=true
```

#### ExternalDNS (Cloudflare)
Replace `YOUR_CF_TOKEN` with your Cloudflare API token (Zone: DNS Edit permission) and `YOUR_PUBLIC_IP` with your box's public IP.

```bash
# Store the token as a secret (never pass tokens as plain Helm values)
kubectl create secret generic cloudflare-api-token \
  -n kube-system \
  --from-literal=CF_API_TOKEN=YOUR_CF_TOKEN

helm install external-dns external-dns/external-dns \
  -n kube-system \
  --set provider=cloudflare \
  --set "env[0].name=CF_API_TOKEN" \
  --set "env[0].valueFrom.secretKeyRef.name=cloudflare-api-token" \
  --set "env[0].valueFrom.secretKeyRef.key=CF_API_TOKEN" \
  --set policy=sync \
  --set txtOwnerId=k3s-dev \
  --set "extraArgs[0]=--default-targets=YOUR_PUBLIC_IP"
```

#### Argo CD
```bash
helm install argocd argo/argo-cd \
  -n argocd --create-namespace \
  --set server.service.type=ClusterIP

# Retrieve the initial admin password
kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath="{.data.password}" | base64 -d && echo
```

> The ArgoCD UI is not exposed by default. Use `kubectl port-forward svc/argocd-server -n argocd 8080:443` to access it locally.

---

## Phase 1: Fill in your secrets

Copy the example file and fill in all values:

```bash
cp ./k8s/helm-charts/infra/my-secrets.example.yaml ./k8s/helm-charts/infra/my-secrets.yaml
# Edit my-secrets.yaml — never commit it (it's in .gitignore)
```

Key things to set:
- `postgres.rootPassword` — shared by all DB-connected services
- `vaultwarden.secrets.ADMIN_TOKEN` — a long random string (or use `openssl rand -hex 32`)
- `lldap.secrets.*`, `gatus.secrets.*`, `vpn.secrets.*`, etc.

> `DOMAIN` and `DATABASE_URL` for Vaultwarden are auto-computed from `ingressDomain`, `hostSuffix` and `postgres.rootPassword` — you do not need to set them manually.
> `POSTGRES_PASSWORD` in `stream-share-secrets` is also auto-injected from `postgres.rootPassword`.

---

## Phase 2: Deploy

```bash
kubectl create namespace infra-dev

helm upgrade --install infra-dev ./k8s/helm-charts/infra \
  -n infra-dev --create-namespace \
  -f ./k8s/helm-charts/infra/values-dev.yaml \
  -f ./k8s/helm-charts/infra/my-secrets.yaml \
  --set vaultwarden.enabled=true \
  --set postgres.enabled=true \
  --set traefik.enabled=true
```

Wait for pods to be ready:
```bash
kubectl -n infra-dev get pods -w
```

Then browse to `https://vault-dev.lucasduport.cc` to verify Vaultwarden is up.

---

## Phase 3: Enable remaining services

Add services one by one or all at once:

```bash
helm upgrade --install infra-dev ./k8s/helm-charts/infra \
  -n infra-dev \
  -f ./k8s/helm-charts/infra/values-dev.yaml \
  -f ./k8s/helm-charts/infra/my-secrets.yaml \
  --set vaultwarden.enabled=true \
  --set postgres.enabled=true \
  --set traefik.enabled=true \
  --set gatus.enabled=true \
  --set lldap.enabled=true \
  --set vpn.enabled=true \
  --set streamShare.enabled=true \
  --set suiviItem.enabled=true
```

---

## Phase 4: Connect Argo CD (GitOps hand-off)
Once you have verified the manual deploy works, hand control to Argo CD so any Git push auto-deploys.

```bash
# Log in (port-forward first if not exposed)
argocd login localhost:8080 --username admin --password <INITIAL_PASSWORD> --insecure

# Apply both Prod and Dev app definitions
kubectl apply -f ./k8s/argocd/applications.yaml
```

Argo CD will now sync `dev/migration_kubernetes` → `infra-dev` namespace automatically.

---

## Maintenance
- **New secret needed**: Add it to `my-secrets.yaml`, then re-run the `helm upgrade` command.
- **Secret rotation**: Update the value in `my-secrets.yaml` and re-run `helm upgrade`. Pods pick up the change on next restart: `kubectl rollout restart deployment/<name> -n infra-dev`.
- **Check deployed secrets**: `kubectl -n infra-dev get secrets`
- **ArgoCD sync status**: `argocd app get infra-dev`
