# Self-Hosted DNS-over-HTTPS (DoH)

Run your own **private DNS-over-HTTPS** endpoint behind Traefik using Docker Compose. The server forwards DNS queries over **DNS-over-TLS (DoT)** to multiple upstream resolvers (NextDNS, AdGuard, Cloudflare) with automatic failover, while Traefik handles TLS termination, automatic certificates, basic authentication, rate-limiting and compression.

---

## Contents

1. [Features](#features)
2. [Architecture](#architecture)
3. [Requirements](#requirements)
4. [Quick Start](#quick-start)
5. [Configuration](#configuration)
6. [Environment Variables](#environment-variables)
7. [Server Configuration (`doh-server.conf`)](#server-configuration)
8. [Traefik Middleware](#traefik-middleware)
9. [Verifying the Endpoint](#verifying-the-endpoint)
10. [FAQ](#faq)

---

## Features

- **Private** – keep your DNS queries on your own infrastructure.
- **End-to-end encryption** – TLS from client to server (HTTPS) and from server to upstreams (DoT on port 853).
- **Triple upstream failover** – NextDNS (primary), AdGuard, and Cloudflare. If one upstream is slow or down, the server retries another automatically.
- **Basic authentication** – Traefik middleware protects the endpoint with HTTP Basic Auth so only you can use it.
- **Automatic TLS** via Traefik and Let's Encrypt.
- **Rate-limiting** & **compression** middleware pre-configured.
- **Custom config file** (`doh-server.conf`) mounted into the container for full control over upstream and performance settings.

## Architecture

```text
┌──────────┐  HTTPS   ┌─────────┐  DoT (853)  ┌───────────┐
│  Client   │ ──────▶ │ Traefik │ ──────────▶ │ doh-server │
│(Browser/  │         │  (TLS   │              │ (forwards  │──▶ NextDNS / AdGuard / Cloudflare
│ App/      │         │  Auth   │              │  via DoT)  │
│ Little    │         │  Rate   │              └───────────┘
│ Snitch)   │         │  Limit) │
└──────────┘         └─────────┘
```

## Requirements

- A server with **Docker** & **Docker Compose**
- **Traefik v2+** already running and listening on ports 80/443
- A public hostname (e.g. `resolver.example.com`) with a DNS A record pointing to the server's IP
- `htpasswd` (from `apache2-utils`) to generate a password hash for basic auth

> **Note:** Traefik must share a user-defined Docker network with this stack (default: `dokploy-network`). Adjust in `DockerCompose.yaml` if you use a different network.

## Quick Start

```bash
# 1. Clone repo & enter directory
git clone <repo-url> && cd Personal-DoH

# 2. Copy environment template and edit values
cp env-example .env

# 3. Generate a password hash for basic auth
#    (requires apache2-utils or similar)
htpasswd -Bbn myuser mypassword
#    Copy the output hash into DOH_HASHED_PASS in .env

# 4. Edit .env with your domain and credentials
vim .env

# 5. Launch the stack
docker compose up -d
```

After the containers start you should have a working DoH endpoint at:

```text
https://resolver.<DOMAIN>/dns-query
```

## Configuration

The project consists of three key files:

| File | Purpose |
|------|---------|
| `docker-compose.yml` | Service definition, volumes, networking |
| `doh-server.conf` | Server config: upstreams (DoT), timeouts, retries |
| `.env` (from `env-example`) | Domain, path prefix, auth credentials |

## Environment Variables

| Variable | Example | Description |
|----------|---------|-------------|
| `DOMAIN` | `example.com` | Apex domain only (NOT the full subdomain). The compose file builds the FQDN as `resolver.<DOMAIN>`. |
| `DOH_HTTP_PREFIX` | `/dns-query` | URL path the DoH server listens on. |
| `DOH_SERVER_LISTEN` | `8053` | Internal container port (conventional choice). |
| `DOH_USER` | `myusername` | Username for HTTP Basic Auth. |
| `DOH_HASHED_PASS` | `$2y$05$vI8...` | Raw bcrypt hash from `htpasswd -Bbn`. No escaping needed — the container reads the `.env` file directly, bypassing Compose interpolation. |

All variables are loaded from the `.env` file. Credentials (`DOH_USER`, `DOH_HASHED_PASS`) are read directly from the mounted file at container startup, so `$` signs in the bcrypt hash are never mangled.

## Server Configuration

The `doh-server.conf` file is a TOML config mounted into the container. It controls:

- **`listen`** – address and port the server binds to (default `":8053"`)
- **`path`** – HTTP path for DNS queries (must match `DOH_HTTP_PREFIX`)
- **`upstream`** – list of upstream DNS resolvers in `<proto>:<ip>:<port>` format
- **`timeout`** – seconds before an upstream query times out (default `5`)
- **`tries`** – number of retry attempts across upstreams on failure (default `3`)
- **`verbose`** – enable detailed logging (`true`/`false`)
- **`cert`** / **`key`** – TLS cert/key paths (left empty since Traefik handles TLS)

### Upstream format examples

| Format | Protocol |
|--------|----------|
| `udp:1.1.1.1:53` | Plain DNS over UDP |
| `tcp:1.1.1.1:53` | Plain DNS over TCP |
| `tcp-tls:1.1.1.1:853` | DNS-over-TLS (recommended) |

The current configuration uses **DNS-over-TLS** (`tcp-tls`) for all three upstreams so the entire chain is encrypted end-to-end. The server picks one upstream per request and uses the `tries` setting to fail over to another if one is unresponsive.

> **Note:** Options like `upstream_selector`, `weight`, and `[cache]` are **not** supported by the `doh-server` binary (they belong to the `doh-client` component). Do not add them to `doh-server.conf`.

## Traefik Integration

This project uses the **Traefik file provider** — not Docker labels — for routing. At container startup the `app-config` script:

1. Reads `DOH_USER` and `DOH_HASHED_PASS` directly from the mounted `.env` file (immune to Compose `$` interpolation).
2. Reads `DOMAIN` and `DOH_HTTP_PREFIX` from the container environment.
3. Writes a complete Traefik dynamic config to `/etc/dokploy/traefik/dynamic/doh-auth.yml` containing:
   - **HTTPS router** — `Host(`resolver.<DOMAIN>`) && PathPrefix(`<PREFIX>`)`, TLS via Let's Encrypt, `doh-auth` basicAuth middleware.
   - **HTTP → HTTPS redirect router** — same rule, `redirect-to-https` middleware.
   - **Service** — load-balances to `http://doh-server:8053`.
   - **basicAuth middleware** — `user:bcrypt_hash` from the env vars.

Traefik watches the dynamic config directory and picks up changes automatically.

## Verifying the Endpoint

```bash
# Test WITHOUT credentials (should return 401 Unauthorized)
curl -s -o /dev/null -w '%{http_code}' \
  'https://resolver.example.com/dns-query?dns=AAEBAAABAAAAAAAABmdvb2dsZQNjb20AAAEAAQ'

# Test WITH credentials (should return 200 OK with DNS response)
curl -s -o /dev/null -w '%{http_code}' \
  -H 'accept: application/dns-message' \
  -u 'myuser:mypassword' \
  'https://resolver.example.com/dns-query?dns=AAEBAAABAAAAAAAABmdvb2dsZQNjb20AAAEAAQ'
```

### Client configuration (Little Snitch, browsers, etc.)

Use the URL format with embedded credentials:

```text
https://myuser:mypassword@resolver.example.com/dns-query
```

## FAQ

### Can I change the upstream DNS resolvers?

Edit the `upstream` array in `doh-server.conf`. You can use any combination of UDP, TCP, or DoT upstreams.

### Where are logs?

```bash
docker compose logs -f doh-server
```

### How do I update?

```bash
docker compose pull && docker compose up -d
```

### How do I regenerate the password hash?

```bash
htpasswd -Bbn myuser mynewpassword
```

Copy the hash portion into `DOH_HASHED_PASS` in your `.env` file, then restart:

```bash
docker compose up -d
```

### Why not use `[cache]` or `upstream_selector` in the config?

These options belong to the `doh-client` component of `m13253/dns-over-https`, not the server. The server binary will reject them with `unknown option` errors.