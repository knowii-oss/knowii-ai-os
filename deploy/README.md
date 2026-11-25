# Deployment

This directory is the **reference deployment** of Knowii AI OS — it is both
how the managed SaaS runs in production and how self-hosters run their own
instance. One artifact, two audiences, by design.

## Topology

One VPS (production: Hetzner CX32, EU). Everything runs as Docker containers
bound to loopback; **nginx on the host** terminates TLS and routes each domain
to its container:

| Domain (production) | Container | Cloudflare | TLS |
|---|---|---|---|
| `os.knowii.net` | `web` (Web UI) | proxied | Cloudflare origin cert |
| `os-api.knowii.net` | `engine` (API + live rail WS/SSE) | proxied | Cloudflare origin cert |
| `git.knowii.net` | `forgejo` (vault git remotes) | **DNS-only** | certbot / Let's Encrypt |
| — | `postgres` | not exposed | — |

Why git is not proxied: Cloudflare's free-plan 100 MB request-body cap breaks
pushes of attachment-heavy vaults. The git vhost therefore does its own TLS
and streams uploads (`client_max_body_size 0`, request buffering off).

Self-hosters: replace `knowii.net` domains in `nginx/*.conf` with your own,
and if you don't use Cloudflare, point the two origin-cert vhosts at certbot
paths like the git vhost does.

## Files

- `docker-compose.yml` — the services (engine, web, postgres, forgejo)
- `nginx/*.conf` — one vhost per domain, drop into `/etc/nginx/sites-enabled/`
- `.env.example` — configuration template (copy to `.env`)

## Storage decisions baked in

- **Forgejo LFS goes to S3-compatible object storage from day one** (no
  local-disk phase — migrating LFS later is painful).
- **Backups**: nightly `restic` (postgres dump + Forgejo data + configs) to
  Cloudflare R2 — an off-provider copy — plus weekly VPS snapshots.
  If LFS lives on Hetzner Object Storage, the restic job must cover the LFS
  bucket too, to keep the off-provider guarantee.

## Deploying / updating

```sh
cp .env.example .env   # then fill it in
docker compose pull
docker compose up -d
nginx -t && systemctl reload nginx
```

A proper `deploy.sh` (pull, migrate, health-check, reload) lands with the
engine's first release.
