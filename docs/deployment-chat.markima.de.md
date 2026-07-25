# `chat.markima.de` production deployment

The production topology is:

```text
Browser -> Cloudflare -> ZeitPlan Nginx :443 -> public-proxy -> SillyTavern :8000
```

Cloudflare's proxied DNS is an edge reverse proxy, but the server still needs a
gateway to terminate origin TLS and route multiple hostnames on ports 80/443.
The existing ZeitPlan Nginx remains that gateway. SillyTavern itself publishes
no host port.

## 1. Prepare this repository on the server

```bash
git clone --branch release git@github.com:Markima323/SillyTavern.git /srv/sillytavern
cd /srv/sillytavern
cp deploy/.env.production.example .env
chmod 600 .env
```

Replace the Basic Auth password in `.env`. Keep this file only on the server.

## 2. Configure GitHub

Create a `production` Environment in this repository and add these secrets:

- `HETZNER_HOST`
- `HETZNER_PORT` (optional; defaults to `22`)
- `HETZNER_USER`
- `HETZNER_SSH_KEY`
- `HETZNER_SSH_PASSPHRASE` (only for an encrypted key)
- `HETZNER_DEPLOY_PATH` (for example `/srv/sillytavern`)

A push to `release`, or a manual workflow run, validates and builds the image,
backs up the persistent volumes, updates the checkout, and starts the service.
Backups are retained for 14 days under `deploy/backups` on the server.

## 3. Deploy the application, then the gateway

Run the SillyTavern workflow once. Afterwards push/deploy the accompanying
ZeitPlan gateway changes so its Nginx container knows the new virtual host.

On the ZeitPlan server checkout, add this to `.env`:

```dotenv
CHAT_SERVER_NAME=chat.markima.de
```

Expand the existing Let's Encrypt certificate lineage to cover all three
hostnames (replace the values if your `.env` uses different names):

```bash
cd /srv/zeitplan
set -a
. ./.env
set +a
docker compose --env-file .env -f docker-compose.prod.yml run --rm \
  --entrypoint certbot certbot certonly --webroot -w /var/www/certbot \
  --cert-name "$SERVER_NAME" --expand \
  -d "$SERVER_NAME" -d "$RESUME_SERVER_NAME" -d "$CHAT_SERVER_NAME"
docker compose --env-file .env -f docker-compose.prod.yml up -d --force-recreate nginx
```

The certificate must include every name because Cloudflare `Full (strict)`
validates the origin certificate against `chat.markima.de`.

## 4. Configure Cloudflare

In the `markima.de` zone:

1. Add an `A` record named `chat` pointing to the Hetzner server IPv4 address.
2. Enable **Proxied** (orange cloud).
3. Set SSL/TLS encryption mode to **Full (strict)**.
4. Add a Cache Rule that bypasses cache for `chat.markima.de/*`; chat pages and
   API responses are dynamic and should not be cached.

Then open `https://chat.markima.de` and sign in with the Basic Auth credentials
stored in the SillyTavern server `.env`.

## Persistent data

Configuration, users, chats, plugins, third-party extensions and application
backups live in explicitly named Docker volumes. Rebuilding the image does not
remove them. Never run `docker compose down -v` in production unless deleting
all application data is intentional.
