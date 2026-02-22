# Secret Management

Secrets are managed via `my-secrets.yaml` — a local file loaded as a second Helm values file at deploy time. It is **never committed** to Git (listed in `.gitignore`).

---

## How it works

The Helm chart generates plain Kubernetes `Secret` resources directly from the values in `my-secrets.yaml`. Pass it as a second `-f` flag at deploy time — it never touches the repo.

---

## Setup

Copy the example and fill in your values:

```bash
cp ./k8s/helm-charts/infra/my-secrets.example.yaml ./k8s/helm-charts/infra/my-secrets.yaml
```

Then deploy passing both values files:

```bash
helm upgrade --install infra-dev ./k8s/helm-charts/infra \
  -n infra-dev \
  -f ./k8s/helm-charts/infra/values-dev.yaml \
  -f ./k8s/helm-charts/infra/my-secrets.yaml \
  --set vaultwarden.enabled=true \
  --set postgres.enabled=true
```

---

## Auto-computed secrets

These values are **not** set in `my-secrets.yaml` — the chart computes them automatically:

| Secret | Key | Computed from |
|---|---|---|
| `vaultwarden-secret` | `DOMAIN` | `ingressDomain` + `hostSuffix` |
| `vaultwarden-secret` | `DATABASE_URL` | `postgres.rootPassword` + `infraNamespace` |
| `stream-share-secrets` | `POSTGRES_PASSWORD` | `postgres.rootPassword` |

---

## Secret → Kubernetes name mapping

| `my-secrets.yaml` section | K8s Secret name | Namespace |
|---|---|---|
| `postgres.rootPassword` | `postgres-secrets` | `infraNamespace` |
| `vaultwarden.secrets.*` | `vaultwarden-secret` | `infraNamespace` |
| `gatus.secrets.*` | `gatus-secrets` | `infraNamespace` |
| `lldap.secrets.*` | `lldap-secrets` | `infraNamespace` |
| `vpn.secrets.*` | `vpn-secrets` | `infraNamespace` |
| `streamShare.secrets.*` | `stream-share-secrets` | `infraNamespace` |
| `suiviItem.secrets.*` | `suivi-item-secret` | `suiviItemNamespace` |
