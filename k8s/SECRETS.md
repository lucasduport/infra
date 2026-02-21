# Secret Management Strategy: Dual Vaultwarden (Prod/Dev)

Since you have two separate Vaultwarden instances (Prod and Dev), the Helm chart is configured to talk to the correct one based on which environment is being deployed.

## 1. Domain Routing
- **Prod Deployment**: Connects to `https://vault.lucasduport.cc`
- **Dev Deployment**: Connects to `https://vault.dev.lucasduport.cc`

This is handled automatically by the `ingressDomain` variable in your values files.

## 2. One-Time Setup per Namespace
Because each Vaultwarden instance has a different API Key, and secret stores are isolated per namespace, you must create the "Bridge Secret" in **every namespace** that needs secrets.

### A. For the Production Environment
Run these on your cluster:
```bash
# Create for the main apps
kubectl create secret generic vaultwarden-api-key \
  -n infra \
  --from-literal=client_id=PROD_CLIENT_ID \
  --from-literal=client_secret=PROD_CLIENT_SECRET

# Create for the suivi-item app
kubectl create secret generic vaultwarden-api-key \
  -n suivi-item \
  --from-literal=client_id=PROD_CLIENT_ID \
  --from-literal=client_secret=PROD_CLIENT_SECRET
```

### B. For the Development Environment
Run these on your cluster:
```bash
# Create for the dev apps
kubectl create secret generic vaultwarden-api-key \
  -n infra-dev \
  --from-literal=client_id=DEV_CLIENT_ID \
  --from-literal=client_secret=DEV_CLIENT_SECRET

# Create for the suivi-item-dev app
kubectl create secret generic vaultwarden-api-key \
  -n suivi-item-dev \
  --from-literal=client_id=DEV_CLIENT_ID \
  --from-literal=client_secret=DEV_CLIENT_SECRET
```

## 3. Vault Entry Structure
Your secrets in **both** Vaultwarden instances should follow the same path/naming convention:
- `gatus/config` (fields: `discord_webhook_all`)
- `postgres/config` (fields: `root_password`)
- `vpn/config` (fields: `password`, `hash`)
- `lldap/config` (fields: `jwt_secret`, `key_seed`, `admin_pass`)

## 4. Why this setup?
- **Isolation**: Dev secrets never touch the Prod cluster/namespace.
- **Portability**: You can rebuild the Dev environment with its own vault without affecting Prod.
- **Public Repo Security**: Your repo remains 100% clean of all credentials.
