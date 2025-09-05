# Self-Hosted DNS-over-HTTPS (DoH)

Run your own DNS-over-HTTPS endpoint behind Traefik using Docker Compose. The container forwards DNS queries to an upstream resolver such as Cloudflare (1.1.1.1), while Traefik handles TLS termination, automatic certificates, rate-limiting and optional compression.

---

## Contents
1. [Features](#features)  
2. [Requirements](#requirements)  
3. [Quick Start](#quick-start)  
4. [Environment Variables](#environment-variables)  
5. [Traefik Notes](#traefik-notes)  
6. [Verifying the Endpoint](#verifying-the-endpoint)  
7. [FAQ](#faq)

---

## Features
• 🛡 **Private** – keep your DNS queries on your own infrastructure.  
• 🔒 **Automatic TLS** via Traefik and Let’s Encrypt.  
• ⚡️ **Fast** DoH server image (`satishweb/doh-server`).  
• 🚦 **Rate-limiting** & **compression** middleware pre-configured.  
• 🏷 **Single file** deployment with `.env` for easy overrides.

## Requirements
* A server with Docker & Docker Compose
* Traefik (v2) already running and listening on ports 80/443
* A public hostname (`doh.example.com`) pointing to the server’s IP

> **Note** Traefik must share a user-defined Docker network with this stack (default: `dokploy-network`). Adjust in `docker-compose.yaml` if you use a different network.

## Quick Start
```bash
# 1. Clone repo & enter directory
$ git clone <repo-url> && cd Personal-DoH

# 2. Copy environment template and edit values
$ cp env-example .env
$ vim .env  # or nano, code, etc.

# 3. Launch the stack
$ docker compose up -d  # or docker-compose up -d
```
After the containers start you should have a working DoH endpoint at

```
https://<SUBDOMAIN>.<DOMAIN><DOH_HTTP_PREFIX>
# Example: https://doh.example.com/dns-query
```

## Environment Variables
| Variable | Example | Description |
|----------|---------|-------------|
| `DOMAIN` | `example.com` | Apex (root) domain managed in DNS. |
| `SUBDOMAIN` | `doh` | Subdomain that points to this server. Final FQDN will be `${SUBDOMAIN}.${DOMAIN}`. |
| `DOH_HTTP_PREFIX` | `/dns-query` | URL path Traefik will route to the DoH container. |
| `DOH_SERVER_LISTEN` | `8053` | Internal container port for the DoH service. |
| `UPSTREAM_DNS_SERVER` | `udp:1.1.1.1:53` | (In compose) Upstream DNS resolver in `<proto>:<ip>:<port>` format. Change if desired. |

All variables are loaded by Docker Compose from the `.env` file at runtime.

## Traefik Notes
The compose file sets the following Traefik labels:
* Router rule: `Host(`${SUBDOMAIN}.${DOMAIN}`) && Path(`${DOH_HTTP_PREFIX}`)`
* Service port: `${DOH_SERVER_LISTEN}`
* TLS enabled + automatic Let’s Encrypt cert resolver (`letsencrypt`)
* Security middleware: rate limiting (100 req/10s, burst 50) & HSTS redirect

If you use a different cert resolver name or want to tweak limits, edit the labels accordingly.

## Verifying the Endpoint
```bash
# JSON format (RFC 8484)
curl -H 'accept: application/dns-json' \
  "https://${SUBDOMAIN}.${DOMAIN}${DOH_HTTP_PREFIX}?name=cloudflare.com&type=A"

# Wire-format (binary) query using doh-cli (optional)
doh-cli query cloudflare.com A -u "https://${SUBDOMAIN}.${DOMAIN}${DOH_HTTP_PREFIX}"
```
A successful response confirms your DoH service is working.

## FAQ
### Can I use a different upstream DNS resolver?
Yes. Override `UPSTREAM_DNS_SERVER` in `docker-compose.yaml` or via an additional environment variable stanza.

### Where are logs?
Docker logs for the DoH container: `docker compose logs -f doh-server`.

### How do I update?
Pull the latest image and re-deploy:
```bash
$ docker compose pull && docker compose up -d
```

---

Licensed under the MIT License.