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

## Importing stacks via Portainer (workaround)
- Use Portainer Enterprise (free up to 3 nodes) to enable Relative path volumes.
- On the target environment (endpoint) in Portainer: Settings -> Security & Volumes:
	- Enable “Relative path volumes” and set the Stack base path to the same folder.
- Deploy stacks from Git, selecting the compose file (e.g., `caddy/docker-compose.yml`, `gatus/docker-compose.yml`, etc.).
	- Any bind like `./caddy/mount:/etc/caddy` will resolve to `<Stack base path>/caddy/` on the host. In my case: `/data/compose/portainer-compose-unpacker/stacks/caddy/`
- Docs: https://docs.portainer.io/advanced/relative-paths