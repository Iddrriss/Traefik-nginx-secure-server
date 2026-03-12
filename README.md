# Secure Edge Proxy Deployment
### CYS 411 — Group 10

A containerized web architecture implementing Traefik v3 as an edge proxy and NGINX as a backend service. The implementation demonstrates SSL/TLS termination, automatic service discovery via Docker labels, security middleware enforcement, and strict service isolation through Docker bridge networking.

---

## Table of Contents

1. [Project Overview](#project-overview)
2. [Architecture](#architecture)
3. [Security Features](#security-features)
4. [File Structure](#file-structure)
5. [Prerequisites](#prerequisites)
6. [Deployment](#deployment)
7. [Demonstration Sequence](#demonstration-sequence)
8. [Assignment Requirements](#assignment-requirements)

---

## Project Overview

This project addresses a fundamental challenge in web infrastructure security — how do you expose a web service to the public internet without exposing the web server itself?

The solution implements a **reverse proxy architecture** where Traefik serves as the sole public-facing entry point. All inbound traffic — regardless of protocol or origin — is intercepted, inspected, and processed by Traefik before any request reaches the NGINX backend. The NGINX container holds no publicly exposed ports and is reachable only through the internal Docker bridge network.

Two NGINX services are deployed to demonstrate Traefik's automatic service discovery capability. Service 1 (`cysgroup10.local`) represents the primary deployment. Service 2 (`site2.cysgroup10.local`) is introduced post-deployment to demonstrate that Traefik detects and routes to new containers automatically — without modifying any configuration file or restarting any existing container.

---

## Architecture

```
                        CLIENT / BROWSER
                               │
                    HTTP :80 / HTTPS :443
                               │
                    ┌──────────▼──────────┐
                    │   TRAEFIK CONTAINER  │
                    │                      │
                    │  ┌────────────────┐  │
                    │  │ TLS Termination │  │
                    │  │ HTTP→HTTPS :301 │  │
                    │  └───────┬────────┘  │
                    │          │ HTTPS      │
                    │  ┌───────▼────────┐  │
                    │  │  Docker Router  │◄─┼── Docker Socket
                    │  │  Reads Labels   │  │   Auto Discovery
                    │  └───────┬────────┘  │
                    │          │            │
                    │  ┌───────▼────────┐  │
                    │  │   Middleware    │  │
                    │  │ Rate Limit      │  │
                    │  │ Security Headers│  │
                    │  └───────┬────────┘  │
                    │          │            │
                    │   Dashboard :8080     │
                    └──────────┬────────────┘
                               │ HTTP :80 internal
                    ┌──────────▼────────────────┐
                    │  DOCKER BRIDGE NETWORK     │
                    │          "web"             │
                    │                            │
                    │  ┌──────────┐ ┌─────────┐ │
                    │  │  NGINX   │ │  NGINX2  │ │
                    │  │ Service1 │ │ Service2 │ │
                    │  │ :80 int  │ │ :80 int  │ │
                    │  │ No public│ │ No public│ │
                    │  │ port     │ │ port     │ │
                    │  └──────────┘ └─────────┘ │
                    └────────────────────────────┘
```

### Traffic Flow

**HTTPS Request (normal flow):**
1. Client sends request to `cysgroup10.local`
2. DNS resolves to `127.0.0.1` via hosts file
3. Traefik receives on port 443
4. TLS handshake established — connection encrypted
5. Docker Router checks hostname against label rules
6. Middleware enforces rate limiting and security headers
7. Request forwarded internally to `nginx:80` via HTTP
8. NGINX serves `index.html`
9. Response travels back through Traefik, encrypted to client

**HTTP Request (redirect flow):**
1. Client sends HTTP request to port 80
2. Traefik intercepts at web entry point
3. Issues 301 redirect to HTTPS
4. Client follows redirect — same flow as above

**Auto Discovery flow:**
1. New container starts with `traefik.enable=true` label
2. Traefik reads Docker socket — detects new container
3. Router automatically created from label definitions
4. New service immediately accessible — zero config change

---

## Security Features

### 1. Service Isolation

NGINX containers expose no public ports. The only publicly accessible container is Traefik. This means:

- Direct access to NGINX is architecturally impossible from outside Docker
- An attacker cannot target NGINX directly — they must go through Traefik
- The Docker bridge network acts as an isolated internal segment

This mirrors the DMZ principle in enterprise network architecture — public-facing infrastructure (Traefik) sits between the internet and backend services (NGINX), ensuring backend servers are never directly reachable.

### 2. TLS Termination

SSL/TLS is terminated at Traefik using a self-signed RSA 4096-bit certificate. Key design decisions:

- **Encryption scope:** Traffic between client and Traefik is encrypted. Traffic between Traefik and NGINX is plain HTTP — acceptable because the Docker bridge network is a trusted, isolated environment with no external exposure
- **Centralised certificate management:** One certificate at the proxy layer covers all backend services regardless of how many containers exist
- **Certificate generation:**
```bash
openssl req -x509 -newkey rsa:4096 \
  -keyout traefik/certs/key.pem \
  -out traefik/certs/cert.pem \
  -sha256 -days 365 -nodes
```

### 3. Automatic HTTP to HTTPS Redirect

Port 80 entry point is configured to issue a permanent 301 redirect to the HTTPS entry point. Unencrypted access to any service is impossible — the redirect is enforced at the infrastructure level, not the application level.

### 4. Rate Limiting

Configured via Traefik middleware in `routes.yml`:

```yaml
rate-limit:
  rateLimit:
    average: 100
    burst: 50
```

- `average: 100` — sustained request rate of 100 requests per second
- `burst: 50` — allows temporary spike of 50 additional requests above average
- Requests exceeding this threshold receive a `429 Too Many Requests` response
- Applied to all routers via Docker label: `traefik.http.routers.nginx.middlewares=rate-limit@file`
- Mitigates Distributed Denial of Service (DDoS) and brute force attacks

### 5. Security Headers

Applied globally via `secure-headers` middleware:

| Header | Value | Attack Mitigated |
|---|---|---|
| `Strict-Transport-Security` | `max-age=31536000` | SSL stripping |
| `X-Frame-Options` | `DENY` | Clickjacking |
| `X-Content-Type-Options` | `nosniff` | MIME type sniffing |
| `Referrer-Policy` | `strict-origin-when-cross-origin` | Information leakage |
| `X-XSS-Protection` | `1; mode=block` | Cross-site scripting |
| `Content-Security-Policy` | `default-src 'self'` | Malicious resource injection |

These headers instruct the browser on how to handle the response — adding a security enforcement layer at the application level that complements the network and transport level controls.

### 6. Automatic Service Discovery

Traefik's Docker provider monitors the Docker socket (`unix:///var/run/docker.sock`) for container lifecycle events. When a container starts with `traefik.enable=true` and associated routing labels, Traefik automatically:

- Creates a router based on the `Host()` rule
- Registers the container as a backend service
- Applies any specified middleware
- Makes the service immediately accessible

No modification to `traefik.yml` or `routes.yml` is required. This is the distinction between **file-based routing** (static, manual) and **Docker provider routing** (dynamic, automatic).

---

## File Structure

```
411 project/
│
├── docker-compose.yml          # Service orchestration
├── .gitignore                  # Excludes certificates from version control
├── README.md                   # Project documentation
│
├── traefik/
│   ├── traefik.yml             # Static configuration — entry points, providers, API
│   ├── routes.yml              # Dynamic configuration — middleware, TLS certificates
│   └── certs/
│       ├── cert.pem            # SSL certificate — gitignored
│       └── key.pem             # Private key — gitignored, never commit
│
├── nginx/
│   └── default.conf            # NGINX server block configuration
│
├── website/
│   └── index.html              # Service 1 static content
│
└── website2/
    └── index.html              # Service 2 static content
```

### Configuration File Responsibilities

**docker-compose.yml**
Defines all containers, their images, volume mounts, network membership, exposed ports and Docker labels. The Docker socket mount (`/var/run/docker.sock`) is what enables Traefik's auto discovery.

**traefik/traefik.yml**
Static configuration loaded at startup. Defines entry points (`:80`, `:443`, `:8080`), enables the dashboard, configures the Docker provider and file provider, and sets log level. Changes require a container restart.

**traefik/routes.yml**
Dynamic configuration watched continuously by Traefik (`watch: true`). Contains middleware definitions (rate limiting, security headers) and TLS certificate paths. Changes take effect immediately without restart.

**nginx/default.conf**
Minimal NGINX configuration. Listens on port 80 internally, serves files from `/usr/share/nginx/html`, defaults to `index.html`. NGINX has no knowledge of TLS — all encryption is handled upstream by Traefik.

---

## Prerequisites

- Docker Desktop installed and running
- OpenSSL available (included with Git Bash on Windows)
- Administrator access (for hosts file modification)

---

## Deployment

### Step 1 — Add Hostnames to Hosts File

Run PowerShell as Administrator:

```powershell
Add-Content -Path "C:\Windows\System32\drivers\etc\hosts" -Value "127.0.0.1    cysgroup10.local"
Add-Content -Path "C:\Windows\System32\drivers\etc\hosts" -Value "127.0.0.1    site2.cysgroup10.local"
```

### Step 2 — Generate SSL Certificates

```bash
openssl req -x509 -newkey rsa:4096 \
  -keyout traefik/certs/key.pem \
  -out traefik/certs/cert.pem \
  -sha256 -days 365 -nodes
```

### Step 3 — Deploy

```bash
docker-compose up -d
```

### Step 4 — Verify

```bash
docker ps
```

Expected output — three containers running:
- `traefik` — ports 80, 443, 8080 bound publicly
- `nginx` — port 80 internal only
- `nginx2` — port 80 internal only

### Access Points

| Service | URL |
|---|---|
| Service 1 | https://cysgroup10.local |
| Service 2 | https://site2.cysgroup10.local |
| Dashboard | http://localhost:8080/dashboard/ |

**Note:** Browser will display a certificate warning for self-signed certificates. Select Advanced → Proceed to continue.

---

## Demonstration Sequence

### 1. Dashboard Overview
Navigate to `http://localhost:8080/dashboard/`

Point out:
- Three entry points: `:80`, `:443`, `:8080`
- HTTP Routers — 100% success rate
- HTTP Middlewares — `rate-limit@file` and `secure-headers@file` active
- All routers suffixed with `@docker` — confirms auto discovery is active

### 2. HTTPS Enforcement
Navigate to `http://cysgroup10.local` (HTTP)

Observe automatic 301 redirect to `https://cysgroup10.local`. Unencrypted access is impossible — enforced at infrastructure level.

### 3. Service 1
Navigate to `https://cysgroup10.local`

Accept certificate warning. Service 1 is live — routed via `nginx@docker` router, TLS terminated at Traefik, served by NGINX Alpine internally.

### 4. Live Auto Discovery
Stop nginx2 first to reset the demonstration:

```bash
docker-compose stop nginx2
```

Refresh dashboard — `nginx2@docker` router disappears automatically.

Now start nginx2 without touching any configuration:

```bash
docker-compose up -d nginx2
```

Refresh dashboard — `nginx2@docker` router reappears automatically.

Navigate to `https://site2.cysgroup10.local` — immediately accessible.

**This demonstrates auto discovery:** Traefik detected the container via Docker socket, read its labels, created the router, and began routing traffic — with zero manual intervention.

### 5. Security Headers Verification
Open browser developer tools → Network tab → select any request → inspect Response Headers.

Visible headers confirm middleware is active:
- `Strict-Transport-Security`
- `X-Frame-Options: DENY`
- `X-Content-Type-Options: nosniff`
- `X-XSS-Protection`
- `Content-Security-Policy`

---

## Assignment Requirements

| Requirement | Implementation |
|---|---|
| Deploy NGINX container serving static website | Two NGINX Alpine containers — `nginx` and `nginx2` |
| Deploy Traefik as reverse proxy | Traefik v3.6.8 — sole public entry point |
| Automatically discover Docker services | Docker provider + Docker socket + container labels |
| Route traffic to NGINX container | Hostname-based routing via `Host()` rules |
| Provide HTTPS using self-signed certificate | RSA 4096-bit cert — TLS terminated at Traefik |
| Service isolation | NGINX containers expose no public ports |
| Scalability | New services added without configuration changes |
| Security depth | Rate limiting + six security headers + HTTP redirect |

---

*CYS 411 — Group 10*
