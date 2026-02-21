# Infra

Modern Kubernetes GitOps for personal services.

## Architecture
- **Orchestration**: k3s (Kubernetes)
- **Ingress/Proxy**: Traefik (native k3s ingress controller)
- **GitOps**: Argo CD (Syncing `main` and `dev/migration_kubernetes` branches)
- **Secret Management**: External Secrets Operator (ESO) + Vaultwarden (Bitwarden API)
- **DNS Automation**: ExternalDNS + Cloudflare (Automatic A/CNAME updates)
- **Certificates**: cert-manager (Let's Encrypt / HTTP-01 challenge)

## Structure
- `k8s/` — Kubernetes manifests & Helm charts (Source of truth)
- `archives/` — Old Docker Compose files (No longer actively used)

## Getting Started
See the [Kubernetes documentation](k8s/README.md) for cluster setup and deployment instructions.
For secret management details, refer to [SECRETS.md](k8s/SECRETS.md).

---
*Historically managed via Docker Compose & Portainer. Migration to k3s completed in Feb 2026.*
