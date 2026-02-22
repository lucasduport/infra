# Infrastructure & Domain Strategy

This document explains how the production and development environments are managed and exposed via a single public IP.

## 1. Domain Architecture
Both environments share the same `*.lucasduport.cc` Cloudflare wildcard certificate by using a `-dev` suffix:

- **Production**: Root-level subdomains (`vault.lucasduport.cc`, `ldap.lucasduport.cc`, etc.).
- **Development**: Suffixed subdomains (`vault-dev.lucasduport.cc`, `ldap-dev.lucasduport.cc`, etc.).

This is controlled by the `hostSuffix` value in the Helm chart (`""` for prod, `"-dev"` for dev).

## 2. Ingress & Routing (Traefik)
The k3s cluster uses the native **Traefik Ingress Controller**.
- It listens on port **80** (HTTP only — TLS is terminated upstream by Caddy).
- It routes traffic based on the `Host` header defined in [Ingress resources](helm-charts/infra/templates/traefik/ingress.yaml).

## 3. DNS (Cloudflare)
DNS records for `*-dev.lucasduport.cc` are covered by the existing `*.lucasduport.cc` wildcard — Cloudflare proxy (orange cloud) is fully supported.
- Dev DNS entries point to the same public IP as prod (the Caddy server).
- ExternalDNS can manage them automatically when enabled.

## 4. SSL/TLS (Caddy)
Caddy (running on the prod server) terminates all TLS using Let's Encrypt certificates.
- It forwards dev traffic to the k3s Traefik via plain HTTP on port 80.
- No cert-manager or TLS configuration needed on Traefik for dev.

## 5. Traffic Flow
```
User
  → Cloudflare (HTTPS, *.lucasduport.cc wildcard cert)
  → Public IP → Router (port forward 443 → 192.168.1.73)
  → Caddy (TLS termination, Let's Encrypt)
  → Traefik k3s (HTTP port 80, 192.168.1.201)
  → Service Pod (infra-dev namespace)
```
    reverse_proxy 192.168.1.201:8080 {
        header_up Host {host}
        header_up X-Forwarded-Proto {scheme}
    }
}
```

### 3. Development Caddy Configuration (K8s)
The Development Caddy (Helm) listens on port **8080** on the K8s node's host network.

**Key Settings in `values-dev.yaml`**:
```yaml
caddy:
  httpPort: 8080  # Matches the port Prod Caddy is looking for
  httpsPort: 8443 # Alternative port for direct access if needed
```

**Breaking the Redirect Loop**:
Since Prod Caddy terminates TLS and talks to Dev Caddy via HTTP, we must disable automatic HTTPS redirects in the Dev Caddy to prevent an infinite loop:
```plaintext
{
  auto_https disable_redirects
}
```

## Troubleshooting

### Custom 404 Messages
To quickly identify which Caddy instance is responding, we've customized the 404 responses:
*   **"404 Not Found (Production Caddy)"**: The request reached your home but stop at the first proxy. Check the regex matcher in the Prod Caddyfile.
*   **"404 Not Found (Dev Kubernetes Caddy)"**: The request successfully traversed the chain but the Dev Caddy doesn't have a matching handler for that specific host.

### Port Forwarding Check
If you get a 502 Bad Gateway:
1.  Verify the K8s node IP in the Production Caddyfile.
2.  Ensure port 8080 is open on the K8s node.
3.  Check if the `caddy` pod in `infra-dev` namespace is `Running`.

## Summary Flow
`User` -> `Router (443)` -> `Prod Caddy (Docker)` -> `K8s Node (8080)` -> `Dev Caddy (Pod)` -> `Backend Service`
