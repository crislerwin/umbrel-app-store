# AGENT.md - Umbrel App Development Guide

Guidelines for creating effective apps for umbrelOS (Umbrel).

## Overview

Umbrel apps run inside isolated Docker containers. The only requirement is that they should serve a web-based UI. Server apps without a UI should serve a simple web page with connection details, QR codes, and setup instructions.

Users never need CLI access on Umbrel.

## App Structure

Each app lives in its own directory named with the app ID:

```
<app-id>/
├── umbrel-app.yml       # App manifest (metadata)
├── docker-compose.yml   # Service definitions
├── exports.sh           # (Optional) Environment variable exports
└── logo.png             # App icon (optional, or use URL)
```

## App ID Naming

- **Format:** Lowercase letters, numbers, and dashes only
- **Pattern:** `<store-prefix>-<app-name>` for community stores
- **Examples:** `btc-rpc-explorer`, `lgtm-stack`, `lgtm-loki`
- **Must be human-readable and recognizable**

## File: umbrel-app.yml

```yaml
manifestVersion: 1
id: <app-id>
name: <Display Name>
tagline: Short description (keep it under 60 chars)
icon: <URL to icon OR relative path to logo.png>
category: <Finance|Media|Networking|Productivity|Social|Developer Tools|Monitoring>
version: "1.0.0"
port: <port-number>
description: >-
  Multi-line description using YAML folded style (>).
  Supports markdown-style formatting.
  Keep paragraphs short.
  
  - Feature bullet 1
  - Feature bullet 2

developer: <Author Name>
website: <https://example.com>
submitter: <GitHub Username>
submission: <https://github.com/.../repo>
repo: <https://github.com/original/project>
support: <https://community.example.com or support URL>
gallery:
  - <https://i.imgur.com/xxx.jpg>
  - <https://i.imgur.com/yyy.jpg>
releaseNotes: >-
  What's new in this version?
  
dependencies: []      # Other Umbrel apps this requires
path: ""              # Leave empty for proxy-based routing
defaultUsername: ""   # Leave empty if no auth
defaultPassword: ""   # Leave empty if no auth
```

## File: docker-compose.yml

```yaml
version: "3.7"

services:
  # REQUIRED: Umbrel's app proxy service
  app_proxy:
    environment:
      # APP_HOST format: <app-id>_<service-name>_1
      # The '_1' suffix is REQUIRED
      APP_HOST: <app-id>_web_1
      APP_PORT: <web-server-port>

  # Your app service
  web:
    image: <image>:<tag>@sha256:<digest>
    restart: on-failure
    stop_grace_period: 1m
    
    # Only expose non-HTTP ports here
    # The web port is handled by app_proxy
    ports:
      - "<additional-port>:<additional-port>"  # Optional
    
    volumes:
      # Persistent data storage
      - ${APP_DATA_DIR}/data:/app/data
      
      # Optional: Mount other app data (read-only)
      # - ${APP_LIGHTNING_NODE_DATA_DIR}:/lnd:ro
      # - ${APP_BITCOIN_DATA_DIR}:/bitcoin:ro
    
    environment:
      # Umbrel provides these env vars:
      # $DEVICE_HOSTNAME - Device hostname (e.g., "umbrel")
      # $DEVICE_DOMAIN_NAME - .local domain (e.g., "umbrel.local")
      # $TOR_PROXY_IP / $TOR_PROXY_PORT - Tor proxy access
      # $APP_HIDDEN_SERVICE - Tor hidden service address
      # $APP_PASSWORD - Auto-generated password for auth
      # $APP_SEED - 256-bit deterministic seed
      
      CUSTOM_VAR: value

  # Optional: Additional services (db, cache, etc.)
  db:
    image: postgres:15
    restart: on-failure
    volumes:
      - ${APP_DATA_DIR}/db:/var/lib/postgresql/data
```

## Critical Rules

### App Proxy (REQUIRED)
- Every app MUST have an `app_proxy` service
- `APP_HOST` must follow format: `<app-id>_<service-name>_1` (include `_1`)
- `APP_PORT` is the internal port your web server listens on
- Do NOT expose the web port directly - let app_proxy handle it
- Only expose additional ports (like gRPC, custom APIs) via `ports:`

### Service Naming
- The service name in docker-compose becomes the DNS hostname
- Other services reference it by name: `http://web:8080`
- Keep names simple: `web`, `api`, `db`, `loki`, `alloy`

### Data Persistence
- Use `${APP_DATA_DIR}` for all persistent data
- Mount to appropriate paths inside container
- Structure: `${APP_DATA_DIR}/data`, `${APP_DATA_DIR}/config`, etc.

### Docker Image Best Practices
- Use multi-stage builds for smaller images
- Don't run as root (use `USER 1000` or similar)
- Use specific tags, not `latest`
- Include sha256 digest for security: `@sha256:abc123...`
- Support both ARM64 and AMD64 architectures
- Use lightweight base images (Alpine, slim variants)

### Networking
- Apps on the same Umbrel share a Docker network
- Services can communicate by hostname
- External access only through app_proxy or explicit port mappings

## Environment Variables Reference

Umbrel automatically provides these environment variables:

| Variable | Description |
|----------|-------------|
| `DEVICE_HOSTNAME` | Device hostname (e.g., "umbrel") |
| `DEVICE_DOMAIN_NAME` | .local domain (e.g., "umbrel.local") |
| `TOR_PROXY_IP` | Local IP of Tor proxy |
| `TOR_PROXY_PORT` | Port of Tor proxy |
| `APP_HIDDEN_SERVICE` | Tor hidden service address |
| `APP_PASSWORD` | Auto-generated app password |
| `APP_SEED` | 256-bit deterministic seed (hex) |
| `APP_DATA_DIR` | Persistent data directory path |
| `APP_BITCOIN_DATA_DIR` | Bitcoin Core data (if available) |
| `APP_LIGHTNING_NODE_DATA_DIR` | LND data (if available) |

## Community App Store Structure

For custom/community app stores:

```
repo-root/
├── umbrel-app-store.yml    # Store metadata
├── <app1>/
│   ├── umbrel-app.yml
│   ├── docker-compose.yml
│   └── ...
├── <app2>/
│   └── ...
```

### umbrel-app-store.yml

```yaml
id: "<store-id>"
name: "<Store Display Name>"
apps:
  - <app-id-1>
  - <app-id-2>
  - <app-id-3>
```

## Common Patterns

### Multi-Container Apps
```yaml
services:
  app_proxy:
    environment:
      APP_HOST: myapp_web_1
      APP_PORT: 3000

  web:
    image: myapp/web:latest
    depends_on:
      - db

  db:
    image: postgres:15
    volumes:
      - ${APP_DATA_DIR}/db:/var/lib/postgresql/data
```

### Custom Configuration
Mount a config directory or file:
```yaml
volumes:
  - ${APP_DATA_DIR}/config/myapp.conf:/etc/myapp/config.conf
  - ${APP_DATA_DIR}/config:/etc/myapp/config
```

### Log Aggregation (with Alloy)
```yaml
alloy:
  image: grafana/alloy:latest
  volumes:
    - /var/run/docker.sock:/var/run/docker.sock:ro
  environment:
    ALLOY_CONFIG: |
      discovery.docker "host" { ... }
      loki.source.docker "host" { ... }
      loki.write "loki" { ... }
```

## Testing

### Local Testing (outside Umbrel)
```bash
cd <app-id>
# Use override file for local testing
DOCKER_CONFIG=/tmp/docker-cfg docker compose -f docker-compose.yml -f docker-compose.override.yml up -d
```

### On Umbrel Device
1. Add store URL in Umbrel: App Store → Community App Stores
2. Install and test
3. Check logs: Settings → Troubleshoot → App Logs

## Troubleshooting

| Issue | Solution |
|-------|----------|
| "Address cannot be found" | Check APP_HOST format includes `_1` suffix |
| Permission denied | Add `user: "0:0"` or fix volume permissions |
| Port already in use | Remove `ports:` for web port, use app_proxy only |
| Config not loading | Check config file path in volume mount |
| App not appearing | Check umbrel-app-store.yml apps list |

## Resources

- Official Docs: https://github.com/getumbrel/umbrel-apps
- Docker Compose Reference: https://docs.docker.com/compose/
- Umbrel Community: https://community.umbrel.com
