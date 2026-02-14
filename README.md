# Self-Hosted DNS-over-HTTPS (DoH)

> **Author:** Ali Rajabpour Sanati — [Rajabpour.com](https://rajabpour.com)

Run your own **private DNS-over-HTTPS** endpoint behind Traefik using Docker Compose. The server forwards DNS queries over **DNS-over-HTTPS (DoH)** to multiple upstream resolvers (Cloudflare, AdGuard, NextDNS) via a `dnscrypt-proxy` sidecar, while Traefik handles TLS termination, automatic certificates, and basic authentication. All external traffic is encrypted end-to-end over port 443.

---

## Contents

1. [Features](#features)
2. [Architecture](#architecture)
3. [Requirements](#requirements)
4. [Quick Start](#quick-start)
5. [Configuration](#configuration)
6. [Environment Variables](#environment-variables)
7. [Server Configuration (`doh-server.conf`)](#server-configuration)
8. [Traefik Integration](#traefik-integration)
9. [Client Setup](#client-setup)
10. [Verifying the Endpoint](#verifying-the-endpoint)
11. [FAQ](#faq)
12. [License](#license)

---

## Features

- **Private** — keep your DNS queries on your own infrastructure.
- **End-to-end encryption** — HTTPS from client to server, DoH (HTTPS/443) from server to upstreams. All external traffic is encrypted.
- **Aggressive caching** — Unbound with `prefetch` (refreshes entries *before* TTL expires) and `serve-expired` (returns stale data instantly, refreshes in background). Repeat queries are near-instant.
- **NextDNS primary, failover to Cloudflare / AdGuard** — `dnscrypt-proxy` with `lb_strategy = 'first'` always uses NextDNS, falling back only if it's down.
- **Basic authentication** — Traefik basicAuth middleware protects the endpoint so only you can use it.
- **Automatic TLS** — Let's Encrypt certificates managed by Traefik.
- **No credential escaping** — the `app-config` script reads the `.env` file directly at startup, so `$` signs in bcrypt hashes are never mangled by Docker Compose.
- **Custom config file** (`doh-server.conf`) mounted into the container for full control over upstream and performance settings.

## Architecture

```text
                         Docker internal network
┌──────────┐ HTTPS ┌─────────┐       ┌────────────┐       ┌─────────┐       ┌────────────────┐ DoH(443)
│  Client   │─────▶│ Traefik │──────▶│ doh-server  │──────▶│ Unbound │──────▶│ dnscrypt-proxy │────────▶ NextDNS
│           │      │ • TLS   │       │             │       │ • cache │       │ • DoH upstream │────────▶ Cloudflare
│ Little    │      │ • Auth  │       │             │       │ • pre-  │       │ • NextDNS 1st  │────────▶ AdGuard
│ Snitch /  │      │         │       └────────────┘       │   fetch │       │ • failover     │
│ curl      │      └─────────┘                             └─────────┘       └────────────────┘
└──────────┘
```

> **Why not DoT (port 853)?** Many VPS providers block outbound port 853. DoH uses port 443 (standard HTTPS), which is virtually never blocked. Security is equivalent — both encrypt DNS queries with TLS.

## Requirements

- A server with **Docker** and **Docker Compose**
- **Traefik v2+** already running and listening on ports 80/443
- A public hostname (e.g. `resolver.example.com`) with a DNS A record pointing to the server IP
- `htpasswd` (from `apache2-utils`) to generate a bcrypt password hash

> **Note:** Traefik must share a Docker network with this stack (default: `dokploy-network`). Adjust in `docker-compose.yml` if you use a different network name.

## Quick Start

```bash
# 1. Clone the repo
git clone https://github.com/ali-rajabpour/Personal-DoH.git
cd Personal-DoH

# 2. Create your .env from the template
cp env-example .env

# 3. Generate a bcrypt hash for basic auth
htpasswd -Bbn myuser mypassword
# Copy the hash (everything after "myuser:") into DOH_HASHED_PASS in .env

# 4. Edit .env — set your domain and credentials
vim .env

# 5. Start the stack
docker compose up -d
```

After startup the DoH endpoint is available at:

```text
https://resolver.<DOMAIN>/dns-query
```

## Configuration

| File | Purpose |
| ---- | ------- |
| `docker-compose.yml` | Service definitions (doh-server + Unbound + dnscrypt-proxy), volumes, networking |
| `doh-server.conf` | DoH server config: upstream, timeouts, retries |
| `unbound.conf` | Caching resolver: prefetch, serve-expired, forwarding |
| `dnscrypt-proxy.toml` | Upstream DoH config: server priority, failover, bootstrap |
| `app-config` | Startup script that writes Traefik routing + auth config |
| `.env` (from `env-example`) | Domain, path prefix, auth credentials |

## Environment Variables

| Variable | Example | Description |
| -------- | ------- | ----------- |
| `DOMAIN` | `example.com` | Apex domain only — the FQDN is built as `resolver.<DOMAIN>`. |
| `DOH_HTTP_PREFIX` | `/dns-query` | URL path the DoH server listens on. |
| `DOH_SERVER_LISTEN` | `8053` | Internal container port. |
| `DOH_USER` | `myusername` | Username for HTTP Basic Auth. |
| `DOH_HASHED_PASS` | `$2y$05$vI8...` | **Bcrypt hash** from `htpasswd -Bbn`. No `$` escaping needed. |

Credentials are read directly from the mounted `.env` file at container startup (not via Docker Compose environment variable substitution), so `$` signs in the bcrypt hash are never mangled.

> **Important — plain text vs hash:**
>
> | Where | What to use |
> | ----- | ----------- |
> | `.env` file (`DOH_HASHED_PASS`) | The **bcrypt hash** (starts with `$2y$`) |
> | Little Snitch / curl / any client | The **plain text password** you chose when running `htpasswd` |
>
> The server stores only the hash. Clients send the plain password; Traefik hashes it on the fly and compares. **Never put the hash into a client** — it will always return 401.

## Server Configuration

`doh-server.conf` is a TOML file mounted into the container. Supported options:

- **`listen`** — address and port the server binds to (default `":8053"`)
- **`path`** — HTTP path for DNS queries (must match `DOH_HTTP_PREFIX`)
- **`upstream`** — list of upstream DNS resolvers in `<proto>:<host>:<port>` format
- **`timeout`** — seconds before an upstream query times out (default `5`)
- **`tries`** — number of retry attempts across upstreams on failure (default `3`)
- **`verbose`** — enable detailed logging (`true` / `false`)
- **`cert`** / **`key`** — TLS cert/key paths (left empty — Traefik handles TLS)

The default configuration forwards queries to Unbound (`tcp:unbound:53`), which caches responses and forwards cache misses to `dnscrypt-proxy` for encrypted upstream resolution via DoH. You generally do not need to change `doh-server.conf`.

### Unbound (caching layer)

`unbound.conf` provides the aggressive caching that makes repeat queries near-instant:

- **`prefetch: yes`** — when a cached entry reaches 10% remaining TTL, Unbound proactively refreshes it in the background. Popular domains never expire from cache.
- **`serve-expired: yes`** — if a cached entry has expired, Unbound returns the stale data *immediately* (TTL=30) and refreshes in the background. Zero perceived latency on expiry.
- **`cache-min-ttl: 300`** — entries stay cached for at least 5 minutes even if the upstream TTL is shorter.
- **`msg-cache-size: 64m`** / **`rrset-cache-size: 128m`** — generous cache sizes.
- **`qname-minimisation: yes`** — privacy: only sends the minimum necessary query to each upstream hop.

### dnscrypt-proxy (encrypted upstream)

`dnscrypt-proxy.toml` handles the encrypted connection to upstream DoH servers:

- **`server_names`** — `['nextdns', 'cloudflare', 'adguard-dns-doh']` — ordered by priority
- **`lb_strategy = 'first'`** — always use NextDNS; Cloudflare and AdGuard are fallbacks only
- **`doh_servers = true`** — only use DoH (port 443) servers
- **`bootstrap_resolvers`** — plain DNS used *only* to resolve DoH server hostnames on first start

> **Note:** Options like `upstream_selector`, `weight`, and `[cache]` are **not** supported by the `doh-server` binary (they belong to the `doh-client` component). Do not add them to `doh-server.conf`.

## Traefik Integration

This project uses the **Traefik file provider** for routing — not Docker labels. At container startup the `app-config` script:

1. Reads `DOH_USER` and `DOH_HASHED_PASS` directly from the mounted `.env` file (immune to Compose `$` interpolation).
2. Reads `DOMAIN` and `DOH_HTTP_PREFIX` from the container environment.
3. Writes a complete Traefik dynamic config containing:
   - **HTTPS router** with TLS via Let's Encrypt and `doh-auth` basicAuth middleware (priority 200).
   - **HTTP → HTTPS redirect router**.
   - **Service** pointing to `http://doh-server:8053`.
   - **basicAuth middleware** with `user:bcrypt_hash`.

Traefik watches the dynamic config directory and picks up changes automatically. No manual Traefik file editing is required after deployment.

## Client Setup

### Little Snitch (macOS)

Requires **Little Snitch 6.1.3** or later (added DoH password authentication support).

1. Open **Little Snitch Settings → DNS Encryption**.
2. Enable **DNS Encryption**.
3. Set **Encrypted DNS Server** to **Custom**.
4. Choose **DNS over HTTPS (DoH)** as the transport.
5. Fill in the fields:

   | Field | Value |
   | ----- | ----- |
   | Server URL | `https://resolver.example.com/dns-query` |
   | Username | your `DOH_USER` value (e.g. `myusername`) |
   | Password | your **plain text** password — the one you typed into `htpasswd`, **not** the `$2y$` hash |
   | Server SPKI | *(leave empty)* |

6. Click **OK**, then click **Test** to verify.

> **Troubleshooting:**
>
> - **401 Unauthorized** — you are most likely entering the bcrypt hash instead of the plain text password. The Password field must contain the original password, not the `$2y$05$...` string from your `.env` file.
> - **"Wrong URL"** — try the URL without the path (`https://resolver.example.com`) — some versions auto-append `/dns-query`. Also ensure your Mac can resolve the hostname via its current DNS settings before switching.

### macOS / iOS / Little Snitch (DNS profile)

Create an Apple configuration profile (`.mobileconfig`) with the DoH payload. Tools like [dns-profile-creator](https://github.com/niclas-edn/dns-profile-creator) can generate one. Use the server URL with embedded credentials (plain text password, **not** the hash):

```text
https://myuser:mypassword@resolver.example.com/dns-query
```

### Browsers (Firefox, Chrome)

Most browsers do **not** support HTTP Basic Auth for DoH natively. If your browser allows a custom DoH URL, try the embedded-credential format:

```text
https://myuser:mypassword@resolver.example.com/dns-query
```

> **Note:** Firefox supports custom DoH URLs in `about:config` → `network.trr.uri`, but does not reliably pass embedded credentials. Browser DoH with basic auth may require a local proxy such as `dnscrypt-proxy`.

### curl

Use `-u 'user:plainpassword'` — the **plain text** password, not the hash:

```bash
curl -s -H 'accept: application/dns-message' \
  -u 'myuser:mypassword' \
  'https://resolver.example.com/dns-query?dns=AAEBAAABAAAAAAAABmdvb2dsZQNjb20AAAEAAQ'
```

## Verifying the Endpoint

```bash
# Without credentials → expect 401
curl -s -o /dev/null -w '%{http_code}' \
  'https://resolver.example.com/dns-query?dns=AAEBAAABAAAAAAAABmdvb2dsZQNjb20AAAEAAQ'

# With credentials → expect 200
curl -s -o /dev/null -w '%{http_code}' \
  -H 'accept: application/dns-message' \
  -u 'myuser:mypassword' \
  'https://resolver.example.com/dns-query?dns=AAEBAAABAAAAAAAABmdvb2dsZQNjb20AAAEAAQ'

# Wrong credentials → expect 401
curl -s -o /dev/null -w '%{http_code}' \
  -u 'wrong:wrong' \
  'https://resolver.example.com/dns-query?dns=AAEBAAABAAAAAAAABmdvb2dsZQNjb20AAAEAAQ'
```

The base64url string `AAEBAAABAAAAAAAABmdvb2dsZQNjb20AAAEAAQ` is a DNS wire-format query for `google.com A`.

## FAQ

### Can I change the upstream DNS resolvers?

Edit `server_names` in `dnscrypt-proxy.toml`. The available server names come from the [public resolvers list](https://github.com/DNSCrypt/dnscrypt-resolvers). Common choices: `cloudflare`, `adguard-dns-doh`, `nextdns`, `google`, `quad9-dnscrypt-ip4-nofilter-pri`.

### Where are logs?

```bash
# DoH server logs
docker compose logs -f doh-server

# Caching resolver logs
docker compose logs -f unbound

# Upstream DoH resolver logs
docker compose logs -f dnscrypt-proxy
```

### How do I update the container image?

```bash
docker compose pull && docker compose up -d
```

### How do I change the password?

```bash
htpasswd -Bbn myuser mynewpassword
```

Copy the hash into `DOH_HASHED_PASS` in `.env`, then restart:

```bash
docker compose up -d
```

### Why not use `[cache]` or `upstream_selector` in `doh-server.conf`?

These options belong to the `doh-client` component of [m13253/dns-over-https](https://github.com/m13253/dns-over-https), not the server. The server binary rejects them with `unknown option` errors. Caching is handled by Unbound instead.

### Why does the container run as root?

The `app-config` script writes Traefik's dynamic config file into a host-mounted directory owned by `root:root`. The container needs write access to that directory. The DoH server binary itself does not require root privileges.

## License

MIT — see [LICENSE](LICENSE) for details.

---

Made with ☕ by [Ali Rajabpour Sanati](https://rajabpour.com)