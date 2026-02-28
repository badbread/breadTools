# breadTools

**A self-hosted web dashboard and SSH terminal menu for running home lab management tools on a Synology NAS.**

A Flask-based control panel that replaces ad-hoc SSH sessions with a real-time streaming web UI. Tools run inside a Docker container and stream output live to the browser via Server-Sent Events. Also accessible as an interactive Rich terminal menu over SSH.

~600 lines of Python | 8 tool scripts | Dockerized on Synology NAS

---

## What It Does

breadTools provides a central interface for running maintenance scripts across a home lab:

- **RPDB Artwork** — Generate missing artwork reports and inject TMDB fallback posters for the RPDB poster service
- **Network** — Scan active devices on the LAN; add new services to Nginx Proxy Manager and Pi-hole DNS; purge Cloudflare CDN cache
- **Docker** — Health-check all running containers
- **Storage** — Generate NAS disk usage reports
- **Backup** — Verify recent Proxmox Backup Server jobs are complete

Each tool runs as a subprocess, streaming stdout live to the browser. No polling — the output panel updates in real time via SSE.

---

## Architecture

```
breadtools/
  app.py ................... Flask web server (port 8765) + SSE streaming
  menu.py .................. Rich terminal menu for SSH access
  scripts/
    rpdb_report.py ......... RPDB missing artwork scan
    rpdb_tmdb_injector.py .. TMDB fallback poster injection
    network_scanner.py ..... LAN device discovery
    docker_health.py ....... Docker container status check
    storage_report.py ...... NAS disk usage report
    backup_verify.py ....... PBS backup job verification
    add_service_dns.py ..... NPM + Pi-hole DNS registration
    cloudflare_cache_purge.py  Cloudflare full-zone cache purge
  templates/
    design1_modern_dark.html  Default dark UI
    design2_glass.html ....... Glass morphism variant
    design3_minimal.html ..... Minimal variant
  data/
    settings.json ............ API keys (TMDB, Cloudflare)
  Dockerfile
  docker-compose.yml
```

### Tool Execution Flow

```
Browser click
    |
    v
POST /api/tool/{id}/run
    |
    +--> subprocess.Popen (shell=True, stdout=PIPE)
    |         |
    |    output_queue (threading.Queue)
    |
GET /api/tool/{id}/output  (SSE stream)
    |
    +--> EventSource in browser
              |
         Live output panel updates
```

All tools are registered in a single `TOOLS` dict in `app.py`. Adding a new tool is one dict entry + one script file — no other changes needed.

### Tool Configuration Options

| Field | Purpose |
|-------|---------|
| `script` | Python file in `/scripts/` |
| `args` | Static CLI args (supports `${ENV_VAR}` expansion) |
| `form` | Dynamic form fields → becomes a modal dialog |
| `confirm` | Prompts for confirmation before running |
| `run_local` | Returns a command string instead of executing (for tools that need LAN access outside the container) |

---

## UI

Three design variants switchable via `?design=` URL parameter:

- `?design=modern` — Dark theme with grid layout (default)
- `?design=glass` — Glass morphism
- `?design=minimal` — Minimal

All designs share the same Flask backend and SSE streaming. Tools with form fields open a modal dialog to collect input before running.

---

## Tech Stack

| Layer | Technology |
|-------|-----------|
| **Runtime** | Python 3.11, Docker, Synology NAS |
| **Web Framework** | Flask + Jinja2 |
| **Streaming** | Server-Sent Events (SSE) via `stream_with_context` |
| **Terminal UI** | Rich (panels, tables, prompts) |
| **Process Management** | `subprocess.Popen` + `threading.Thread` + `queue.Queue` |
| **Settings** | JSON file (`data/settings.json`) with env var injection |
| **DNS / Proxy** | Pi-hole API, Nginx Proxy Manager API |
| **CDN** | Cloudflare API v4 |

---

## Access Methods

**Web UI** — `http://nas-ip:8765` — Full dashboard with live streaming output

**SSH terminal** — `ssh breadtools@nas-ip -p 2222` — Auto-launches Rich menu on login; same tool registry as the web UI

---

## Development Approach

Built and maintained using **Claude Code** as a pair-programming partner. Claude Code reads the codebase, edits files directly, and maintains project memory files across sessions. Source is live-mounted into the container, so changes deploy on next tool run without a rebuild.

---

## Status

**v1.0** &mdash; Running on Synology NAS. Tools added as needed.

This is a private infrastructure project. Source code is not published, but the architecture is documented here for portfolio reference.

---

*Built with Python, Flask, and a preference for having buttons instead of typing commands.*
