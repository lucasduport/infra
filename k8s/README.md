# Kubernetes Infrastructure (k3s)

This folder contains the complete transition from Docker Compose (Portainer) to a native **k3s (Kubernetes)** cluster managed via **GitOps (Argo CD)**.

## High-Level Architecture

| Layer           | Implementation                               | Purpose                                      |
|-----------------|----------------------------------------------|----------------------------------------------|
| Orchestrator    | **k3s** (v1.28+)                             | Lightweight Kubernetes for home servers.      |
| Ingress         | **Traefik** (Native)                         | Handles all HTTP/HTTPS traffic.              |
| GitOps          | **Argo CD**                                  | Automated sync between GitHub and Cluster.   |
| Secret Mgmt     | **External Secrets (ESO)**                   | Imports secrets directly from **Vaultwarden**.|
| DNS Automation  | **ExternalDNS**                              | Auto-updates Cloudflare records for services.|
| SSL/TLS         | **cert-manager**                             | Let's Encrypt (HTTP-01) automation.          |

## Environment Isolation

We maintain two distinct environments on the same cluster using separate namespaces:

- **Production (`infra` namespace)**:
  - Branch: `main`
  - Domain: `*.lucasduport.cc`
  - Secret Store: [vault.lucasduport.cc](https://vault.lucasduport.cc)
- **Development (`infra-dev` namespace)**:
  - Branch: `dev/migration_kubernetes`
  - Domain: `*.dev.lucasduport.cc`
  - Secret Store: [vault.dev.lucasduport.cc](https://vault.dev.lucasduport.cc)

## Infrastructure Setup

### 1. Bootstrap Secrets
Before deploying, you must bootstrap the "Bridge Secrets" so the cluster can talk to your Vaultwarden instance and Cloudflare.
Refer to [k8s/SECRETS.md](SECRETS.md) for detailed instructions.

### 2. Install Core Operators
Run these once on the cluster to enable automation:

```bash
# Register needed Helm repositories
helm repo add external-secrets https://charts.external-secrets.io
helm repo add external-dns https://kubernetes-sigs.github.io/external-dns/
helm repo update

# Install External Secrets Operator
helm install external-secrets external-secrets/external-secrets \
  -n external-secrets --create-namespace --set installCRDs=true

# Install ExternalDNS (Cloudflare)
helm install external-dns external-dns/external-dns \
  -n kube-system --set provider=cloudflare \
  --set env[0].name=CF_API_TOKEN --set env[0].value=YOUR_TOKEN
```

### 3. Deploy via Argo CD
The cluster monitors the repository and automatically manages deployments.
- **Prod Application**: Watches `main` at `k8s/helm-charts/infra` with `values.yaml`.
- **Dev Application**: Watches `dev/migration_kubernetes` at `k8s/helm-charts/infra` with `values-dev.yaml`.

## Application Layout (Helm)
The deployment is structured as a single "App Chart" containing templates for:
- [lldap](helm-charts/infra/templates/lldap/resources.yaml)
- [stream-share](helm-charts/infra/templates/stream-share/resources.yaml)
- [postgres](helm-charts/infra/templates/postgres/resources.yaml)
- [gatus](helm-charts/infra/templates/gatus/resources.yaml)
- [vaultwarden](helm-charts/infra/templates/vaultwarden/resources.yaml)
- [vpn](helm-charts/infra/templates/vpn/resources.yaml)

To enable/disable a service, toggle the `enabled` flag in the relevant `values.yaml` file.

- **Namespace**: `infra` (prod) / `infra-dev` (dev)
- **Experimental Setup**: See [ARCHITECTURE_DEV.md](ARCHITECTURE_DEV.md) for details on exposing the dev environment on a single IP.
- **Storage**: local-path (k3s default)
- **Ingress**: Caddy with Cloudflare DNS challenge (not Traefik)
- **Secrets**: Kubernetes Secrets (managed via Helm values or manually)
- **Deployment**: Helm Chart (`k8s/helm-charts/infra`)

### Service Discovery

Internal services reference each other via **Kubernetes DNS** — no hardcoded IPs needed:

| Service       | Internal DNS                                  | Used by          |
|---------------|-----------------------------------------------|------------------|
| LLDAP (LDAP)  | `lldap.infra.svc.cluster.local:3890`          | stream-share     |
| LLDAP (Web)   | `lldap.infra.svc.cluster.local:17170`         | caddy, gatus     |
| PostgreSQL    | `postgres.infra.svc.cluster.local:5432`       | stream-share     |
| Stream Share  | `stream-share.infra.svc.cluster.local:8080`   | caddy, gatus     |
| Gatus         | `gatus.infra.svc.cluster.local:8080`          | caddy            |
| Caddy         | `caddy.infra.svc.cluster.local:443`           | —                |
| WireGuard UI  | `wg-easy.infra.svc.cluster.local:51821`       | —                |

**Only external services** (like TrueNAS UI) still need IP:port in secrets.

## Prerequisites

### Install k3s (master node — VM on TrueNAS)

```bash
# Install k3s WITHOUT Traefik (we use Caddy instead)
curl -sfL https://get.k3s.io | INSTALL_K3S_EXEC="--disable=traefik" sh -

# Allow unsafe sysctls for WireGuard
cat <<EOF | sudo tee /etc/rancher/k3s/config.yaml
kubelet-arg:
  - "allowed-unsafe-sysctls=net.ipv4.ip_forward,net.ipv6.conf.all.forwarding"
EOF
sudo systemctl restart k3s

# Get the node token for joining workers
sudo cat /var/lib/rancher/k3s/server/node-token
```

### Join Raspberry Pi as worker node

```bash
curl -sfL https://get.k3s.io | K3S_URL=https://<MASTER_IP>:6443 K3S_TOKEN=<NODE_TOKEN> sh -
```

### Label the master node

```bash
kubectl label node <master-node-name> node-role.kubernetes.io/master=true
```

## Deployment

Deployment is performed using **Helm**. Services are **disabled by default**, allowing you to deploy them independently and iterate on your migration.

### 1. Preparation

Copy the default values and create your own secrets:

```bash
cp k8s/helm-charts/infra/values.yaml k8s/helm-charts/infra/my-secrets.yaml
```

Update `my-secrets.yaml` with your real tokens and passwords. **Never commit this file.**

### 2. Basic Usage (Incremental Deployment)

To deploy only **Caddy** (for example):
```bash
helm upgrade --install infra ./k8s/helm-charts/infra \
  -f ./k8s/helm-charts/infra/my-secrets.yaml \
  --set caddy.enabled=true
```

To enable more services, simply append them to the command:
```bash
helm upgrade --install infra ./k8s/helm-charts/infra \
  -f ./k8s/helm-charts/infra/my-secrets.yaml \
  --set caddy.enabled=true \
  --set lldap.enabled=true \
  --set gatus.enabled=true
```

### 3. Production Deployment (Full Stack)

Once everything is ready, you can deploy the full stack by enabling all services in a `values-prod.yaml` or via the command line.

### 4. Development Deployment

```bash
helm upgrade --install infra-dev ./k8s/helm-charts/infra \
  -f ./k8s/helm-charts/infra/values-dev.yaml \
  -f ./k8s/helm-charts/infra/my-secrets.yaml \
  --set caddy.enabled=true \
  --set gatus.enabled=true
```

### 5. Customizing Config

The **Caddyfile** and **Gatus** configurations are externalized in `k8s/helm-charts/infra/config/`.
If you update them, simply re-run the `helm upgrade` command. The pods will automatically restart to apply changes.

### 6. Special manual secrets (Registry)

For `suivi-item` image pulls:
```bash
kubectl create secret docker-registry gitlab-registry-secret -n suivi-item \
  --docker-server=registry.gitlab.com \
  --docker-username=<username> \
  --docker-password=<token>
```

## Cloudflare DNS Setup

### For Development

In Cloudflare DNS for `lucasduport.cc`, add **one** wildcard record:

| Type | Name              | Content           | Proxy  |
|------|-------------------|--------------------|--------|
| A    | `*.dev`           | `<K3S_MASTER_IP>`  | DNS only (grey cloud) |

> **Important**: Use "DNS only" (grey cloud), NOT "Proxied" (orange cloud). Caddy handles TLS itself via the Cloudflare API token — if you proxy through Cloudflare, it will conflict with Caddy's certificate management.

### For Production (migration day)

Update the existing wildcard record to point to the k3s node:

| Type | Name | Content           | Proxy  |
|------|------|--------------------|--------|
| A    | `*`  | `<K3S_MASTER_IP>`  | DNS only |

## Useful Commands

```bash
# Check pod logs
kubectl logs -n infra deployment/caddy
kubectl logs -n infra statefulset/lldap

# Restart a deployment
kubectl rollout restart -n infra deployment/caddy

# Check resource usage
kubectl top pods -n infra

# Shell into a pod
kubectl exec -it -n infra deployment/caddy -- sh
```
