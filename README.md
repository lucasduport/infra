# Infra

Minimal Docker GitOps for personal services.

## Repo layout
- caddy/ — reverse proxy (Caddy)
- lldap/ — LDAP server (LLDAP)
- stream-share/ — IPTV proxy app + Postgres
- vpn/ — WireGuard VPN with web UI (wg-easy)
- gatus/ — Uptime monitoring
- archives/ — deprecated/old compose files

## Secrets & env
- Keep all secrets out of Git. Use .env.example for placeholders only.
- Set real values in Portainer’s Environment tab per stack.