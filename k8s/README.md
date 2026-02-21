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

### 1. Bootstrap Phase
Before you can automate everything, you must deploy Vaultwarden manually to generate your API keys.
Refer to the **[Step-by-Step Deployment Guide](DEPLOYMENT.md)** for detailed instructions on the initial bootstrap.

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

## Service Discovery

Internal services reference each other via **Kubernetes DNS** — no hardcoded IPs needed:

| Service       | Internal DNS                                  | Used by          |
|---------------|-----------------------------------------------|------------------|
| LLDAP (LDAP)  | `lldap.infra.svc.cluster.local:3890`          | stream-share     |
| LLDAP (Web)   | `lldap.infra.svc.cluster.local:17170`         | gatus            |
| PostgreSQL    | `postgres.infra.svc.cluster.local:5432`       | stream-share     |
| Stream Share  | `stream-share.infra.svc.cluster.local:8080`   | gatus            |
| Gatus         | `gatus.infra.svc.cluster.local:8080`          | —                |

## Special Manual Secrets (Registry)

For `suivi-item` image pulls, we still need a manual registry secret in its namespace:
```bash
kubectl create secret docker-registry gitlab-registry-secret -n suivi-item-dev \
  --docker-server=registry.gitlab.com \
  --docker-username=<username> \
  --docker-password=<token>
```

## Cloudflare DNS Automation

With **ExternalDNS** installed, you no longer need to manually manage DNS records in the Cloudflare dashboard.

### How it works:
1. When you enable a service (e.g., `gatus`), an **Ingress** resource is created with a host like `status.dev.lucasduport.cc`.
2. **ExternalDNS** detects this Ingress and automatically creates the corresponding `A` record in Cloudflare pointing to your public IP.
3. If you disable the service or change the host, ExternalDNS will automatically update or delete the record.

### Proxy Status (Grey Cloud)
By default, records are created as **DNS Only (Grey Cloud)**.
- This is necessary for `*.dev.lucasduport.cc` because Cloudflare's free tier only supports SSL for one level of subdomains.
- **Traefik + cert-manager** handles the SSL certificates locally on your cluster.

## Useful Commands

```bash
# Check ExternalDNS logs to see sync status
kubectl logs -n kube-system -l app.kubernetes.io/name=external-dns

# Check Traefik ingress status
kubectl get ingress -A

# Check cert-manager certificates
kubectl get certificate -A
```
