# Infrastructure Deployment Guide

This guide explains how to deploy your infrastructure from scratch, following the "Zero-Knowledge" path to ensure all secrets are managed securely via Vaultwarden and External Secrets Operator (ESO).

## Prerequisites
- A running **k3s** cluster.
- **Helm**, **kubectl**, and **Argo CD** CLI installed.
- Core operators installed as per [k8s/README.md](README.md#2-install-core-operators):
  - `cert-manager`
  - `external-dns`
  - `external-secrets`

---

## Phase 1: Bootstrapping Vaultwarden

Vaultwarden is the source of truth for all other secrets. We must deploy it first using temporary manual secrets.

### 1. Create the Bootstrap Secret
Manually create the secret that Vaultwarden needs to start.
```bash
# Set your namespace (infra or infra-dev)
NAMESPACE="infra-dev"
DOMAIN="vault.dev.lucasduport.cc"

kubectl create secret generic vaultwarden-secret \
  -n $NAMESPACE \
  --from-literal=DOMAIN="https://$DOMAIN" \
  --from-literal=ADMIN_TOKEN="choose-a-very-long-admin-token" \
  --from-literal=DATABASE_URL="postgresql://postgres:changeme@postgres.$NAMESPACE.svc.cluster.local:5432/vaultwarden"
```

### 2. Deploy the "Bootstrap" version of the chart
Deploy the chart with `externalSecrets.enabled` set to **false**. This tells the chart to use the local `vaultwarden-secret` we just created.

```bash
helm upgrade --install infra-dev ./k8s/helm-charts/infra \
  -f ./k8s/helm-charts/infra/values-dev.yaml \
  --set externalSecrets.enabled=false \
  --set vaultwarden.enabled=true \
  --set postgres.enabled=true
```

### 3. Setup your Vaultwarden Account
1. Go to your vault domain (e.g., `https://vault.dev.lucasduport.cc`).
2. Create your account.
3. **Important**: Go to **Account Settings** -> **Security** -> **Keys**.
4. Click **View API Key** and take note of your `client_id` and `client_secret`. You will need these for the next step.

---

## Phase 2: Populating Secrets

### 1. Create the "Bridge Secret"
Now that you have your API keys, create the secret that allows ESO to talk to Vaultwarden.
```bash
kubectl create secret generic vaultwarden-api-key \
  -n $NAMESPACE \
  --from-literal=client_id=YOUR_CLIENT_ID \
  --from-literal=client_secret=YOUR_CLIENT_SECRET
```

### 2. Add Secrets to Vaultwarden
Inside the Vaultwarden web UI, create "Login" entries for each service. Use the **Title** as the path (e.g., `gatus/config`) and add the required fields in the **Custom Fields** section (type "Hidden").

| Entry Title (Key) | Field Name (Property) | Description |
|-------------------|-----------------------|-------------|
| `gatus/config`    | `discord_webhook_all` | Discord webhook URL |
| `vpn/config`      | `password`            | WG-Easy login password |
| `vpn/config`      | `hash`                | Argon2 password hash |
| `lldap/config`    | `jwt_secret`          | Random string for JWT |
| `lldap/config`    | `admin_pass`          | Admin password for LLDAP |

---

## Phase 3: Full Automation

Now that Vaultwarden is populated and the bridge is built, we can enable full automation.

### 1. Switch to External Secrets
Update your deployment to enable ESO. This will switch the source of secrets from local manifests to Vaultwarden.

```bash
helm upgrade --install infra-dev ./k8s/helm-charts/infra \
  -f ./k8s/helm-charts/infra/values-dev.yaml \
  --set externalSecrets.enabled=true
```

### 2. Enable remaining services
You can now enable all other services (Gatus, VPN, LLDAP, etc.) in your `values-dev.yaml` or via Helm `--set`. They will automatically wait for ESO to sync their secrets.

### 3. Connect Argo CD
Once you've verified the manual deployment works, you can point Argo CD to this repository. From then on, any change to `values.yaml` in Git will be automatically applied to the cluster.

---

## Maintenance
- **Adding a new secret**: Simply add it to Vaultwarden. ESO will sync it within 1 hour (default) or immediately if you restart the ESO pods.
- **Updating tokens**: Update the value in Vaultwarden. The app will receive it after the next sync and pod restart.
