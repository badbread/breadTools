# breadTools

**Self-hosted admin dashboard and infrastructure provisioning toolkit running on a Synology NAS.**

A Flask-based control panel that replaces ad-hoc SSH sessions and manual cloud provider UI clicks with a real-time streaming web interface. Tools run inside a Docker container and stream output live to the browser via Server-Sent Events.

The highlight: a **self-service SaaS instance provisioner** that creates production-ready Railway deployments end-to-end — project, database, DNS, environment config — from a single form submission.

---

## Self-Service Railway Provisioner

The flagship tool. One form, one click, one fully deployed production SaaS instance.

**What it does end-to-end:**
1. Creates a new Railway project via GraphQL API
2. Provisions a PostgreSQL 16 service with a randomly generated secure password
3. Links the web service to a GitHub repository
4. Injects all required environment variables (DATABASE_URL, admin credentials, session secrets, API keys)
5. Registers a custom subdomain on Railway (e.g., `client.simplyraffle.com`)
6. Creates the Cloudflare CNAME DNS record via REST API
7. Retrieves Railway's TXT verification token and creates the Cloudflare TXT record
8. Streams every step back to the browser in real time via SSE

**Technical details:**
- Railway has no official Python SDK — all interactions are raw **GraphQL mutations** via `urllib.request` with full error parsing and response validation
- Cloudflare DNS records are managed via **REST API** with bearer token authentication
- Password hashing uses **bcrypt (12 rounds)** at provisioning time — not SHA-256
- Environment variable interpolation handles Railway's `${{{}}}` triple-brace template syntax (JSON escaping + template engine escaping)
- Domain verification required **GraphQL schema introspection** to discover — Railway stores TXT verification data as `verificationDnsHost` and `verificationToken` fields on the `CustomDomainStatus` type, not in the `dnsRecords` array where you'd expect it. This is undocumented and was only found by querying the schema directly
- The Cloudflare TXT record value must be wrapped in quotes to match Railway's expected format — a detail caught by comparing against known-good records

**Troubleshooting solved during development:**
- **Prisma migration lockup on Railway:** Builds hanging during `npx prisma migrate deploy`. Root cause: a custom `railway.toml` start command was losing the PATH that made `prisma` available. Fix: removed the override and let Railway use its default build pipeline
- **Cloudflare proxy breaks Railway SSL:** Railway's initial SSL certificate generation fails if Cloudflare proxies the DNS record (Cloudflare intercepts the cert validation request). Fix: create the CNAME as DNS-only, let Railway issue the cert, then proxying is safe
- **Railway volumes can't be attached via API:** The provisioner prints a post-deploy checklist for manual steps that the current API doesn't support
- **Missing User-Agent header:** Cloudflare was blocking Railway API requests made without a User-Agent — added the header to all outbound requests

---

## DNS Audit Tool

A read-only diagnostic that cross-references Railway deployments against Cloudflare DNS records to identify stale, orphaned, or misconfigured entries.

Queries both APIs and reports:
- CNAME records pointing to Railway projects that no longer exist
- TXT verification records for deleted domains
- Railway domains missing their Cloudflare DNS records
- Certificate status for all active custom domains

Built to prevent DNS record accumulation as instances are created and decommissioned over time.

---

## Service Registration (NPM + Pi-hole)

One-button service onboarding: registers a new service in both **Nginx Proxy Manager** (reverse proxy with automatic SSL) and **Pi-hole** (local DNS resolution) simultaneously.

**What it automates:**
1. Creates an NPM proxy host entry with the service's internal IP, port, and subdomain
2. Configures SSL certificate provisioning via Let's Encrypt
3. Registers the local DNS record in Pi-hole so the service resolves immediately on the LAN

Without this tool, adding a new service means: log into NPM, create a proxy host, configure SSL settings, log into Pi-hole, add a DNS record. With it: fill in one form, click run.

**Pi-hole v6 API migration:**
Pi-hole v6 changed the API path from `/admin/api.php` to `/api/` and restructured authentication from token-based query parameters to session-based bearer tokens. This broke the registration tool silently — requests returned 200 OK with empty responses instead of errors. Required significant debugging to identify since the failure mode gave no indication the API had changed.

---

## Architecture

### Tool Execution Flow

```
Browser form submit
       |
       v
POST /api/tool/{id}/run
       |
       +--> subprocess.Popen (stdout=PIPE, stderr=PIPE)
       |         |
       |    threading.Queue (non-blocking read)
       |
GET /api/tool/{id}/output  (SSE stream)
       |
       +--> EventSource in browser
                |
           Live output panel updates line-by-line
```

### Auto-Generated Forms

Any tool can declare a `"form": {}` schema in the tool registry and breadTools auto-generates a modal dialog for it. Flask converts form fields to `--flag value` CLI arguments and streams the subprocess output via SSE. **No HTML changes required to add new tools** — the UI is entirely data-driven from the registry.

### Three Access Methods

| Method | Interface | Use Case |
|--------|-----------|----------|
| **Web UI** | Flask dashboard (port 8765) | Primary — forms, streaming output, settings |
| **SSH** | Rich terminal menu (port 2222) | Quick access from any terminal |
| **PowerShell** | Windows integration | Run tools from Windows workstation |

### Live-Mounted Source

The scripts directory is NAS-mounted into the container. Script changes are picked up immediately on the next button press — no container rebuild needed. Only `app.py` or template changes require a restart.

---

## Tool Registry

| Category | Tools |
|----------|-------|
| **Infrastructure** | Railway instance provisioner, Railway DNS audit, Cloudflare cache purge |
| **Network** | LAN device scanner, NPM + Pi-hole DNS registration |
| **Docker** | Container health checker |
| **Storage** | NAS disk usage reporter |
| **Backup** | Proxmox Backup Server job verifier |
| **Media** | RPDB missing artwork report, TMDB poster injector |

---

## Tech Stack

| Layer | Technology |
|-------|-----------|
| **Runtime** | Python 3.11, Docker, Synology NAS |
| **Web** | Flask, Jinja2, Server-Sent Events |
| **APIs** | Railway GraphQL, Cloudflare REST v4, Pi-hole v6, Nginx Proxy Manager |
| **Security** | bcrypt password hashing, bearer token auth, env var injection |
| **Terminal** | Rich (panels, tables, prompts) |
| **Process** | `subprocess.Popen` + `threading.Thread` + `queue.Queue` |

---

## Development Approach

Built and maintained using **AI-assisted development** (Claude Code). The provisioner's multi-API orchestration, GraphQL schema introspection, and DNS automation were developed iteratively with rapid feedback loops — a dedicated test button was built specifically to validate DNS behavior against existing Railway projects without spinning up full deployments.

Source code is not published. This README documents the architecture and technical decisions for portfolio reference.

---

*Built because clicking through Railway's dashboard and Cloudflare's DNS panel for every new tenant deployment wasn't going to scale.*
