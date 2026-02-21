# Kubernetes Migration

Migration from Docker Compose (Portainer) to Kubernetes (k3s).

## Architecture

```
k3s cluster
├── node: vm-truenas (master + workloads)
│   ├── caddy        (Deployment — reverse proxy / ingress)
│   ├── lldap        (StatefulSet — LDAP authentication)
│   ├── stream-share (Deployment — IPTV proxy)
│   ├── postgres     (StatefulSet — PostgreSQL for stream-share)
│   ├── wg-easy      (Deployment — WireGuard VPN, hostNetwork)
│   └── gatus        (Deployment — uptime monitoring)
│
└── node: raspi (worker — lightweight services / overflow)
```

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
