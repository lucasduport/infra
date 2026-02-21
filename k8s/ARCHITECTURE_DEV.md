# Development Environment Architecture

This document explains how the development environment (`infra-dev`) is exposed to the internet alongside the production environment, given the limitation of a single public IP and a single port 443.

## The Challenge
*   **Single Public IP**: The router can only forward port 443 to one internal IP.
*   **Port Conflict**: Port 443 is already used by the Production Caddy (running in Docker).
*   **Domain Depth**: Cloudflare's free SSL certificates do not support 2nd-level wildcards (e.g., `*.dev.lucasduport.cc` fails).

## The Solution: Chained Caddy Reverse Proxy

We use a "Gateway" architecture where the Production Caddy forwards traffic for development subdomains to the Kubernetes node.

### 1. DNS Strategy (Flat Subdomains)
To stay within Cloudflare's SSL limits, we use a prefix instead of a nested subdomain:
*   **Production**: `status.lucasduport.cc`
*   **Development**: `dev-status.lucasduport.cc`

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
