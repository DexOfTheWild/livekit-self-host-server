# livekit-deploy

Minimal single-box deployment repo for a self-hosted [LiveKit Server](https://docs.livekit.io/transport/self-hosting.md) on Ubuntu or DigitalOcean.

This repo does one thing: run the actual LiveKit realtime server that browsers connect to at a URL such as `wss://livekit.example.com`.

It is intentionally separate from `../livekit-api`, which remains the small Node/Express token service.

## Shippable By Default

The repo is meant to be safe to publish:

- real domains, IPs, and secrets live in local `.env`
- `.env` is gitignored
- checked-in files stay generic
- `livekit.yaml.example` is a reference file only, not the runtime source of truth

The running `livekit` container now gets its config from `.env` through `docker-compose.yml`, so you no longer have to put your real key, secret, TURN hostname, or TURN shared secret into a tracked YAML file.

## What Lives Where

- `../livekit-api`
  - serves `GET /token` and `GET /health`
  - signs join tokens with your LiveKit API key and secret
  - returns `wsUrl` to the browser
- `../livekit-deploy`
  - runs the real self-hosted LiveKit Server
  - exposes the browser-facing websocket endpoint, for example `wss://livekit.example.com`
  - handles the realtime media plane

Intended public architecture:

```text
https://voice.example.com/token    -> ../livekit-api
https://voice.example.com/health   -> ../livekit-api
wss://livekit.example.com          -> LiveKit Server from this repo
```

## First-Pass Deployment Shape

This repo uses:

- Docker Compose
- `livekit/livekit-server`
- Redis on the same box
- external `coturn` for STUN/TURN/TURNS
- an edge proxy for TLS termination on `livekit.example.com`

That keeps the first deployment simple while still matching LiveKit's documented production guidance:

- terminate TLS in a reverse proxy or load balancer for the signal endpoint
- give LiveKit access to UDP for WebRTC
- provide external TURN/TLS for restrictive networks
- set `rtc.use_external_ip: true` for typical cloud VM deployments
- use a config file for production-style settings

On this box, Apache already owns `80/443`, so the included `caddy` service is optional and behind the `caddy-edge` Compose profile. If Apache is your public edge, proxy `livekit.example.com` to `127.0.0.1:7880` instead of starting Caddy.

Because Apache owns `443/tcp` on this single-IP host, phase 1 exposes TURNS on `turn.example.com:5349` instead of `:443`. That still gives you encrypted TURN/TLS, but if you later need the broadest firewall compatibility, plan for either a dedicated TURN IP or a different edge layout so `turn.example.com:443` can belong to coturn.

Relevant docs:

- [Self-hosting overview](https://docs.livekit.io/transport/self-hosting.md)
- [Deploying LiveKit](https://docs.livekit.io/transport/self-hosting/deployment.md)
- [Ports and firewall](https://docs.livekit.io/transport/self-hosting/ports-firewall.md)
- [Running locally](https://docs.livekit.io/transport/self-hosting/local.md)

## Files

- `docker-compose.yml` starts Redis, LiveKit, and coturn by default, with optional Caddy under the `caddy-edge` profile
- `monitoring/prometheus.yml` scrapes the local API, LiveKit, coturn, Redis, and host exporters
- `livekit.yaml.example` shows the equivalent LiveKit config shape for reference
- `Caddyfile` is only needed if you want this repo to own the public HTTPS/WSS edge instead of Apache or another reverse proxy
- `.env.example` is the operator checklist for hostnames, image tags, and the credentials you must keep in sync

## Required Values

You must choose five values for the deployment:

- the public websocket hostname, for example `livekit.example.com`
- the public TURN hostname, for example `turn.example.com`
- a self-hosted LiveKit API key
- a self-hosted LiveKit API secret
- a TURN shared secret used by coturn and by LiveKit's advertised `rtc.turn_servers`

Generate them with something high-entropy, for example:

```bash
openssl rand -hex 16
openssl rand -hex 32
```

These values are for your self-hosted LiveKit server configuration. They are not special credentials issued by LiveKit Cloud, and they do not need to come from a LiveKit Cloud project. You choose and generate them yourself, then use the same values in both this repo and `../livekit-api`.

Example:

```bash
# Generate a key identifier for your self-hosted LiveKit server.
openssl rand -hex 16

# Generate the matching secret.
openssl rand -hex 32
```

Take the first output and put it in:

```dotenv
LIVEKIT_API_KEY=first-generated-value
```

Take the second output and put it in:

```dotenv
LIVEKIT_API_SECRET=second-generated-value
```

Use the generated values in both places:

1. In this repo, set them in local `.env`.
2. In `../livekit-api/.env`, set:

```dotenv
VOIP_LIVEKIT_API_KEY=the-same-livekit-api-key
VOIP_LIVEKIT_API_SECRET=the-same-livekit-api-secret
VOIP_LIVEKIT_WS_URL=wss://livekit.example.com
```

If you ever use LiveKit Cloud instead of this self-hosted deployment, that is a different setup with a different source of credentials.

Important:

- `VOIP_ROOM_NAME` stays in `../livekit-api`; it is not part of the LiveKit server deployment config.
- `../livekit-api` is not the media server. It only mints tokens and tells the browser where the real LiveKit server lives.
- `docker-compose.yml` now renders the LiveKit `keys`, `webhook.api_key`, and advertised TURN secrets from `.env`, so you only set them once locally.
- `livekit.yaml.example` is just a reference copy of the rendered config shape.

## DNS, TLS, and Reverse Proxy Expectations

Before bringing the stack up publicly:

1. Create a DNS record for `livekit.example.com` pointing at your Ubuntu or DigitalOcean box.
2. Create a DNS record for `turn.example.com` pointing at the same box.
3. Make sure ports `80/tcp` and `443/tcp` are reachable from the internet so the HTTPS certificate can be issued and renewed.
4. Keep `wss://livekit.example.com` as the browser-facing URL that you place in `../livekit-api/.env`.
5. Use a TLS certificate that covers both `livekit.example.com` and `turn.example.com`.

### How To Set Up The TURN Subdomain

If you already know how to create DNS records, `turn.example.com` is just another subdomain that points at the same server.

Example:

- root domain: `example.com`
- LiveKit signal hostname: `livekit.example.com`
- TURN hostname: `turn.example.com`
- server public IP: `203.0.113.10`

In your DNS provider, create an `A` record like:

```text
Type: A
Name: turn
Value: 203.0.113.10
```

If you use IPv6, also create the matching `AAAA` record for `turn.example.com`.

Then set the same hostname in `.env`:

```dotenv
TURN_DOMAIN=turn.example.com
TURN_PUBLIC_IP=203.0.113.10
```

The important part is that:

- `turn.example.com` resolves publicly to your server
- your TLS certificate covers `turn.example.com`
- the hostname in `TURN_DOMAIN` matches the DNS record and certificate name exactly

The included `Caddyfile` only handles HTTPS/WSS for the LiveKit signal endpoint. TURN and WebRTC media still depend on the additional TCP/UDP ports described below.

## Firewall and Networking Notes

For this first pass, the rendered LiveKit config uses:

- `rtc.tcp_port: 7881`
- `rtc.udp_port: 7882`
- `rtc.use_external_ip: true`
- external TURN servers on `turn.example.com`
- TURN relay UDP range `49160-49259`

That means you should allow inbound traffic for:

- `80/tcp` for ACME and HTTPS redirects
- `443/tcp` for HTTPS and `wss://livekit.example.com`
- `3478/tcp` for TURN over TCP
- `3478/udp` for TURN/STUN over UDP
- `5349/tcp` for TURN over TLS
- `7881/tcp` for ICE/TCP fallback
- `7882/udp` for WebRTC UDP mux
- `49160-49259/udp` for coturn relay allocations

Why a single UDP port instead of the larger `50000-60000` range shown in LiveKit's broader production example?

- it keeps first bring-up and firewalling simpler on one box
- it is a reasonable trade-off for an initial deployment
- you can switch to a wider UDP port range later if you need more headroom

## TURN Notes

Phase 1 now includes external coturn:

- `turn.example.com:3478` for STUN/TURN over UDP/TCP
- `turn.example.com:5349` for encrypted TURN/TLS
- relay ports `49160-49259/udp`

LiveKit advertises these TURN servers to clients through `rtc.turn_servers` in the rendered config.

Important limitation on this specific host:

- Apache already owns `443/tcp`
- coturn therefore cannot also own `turn.example.com:443` on the same public IP
- the current encrypted TURN fallback is `turns:turn.example.com:5349`

That is enough to validate TURN/TLS and remote ICE reachability, but it is not the absolute best-case corporate-firewall posture. If you later need TURN/TLS on `443`, give coturn its own edge or its own public IP.

## Observability

Phase 3 adds a minimal local monitoring stack:

- LiveKit exposes Prometheus metrics on `127.0.0.1:6789`
- coturn exposes Prometheus metrics on `127.0.0.1:9641`
- node-exporter exposes host CPU, RAM, disk, and network metrics on `127.0.0.1:9100`
- redis-exporter exposes Redis health on `127.0.0.1:9121`
- Prometheus scrapes those plus the token API at `127.0.0.1:3000/metrics` and serves a local UI on `127.0.0.1:9090`
- Docker container logs for `livekit`, `coturn`, `redis`, and the monitoring services are rotated with the `json-file` driver to avoid unbounded growth

LiveKit also posts signed room lifecycle and participant events to the token API at `http://127.0.0.1:3000/livekit/webhook`, which gives you a lightweight join/leave event trail in the API logs without exposing a new public endpoint.

## First Bring-Up

1. Copy the environment example:

   ```bash
   cp .env.example .env
   ```

2. Edit `.env` and fill in:

   - `LIVEKIT_DOMAIN`
   - `TURN_DOMAIN`
   - `TURN_PUBLIC_IP`
   - `TURN_SHARED_SECRET`
   - `TURN_CERT_FULLCHAIN_FILE`
   - `TURN_CERT_PRIVKEY_FILE`
   - `LIVEKIT_API_KEY`
   - `LIVEKIT_API_SECRET`

3. Optionally inspect `livekit.yaml.example` if you want to see the config shape that Compose will render for LiveKit.

4. If you changed the domains from `livekit.example.com` or `turn.example.com`, update `.env` so the rendered config, coturn, and Caddy all stay aligned.

5. Start the stack:

   ```bash
   docker compose up -d
   ```

   If you want this repo to run its own Caddy edge instead of Apache, start it with:

   ```bash
   docker compose --profile caddy-edge up -d
   ```

6. Watch startup logs:

   ```bash
   docker compose logs -f livekit coturn caddy redis prometheus node-exporter redis-exporter
   ```

## Verifying Reachability

After startup:

1. Confirm the containers are up:

   ```bash
   docker compose ps
   ```

2. Confirm TLS and HTTP reach the LiveKit signal endpoint:

   ```bash
   curl -I https://livekit.example.com
   ```

   You do not need a pretty HTML page here. Any valid HTTP response over trusted TLS proves the hostname and proxy path are alive.

3. Confirm the TURN TLS endpoint answers with the expected certificate:

   ```bash
   openssl s_client -connect turn.example.com:5349 -servername turn.example.com -brief </dev/null
   ```

4. Point `../livekit-api/.env` at the same websocket URL:

   ```dotenv
   VOIP_LIVEKIT_WS_URL=wss://livekit.example.com
   VOIP_LIVEKIT_API_KEY=the-same-livekit-api-key
   VOIP_LIVEKIT_API_SECRET=the-same-livekit-api-secret
   ```

5. Restart `../livekit-api`, then check:

   ```bash
   curl https://voice.example.com/health
   ```

   The returned `wsUrl` should match `wss://livekit.example.com`.

6. Optionally verify coturn listeners on the host:

   ```bash
   ss -ltnup | awk '$5 ~ /:3478$|:5349$|:49160$/ { print }'
   ```

7. Verify local observability endpoints:

   ```bash
   curl http://127.0.0.1:3000/metrics
   curl http://127.0.0.1:6789/metrics
   curl http://127.0.0.1:9641/metrics
   curl http://127.0.0.1:9100/metrics
   curl http://127.0.0.1:9121/metrics
   curl http://127.0.0.1:9090/-/ready
   ```

At that point the browser can request a token from `../livekit-api` and try to connect to the self-hosted LiveKit server with TURN fallback available.

## Local / Dev Notes

For local-only experiments, the fastest official option is still:

```bash
livekit-server --dev
```

That starts a local instance with:

- API key: `devkey`
- API secret: `secret`
- default signal URL: `ws://127.0.0.1:7880`

If you want to test `../livekit-api` locally against a local LiveKit instance, use:

```dotenv
VOIP_LIVEKIT_WS_URL=ws://127.0.0.1:7880
VOIP_LIVEKIT_API_KEY=devkey
VOIP_LIVEKIT_API_SECRET=secret
```

For public deployment, keep using a TLS-backed hostname instead of a raw `ws://` endpoint.

## What To Set In `../livekit-api/.env`

Once this repo is up, the key values to copy into `../livekit-api/.env` are:

```dotenv
VOIP_LIVEKIT_API_KEY=<same value used in livekit-deploy/.env>
VOIP_LIVEKIT_API_SECRET=<same value used in livekit-deploy/.env>
VOIP_LIVEKIT_WS_URL=wss://livekit.example.com
```

Those values must stay in sync or token minting / joins will fail.

If any real secrets or domains were already committed before this cleanup, rotate them before publishing the repo.

## Known Limitations and Follow-Ups

- single node only; no high availability
- TURN/TLS is implemented with external coturn, but currently on `5349` rather than `443` because Apache owns `443`
- no Prometheus / log shipping / dashboards yet
- image tags are intentionally lightweight for first bring-up; pin exact versions after your first successful deployment
- if you outgrow UDP mux on `7882`, switch to a wider UDP port range and update firewall rules
- if you deploy behind Cloudflare or another proxy, avoid proxying the raw WebRTC UDP traffic through it; keep the LiveKit media ports directly reachable
- if Apache already fronts the hostname on this server, do not start the optional `caddy` profile at the same time
