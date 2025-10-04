# Caddy + Cloudflare DNS-01 for Nextcloud (Docker) — LAN‑only, Real HTTPS

This guide shows how to front an **existing Nextcloud-on-Docker** stack with **Caddy** and obtain **real Let’s Encrypt certificates** using the **Cloudflare DNS-01** challenge. The goal is clean HTTPS on your **LAN only** (no inbound ports opened to the Internet), e.g.:

```
https://nextcloud.intra.<your-domain>
```

The steps assume you already have a working Nextcloud stack (e.g., `nextcloud:apache` + MariaDB + Redis) running on a Linux VM with Docker Compose.

---

## What you get

- A Caddy reverse proxy container that **terminates TLS** for your apps.
- **Automatic HTTPS** certificates via Let’s Encrypt using **Cloudflare DNS-01** (wildcard‑ready).
- A setup that works fine for LAN‑only access; **no router port‑forwards** required.
- A repeatable project layout you can drop into an existing compose stack.

---

## Prerequisites

- Docker + Docker Compose v2 on the VM that runs Nextcloud.
- A domain in **Cloudflare** (zone controlled by you).
- A desired internal hostname like `nextcloud.intra.example.com` that should resolve to your VM’s **LAN IP** (e.g., `192.168.1.xxx`).
- Your existing Nextcloud stack (service `app` for the web container is assumed below; adapt if yours differs).

> You can achieve LAN resolution either by **public DNS “DNS‑only” records** pointing to your RFC1918 LAN IP (simple but has caveats), or by **Split DNS** (Pi‑hole/AdGuard Home Local DNS). DNS choice does not affect certificate issuance because we’re using **DNS‑01**.

---

## 1) DNS records (recommended pattern)

In Cloudflare (DNS‑only, grey cloud), add:

- **A** `intra` → `192.168.1.xxx` (your VM’s LAN IP)
- **CNAME** `*.intra` → `intra`

This makes both `intra.<domain>` **and** any subdomain like `nextcloud.intra.<domain>` resolve to your VM.

**Check:**

```bash
nslookup nextcloud.intra.<your-domain>
# Should resolve to your VM LAN IP (e.g., 192.168.1.xxx)
```

> Alternative: Use **Split DNS** on your LAN so these names resolve to your VM without publishing RFC1918 records publicly. Cert issuance still works via DNS‑01 either way.

---

## 2) Project layout

Place these files **next to your existing `docker-compose.yml`**:

```
/your/project/
├─ docker-compose.yml          # your existing compose (Nextcloud stack)
├─ Caddyfile                   # new
├─ Dockerfile.caddy            # new
└─ .env                        # new (holds your Cloudflare token; do NOT commit)
```

Add a `.gitignore` entry so you don’t commit secrets:

```
.env
```

---

## 3) `.env` — Cloudflare API token

Create `.env` with the following line (no quotes, no spaces):

```
CLOUDFLARE_API_TOKEN=YOUR_REAL_CLOUDFLARE_TOKEN
```

**Required token permissions** (scoped to your zone):
- Zone → **DNS**: Edit
- Zone → **Zone**: Read

> This token lets Caddy publish temporary `_acme-challenge` TXT records to solve DNS‑01. Keep it secret.

---

## 4) `Dockerfile.caddy` — build Caddy with Cloudflare DNS plugin

```Dockerfile
# Dockerfile.caddy
FROM caddy:builder AS builder
RUN xcaddy build \
    --with github.com/caddy-dns/cloudflare

FROM caddy:latest
COPY --from=builder /usr/bin/caddy /usr/bin/caddy
```

This produces a Caddy image that knows how to talk to Cloudflare’s DNS API.

---

## 5) `Caddyfile` — site definition for Nextcloud

Replace `<your-domain>` with your actual domain.

```caddy
{
  # Optional but recommended so ACME account notices reach you
  email your.real.email@yourprovider.tld
}

# Nextcloud over HTTPS via Cloudflare DNS-01
nextcloud.intra.<your-domain> {
  encode zstd gzip

  # Real certs via Cloudflare DNS-01; token is provided via container env
  tls {
    dns cloudflare {env.CLOUDFLARE_API_TOKEN}
  }

  # Well-known DAV redirects (silences Nextcloud admin notices)
  redir /.well-known/carddav /remote.php/dav/ 301
  redir /.well-known/caldav  /remote.php/dav/ 301

  # Reverse proxy to your Nextcloud web container (service name `app`)
  reverse_proxy app:80

  # Basic hardening headers
  header {
    Strict-Transport-Security "max-age=63072000; includeSubDomains; preload"
    X-Content-Type-Options "nosniff"
    Referrer-Policy "no-referrer"
  }
}
```

> If your Nextcloud web service is not named `app`, change `reverse_proxy app:80` accordingly.

---

## 6) Amend `docker-compose.yml` — add Caddy service (Nextcloud unchanged)

Add this service and two volumes; keep your existing `db`, `redis`, and `app` as they are.

```yaml
version: "3.8"

services:
  caddy:
    build:
      context: .
      dockerfile: Dockerfile.caddy
    container_name: caddy
    restart: unless-stopped
    ports:
      - "80:80"
      - "443:443"
    environment:
      # Pulled from .env (no quotes, no spaces in .env)
      CLOUDFLARE_API_TOKEN: "${CLOUDFLARE_API_TOKEN}"
    volumes:
      - ./Caddyfile:/etc/caddy/Caddyfile:ro
      - caddy_data:/data
      - caddy_config:/config

  # --- your existing services below ---
  # db:
  #   ...
  # redis:
  #   ...
  # app:
  #   image: nextcloud:apache
  #   container_name: nextcloud-app
  #   ports:
  #     - "8080:80"     # keep for now; remove later to force HTTPS-only
  #   ...

volumes:
  caddy_data:
  caddy_config:
```

> If you were previously running **Nginx Proxy Manager** or any other process listening on ports **80/443**, stop/remove it first to avoid a port conflict.

---

## 7) Start Caddy and watch issuance

```bash
# Validate your compose
docker compose config

# Build Caddy with the Cloudflare plugin
docker compose build caddy

# Start Caddy
docker compose up -d caddy

# Confirm it’s listening
sudo ss -tulpn | egrep ':(80|443)\s'

# Check Caddy logs for ACME/DNS-01 activity
docker logs --tail=200 -f caddy

# Confirm the plugin is present
docker exec -it caddy caddy list-modules | grep -i cloudflare
```

You should see logs like “obtaining certificate for nextcloud.intra.<your-domain>” and eventually success.

**Quick wire check:**

```bash
curl -vkI https://nextcloud.intra.<your-domain>/
# Expect issuer: Let's Encrypt (or ZeroSSL); NOT "Caddy Local Authority"
```

---

## 8) Tell Nextcloud about the reverse proxy

Nextcloud must trust your proxy and generate correct HTTPS links.

1) **Find your Docker bridge subnet** (the compose default network):

```bash
docker network ls | awk '/_default/ {print $2}'
docker network inspect <that_name> | grep Subnet
# Example: "Subnet": "172.18.0.0/16"
```

2) **Apply Nextcloud settings** (run once; adjust domain and subnet to yours):

```bash
# Add your FQDN to trusted domains (use next free index if needed)
docker exec -u www-data nextcloud-app php occ config:system:set trusted_domains 2 --value=nextcloud.intra.<your-domain>

# Make Nextcloud generate proper HTTPS links
docker exec -u www-data nextcloud-app php occ config:system:set overwritehost --value=nextcloud.intra.<your-domain>
docker exec -u www-data nextcloud-app php occ config:system:set overwriteprotocol --value=https

# Trust the proxy network so client IPs are preserved
docker exec -u www-data nextcloud-app php occ config:system:set trusted_proxies 0 --value=172.18.0.0/16

# (Optional, explicit forwarded headers)
docker exec -u www-data nextcloud-app php occ config:system:set forwarded_for_headers 0 --value=HTTP_X_FORWARDED_FOR
docker exec -u www-data nextcloud-app php occ config:system:set forwarded_for_headers 1 --value=HTTP_FORWARDED

docker restart nextcloud-app
```

Re-check **Settings → Administration → Overview**; the “reverse proxy header configuration” warning should be gone.

---

## 9) Verify end-to-end

- DNS:
  ```bash
  nslookup nextcloud.intra.<your-domain>
  # -> LAN IP of your VM
  ```

- Cert on the wire:
  ```bash
  curl -vkI https://nextcloud.intra.<your-domain>/
  ```

- Upstream reachability from Caddy:
  ```bash
  docker exec -it caddy sh -lc "wget -S -O - http://app:80/status.php"
  # Expect: {"installed":true, ...}
  ```

- Browser: open `https://nextcloud.intra.<your-domain>` → valid Let’s Encrypt padlock.

---

## 10) Enforce HTTPS‑only (optional but recommended)

Once you’re satisfied everything works through Caddy, remove the direct host port from the Nextcloud service:

```yaml
# app:
#   ports:
#     - "8080:80"   # remove this
```

Apply:
```bash
docker compose up -d app
```

Now users can only reach Nextcloud via Caddy over HTTPS.

---

## 11) Add more services (copy‑paste)

Add blocks to the same `Caddyfile` and ensure the containers are in the same compose/network.

**Plex**
```caddy
plex.intra.<your-domain> {
  tls { dns cloudflare {env.CLOUDFLARE_API_TOKEN} }
  reverse_proxy plex:32400
}
```

**Jellyfin**
```caddy
jellyfin.intra.<your-domain> {
  tls { dns cloudflare {env.CLOUDFLARE_API_TOKEN} }
  reverse_proxy jellyfin:8096
}
```

**Home Assistant**
```caddy
homeassistant.intra.<your-domain> {
  tls { dns cloudflare {env.CLOUDFLARE_API_TOKEN} }
  reverse_proxy homeassistant:8123
}
```

Reload Caddy after edits:
```bash
docker compose up -d caddy
```

---

## 12) Backups & maintenance

- **Nextcloud data**: your bind mount (e.g., `/srv/nextcloud-data`).
- **Nextcloud config** volume: contains `config.php` and app configs.
- **Caddy volumes**: `caddy_data` and `caddy_config` (certs/keys/state).

Useful commands:
```bash
# Restart Docker engine (all containers stop/start)
sudo systemctl restart docker

# Restart services
docker compose restart caddy
docker compose restart app

# Logs
docker logs --tail=200 caddy
docker logs --tail=200 nextcloud-app
```

---

## Troubleshooting

- **“Caddy Local Authority” in browser**  
  ACME failed; Caddy fell back to a local cert. Check `docker logs caddy` for DNS‑01 errors.

- **`unrecognized DNS provider "cloudflare"` in logs**  
  Your Caddy image lacks the plugin. Rebuild using `Dockerfile.caddy` (xcaddy line), then:
  ```bash
  docker compose build caddy && docker compose up -d caddy
  ```

- **`403` / permission errors from Cloudflare**  
  Fix your API token permissions (Zone: DNS Edit, Zone: Read) and make sure it’s scoped to the correct zone.

- **`timed out waiting for record to fully propagate`**  
  Temporary DNS lag or token/zone misconfig. Verify the token, wait a minute, and let Caddy retry automatically.

- **TLS alert / handshake errors**  
  Often transient during issuance. Re-run `curl -vkI https://nextcloud.intra.<your-domain>/` after logs show success.

- **Nextcloud admin warning “reverse proxy header configuration is incorrect”**  
  Set `trusted_proxies` to either the Caddy container IP or the Docker bridge CIDR and (optionally) set `forwarded_for_headers` as shown above, then restart the app.

- **502 / Bad Gateway**  
  Check that your upstream is correct: `reverse_proxy app:80` (use the **service name**, not `container_name`). Test from Caddy:
  ```bash
  docker exec -it caddy sh -lc "wget -S -O - http://app:80/ | head"
  ```

- **Ports 80/443 already in use**  
  Stop/remove other proxies or web servers (e.g., NPM, nginx, apache) before starting Caddy.

- **Why container IPs like `172.18.0.5`?**  
  That’s Docker’s internal network. Your LAN stays `192.168.x.y`. Trust the Docker subnet in `trusted_proxies` so Nextcloud logs real client IPs.

---

## Security notes

- Keep the Cloudflare token in **`.env`** (not committed).  
- Prefer **Split DNS** if you don’t want to publish RFC1918 addresses in public DNS. Certificates still work via DNS‑01 either way.  
- Remove the Nextcloud direct port (`8080:80`) after verifying HTTPS, so all access is forced through Caddy.

---

## Appendix: Full example Caddy block

```caddy
{
  email your.real.email@yourprovider.tld
}

nextcloud.intra.<your-domain> {
  encode zstd gzip
  tls {
    dns cloudflare {env.CLOUDFLARE_API_TOKEN}
  }
  redir /.well-known/carddav /remote.php/dav/ 301
  redir /.well-known/caldav  /remote.php/dav/ 301
  reverse_proxy app:80

  header {
    Strict-Transport-Security "max-age=63072000; includeSubDomains; preload"
    X-Content-Type-Options "nosniff"
    Referrer-Policy "no-referrer"
  }
}
```

---

## Appendix: Minimal Caddy service for `docker-compose.yml`

```yaml
services:
  caddy:
    build:
      context: .
      dockerfile: Dockerfile.caddy
    container_name: caddy
    restart: unless-stopped
    ports:
      - "80:80"
      - "443:443"
    environment:
      CLOUDFLARE_API_TOKEN: "${CLOUDFLARE_API_TOKEN}"
    volumes:
      - ./Caddyfile:/etc/caddy/Caddyfile:ro
      - caddy_data:/data
      - caddy_config:/config

volumes:
  caddy_data:
  caddy_config:
```
