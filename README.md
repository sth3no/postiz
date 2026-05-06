# Postiz — Portainer + Git Auto-Deploy

A parameterized Postiz self-host stack designed to be deployed as a Portainer
**Stack from Repository** (git auto-deploy). Secrets and per-environment
values live in Portainer's stack environment variables, never in git.

## Layout

- `docker-compose.yaml` — the stack (Postiz + Postgres + Redis + Temporal)
- `.env.example` — every env var Portainer needs (paste into Portainer's env UI)
- `dynamicconfig/` — Temporal dynamic config (must ship with the compose file)

## What you'll need

- A VPS with Docker + Portainer installed and reachable
- A git repo (private or public) Portainer can pull from
- A domain or subdomain pointing at the VPS (e.g. `postiz.example.com`)
- A reverse proxy on the VPS terminating HTTPS (nginx / Caddy / Traefik).
  This compose binds Postiz to `127.0.0.1:4007` by default — set
  `POSTIZ_BIND=0.0.0.0` only if you really want to expose it directly.

## Step 1 — Push this folder to your git host

From this directory:

```bash
git init -b main
git add .
git commit -m "Postiz self-host stack"
git remote add origin <your-git-url>
git push -u origin main
```

Anything matched by `.gitignore` (real `.env` files, secret JSON) stays local.

## Step 2 — Create the stack in Portainer

1. Portainer → **Stacks** → **Add stack**
2. Build method: **Repository**
3. Repository URL: your git URL (use HTTPS + a deploy token for private repos)
4. Repository reference: `refs/heads/main`
5. Compose path: `docker-compose.yaml`
6. **Enable** *Automatic updates* — pick polling (e.g. every 5m) or use the
   webhook URL Portainer generates and add it to your git host

## Step 3 — Set environment variables

In the same stack form, paste the contents of `.env.example` into the
**Environment variables** → **Advanced mode** box, then fill in values.

Required to boot:

| Variable | Example |
| --- | --- |
| `MAIN_URL` | `https://postiz.example.com` |
| `FRONTEND_URL` | `https://postiz.example.com` |
| `NEXT_PUBLIC_BACKEND_URL` | `https://postiz.example.com/api` |
| `JWT_SECRET` | 48+ bytes of random base64 |
| `POSTGRES_USER` | `postiz` |
| `POSTGRES_PASSWORD` | strong random string |
| `POSTGRES_DB` | `postiz` |
| `TEMPORAL_POSTGRES_PASSWORD` | strong random string |

Generate secrets locally:

```bash
# JWT secret
openssl rand -base64 48
# Postgres passwords (alphanumeric)
openssl rand -base64 24 | tr -d '+/='
```

Click **Deploy the stack**. First boot takes a few minutes — the Postiz image
runs DB migrations on start, and the Temporal stack pulls Elasticsearch.

## Step 4 — Reverse proxy

Postiz needs the public origin (`MAIN_URL`) to match what the browser sees,
and OAuth callbacks to terminate on HTTPS. Example minimal nginx vhost:

```nginx
server {
    listen 443 ssl http2;
    server_name postiz.example.com;
    # ssl_certificate / ssl_certificate_key from your ACME tool

    client_max_body_size 100M;

    location / {
        proxy_pass http://127.0.0.1:4007;
        proxy_http_version 1.1;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_read_timeout 300s;
    }
}
```

Caddy equivalent:

```caddyfile
postiz.example.com {
    reverse_proxy 127.0.0.1:4007
}
```

## Step 5 — Create your first account

Visit `MAIN_URL` and register. The first registered account is the admin.
Once you're in, set `DISABLE_REGISTRATION=true` in Portainer and redeploy
the stack if you don't want public sign-ups.

## Step 6 — Add social platform OAuth apps

Each platform you want to publish to needs an OAuth app at the platform's
developer portal. The redirect URI Postiz expects is:

```
${MAIN_URL}/integrations/social/<platform>
```

Fill the matching `*_CLIENT_ID` / `*_CLIENT_SECRET` env vars in Portainer
and redeploy. Postiz's Settings → Integrations page surfaces what's wired.

## Updates

- **Image tag:** `POSTIZ_VERSION=latest` pulls the newest Postiz image on each
  redeploy. Pin to a tag (e.g. `v2.13.0`) if you want stability.
- **Git push:** any commit to the tracked branch triggers Portainer to pull
  and re-apply the stack (with the auto-update setting from Step 2).
- **Migration notes:** the Postiz docs flag a manual migration step when
  upgrading across `v2.11.2 → v2.12.0`. Read release notes before bumping.

## Backups

The stack persists data in named Docker volumes. Snapshot these (or the
underlying VPS disk) on a schedule:

- `postgres-volume` — Postiz database (the most important one)
- `postiz-uploads` — user-uploaded media (skip if you use Cloudflare R2)
- `postiz-config` — runtime config
- `temporal-postgres-data` — Temporal job state (rebuildable, but useful)

Quick dump:

```bash
docker exec -t postiz-postgres pg_dump -U "$POSTGRES_USER" "$POSTGRES_DB" \
  | gzip > postiz-$(date +%F).sql.gz
```

## Troubleshooting

- **`MAIN_URL mismatch` / OAuth redirect errors:** the URL in the browser
  must exactly match `MAIN_URL` (scheme, host, port). Trailing slashes
  matter on some platforms.
- **Container restarts on boot:** check `docker logs postiz` — usually it's
  a missing env var or a Temporal not-yet-ready race.
- **Frontend loads but API 502s:** reverse proxy isn't forwarding to
  `127.0.0.1:4007`, or `POSTIZ_BIND` got changed.
- **Changing env vars doesn't take effect:** Postiz docs require
  `docker compose down && docker compose up`. Portainer's "Update stack"
  recreates containers, which is equivalent.
