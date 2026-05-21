# Deploying condo on a single Linux host

This fork deploys condo on one Linux box (EC2, Hetzner, your own VPS — anything
running Ubuntu/Debian with Docker available) using `docker compose` and a Caddy
reverse proxy that handles SSL automatically via Let's Encrypt.

The whole stack lives in `docker-compose.yml`. One script — `bin/deploy.sh` —
bootstraps the host on first run and redeploys on subsequent runs.

## Prerequisites

- A Linux host (tested on Ubuntu 22.04/24.04) with at least:
  - 4 vCPUs, 8 GB RAM (`t3.large` works; smaller boxes OOM during `yarn build`)
  - 30 GB free disk (image + cache + Postgres volume)
  - Inbound TCP 80 + 443 open from the internet
  - Inbound TCP 22 open from your IP for SSH
- A DNS A record pointing your domain at the host's public IP. Set this **before**
  the first deploy — Let's Encrypt's HTTP-01 challenge verifies it.
- An email address for Let's Encrypt registration (used only for renewal warnings).

## First deploy

SSH to the host as a user with `sudo`, then:

```bash
sudo ACME_EMAIL=ops@yourdomain.com bash -c \
  "$(curl -fsSL https://raw.githubusercontent.com/Senseering/condo/main/bin/deploy.sh)"
```

The script:

1. Installs git + Docker if missing.
2. Clones the repo into `/opt/condo`.
3. Generates `/opt/condo/.env` with random `POSTGRES_PASSWORD`, `COOKIE_SECRET`,
   and `ADMIN_PASSWORD`. **Prints the admin password once — copy it then.**
4. Runs `docker compose build condo` (cold cache: ~30 minutes).
5. Runs `docker compose up -d`. Caddy obtains the SSL cert within ~30 seconds
   after the condo service is up.

Cold build takes a while because the monorepo has 26 workspaces and ~3000 npm
dependencies. Subsequent builds reuse the BuildKit cache and complete in under
a minute when nothing has changed.

### Override defaults

The script reads these env vars:

| Variable     | Default                                | Purpose                                  |
| ------------ | -------------------------------------- | ---------------------------------------- |
| `DOMAIN`     | `condo.datamatters.io`                 | Public hostname; written into Caddy + Keystone SERVER_URL |
| `ACME_EMAIL` | _(required on first run)_              | Let's Encrypt registration email          |
| `APP_DIR`    | `/opt/condo`                           | Where the repo is cloned                 |
| `REPO_URL`   | `https://github.com/Senseering/condo` | Git remote                               |
| `BRANCH`     | `main`                                 | Branch to deploy                         |

Example (different domain):

```bash
sudo DOMAIN=tenants.example.com ACME_EMAIL=ops@example.com \
  bash -c "$(curl -fsSL https://raw.githubusercontent.com/Senseering/condo/main/bin/deploy.sh)"
```

## Redeploying

After the host is bootstrapped, redeploys are one command:

```bash
cd /opt/condo && sudo ./bin/deploy.sh
```

It `git pull`s, rebuilds the image (fast if nothing changed), and restarts only
the containers whose image hash changed. Postgres and Redis volumes persist
across redeploys — your data is safe.

## After the first deploy

The init container seeds an admin user with the password printed by the script.
Log in at `https://<your-domain>/admin/signin`, then **change the admin password
via the admin UI**. The script does not store the password anywhere; `.env` only
contains the *seed* value used during the first migration run.

## Operations

```bash
# Where to run these: in /opt/condo on the host.

docker compose ps                  # what's running
docker compose logs -f condo       # web server logs
docker compose logs -f worker      # background-job logs
docker compose logs -f caddy       # SSL issuance and HTTP routing
docker compose logs init           # see migration output from the last deploy

docker compose exec postgres psql -U postgres local-condo   # SQL shell
docker compose exec condo sh       # shell inside the web container

docker compose restart condo       # restart just the web service
docker compose down                # stop everything (volumes persist)
docker compose down -v             # nuke everything including volumes
```

## Enabling Twilio SMS

By default, outbound SMS (resident sign-up OTPs, password-reset codes) is
routed to the worker's stdout — search `docker compose logs worker` for
the code if you need it during testing. To send real SMS via Twilio:

1. Buy an SMS-capable phone number in the Twilio Console (or claim a trial
   one) and grab the **Account SID**, **Auth Token**, and the **From number**
   in E.164 format (e.g. `+12025551234`).
2. On the host, append to `/opt/condo/.env`:

   ```
   SMS_PROVIDER=TWILIO
   TWILIO_ACCOUNT_SID=AC...
   TWILIO_AUTH_TOKEN=...
   TWILIO_FROM_NUMBER=+12025551234
   NOTIFICATION_SEND_ALL_MESSAGES_TO_CONSOLE=false
   ```

   All four `TWILIO_*` vars are required, and the console-sink flag must be
   flipped to `false` — otherwise the worker will keep short-circuiting
   messages to stdout regardless of the provider config.

3. `cd /opt/condo && sudo ./bin/deploy.sh` (or just
   `sudo docker compose up -d condo worker` if you don't want to redeploy
   the image).

4. Verify outbound delivery in Twilio Console → Monitor → Logs → Messaging
   after triggering a resident sign-up. The adapter sends `Messages.json`
   requests against `api.twilio.com/2010-04-01`.

Switching to another provider (`SMSC`, `INSTASENT`) follows the same
shape — see `apps/condo/domains/notification/adapters/smsAdapter.js` for
the JSON each one expects.

## Backups

Nothing is automated. At a minimum, schedule `pg_dump`:

```bash
# As a cron job on the host:
docker compose exec -T postgres pg_dump -U postgres local-condo \
  | gzip > /backup/condo-$(date +%F).sql.gz
```

Ship the dumps off-host (S3, another VPS). Test restores periodically.

## What's missing

This setup is deliberately small. It does **not** include:

- File uploads to S3 — `FILE_FIELD_ADAPTER=local` keeps uploads on the
  container's ephemeral disk; redeploys do not lose them (the volume persists)
  but losing the host loses them. Add an S3 adapter via env vars when needed.
- Push notification adapters (Firebase, Apple, HCM, RedStore) — they log
  "config not provided" warnings at startup, which is harmless if you aren't
  sending push notifications.
- NATS messaging — the app logs `CONNECTION_REFUSED` for the NATS auth callout
  service if not present, also harmless.
- Address-service, dev-portal, miniapp, and the other monorepo apps. Only
  `@app/condo` and `@app/address-service` (the latter via condo's fake-client
  fallback) are deployed.
- Horizontal scaling. One container per service. For multi-host or HA,
  graduate to a real orchestrator.

## Troubleshooting

**Caddy says `tls: error obtaining certificate`.** DNS isn't pointing at the
host yet, or 80/443 isn't reachable. Verify with `dig condo.datamatters.io`
and `curl -v http://<your-ip>` from outside the host.

**Build OOM-killed.** You're on a host smaller than 8 GB RAM. The Dockerfile
serializes turbo to one workspace at a time and caps node heap at 2 GB
specifically to fit in modest hosts, but the lower bound is roughly 4 GB.
Upgrade the host or build elsewhere and push to a registry.

**`init` keeps exiting non-zero.** Check `docker compose logs init`. Most
likely a database connectivity issue; verify `POSTGRES_PASSWORD` in `.env`
matches what was used the first time (rotating it without rotating the
volume bricks Postgres auth).

**Want to wipe and start over.** `docker compose down -v && rm .env && ./bin/deploy.sh`.
Destroys all data, regenerates secrets, fresh admin password.
