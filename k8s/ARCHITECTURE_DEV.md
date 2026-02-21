# Infrastructure & Domain Strategy

This document explains how the production and development environments are managed and exposed via a single public IP.

## 1. Domain Architecture
To provide a clean separation between environments while staying within Cloudflare's free-tier limitations:

- **Production**: Root-level subdomains (`app.lucasduport.cc`).
- **Development**: Nested subdomains (`app.dev.lucasduport.cc`).

## 2. Ingress & Routing (Traefik)
The k3s cluster uses the native **Traefik Ingress Controller**.
- It listens on ports **80** and **443** (Host Network).
- It routes traffic based on the `Host` header defined in [Ingress resources](helm-charts/infra/templates/traefik/ingress.yaml).

## 3. DNS Automation (ExternalDNS)
We use `external-dns` to synchronize Kubernetes Ingress hosts with Cloudflare DNS records.
- **Provider**: Cloudflare.
- **Policy**: `sync` (automatically creates and deletes records).
- **Proxy Status**: Disabled (Grey Cloud).

> **Why Grey Cloud?**
> Cloudflare's free tier does not support SSL for nested wildcards like `*.dev.lucasduport.cc`. By using "Grey Cloud" (DNS only), we let **Traefik + cert-manager** handle the TLS certificates locally via Let's Encrypt, bypassing Cloudflare's proxy limitations.

## 4. SSL/TLS (cert-manager)
- **Issuer**: Let's Encrypt (Production).
- **Challenge Type**: HTTP-01 (handled automatically via Traefik).

## 5. Traffic Flow
`User` -> `Your Public IP` -> `Router (443)` -> `k3s Node (Traefik)` -> `Namespace (Prod/Dev)` -> `Service Pod`

### 2. Production Caddy Configuration (Gateway)
The Production Caddy (Docker) is configured to match `dev-*` subdomains and forward them to the Kubernetes node on a specific port (**8080**).

**File**: `caddy/mount/Caddyfile`
```plaintext
@dev {
    header_regexp host Host ^dev-.*\.lucasduport\.cc$
}
handle @dev {
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
