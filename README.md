# Crisler Umbrel App Store

A custom [Umbrel Community App Store](https://github.com/getumbrel/umbrel) by Crisler, distributing self-hosted observability apps for umbrelOS.

---

## Apps

### lgtm-stack
Full Grafana observability stack — Loki, Grafana, Tempo, Mimir + Alloy log collector. Grafana uses **PostgreSQL** as its database (replaces the default SQLite).

- **Grafana UI**: port `3010`  
- **OTLP gRPC**: port `4317`    
- **OTLP HTTP**: port `4318`

> To also collect logs from **Komodo DinD**, uncomment the volume and Alloy config blocks marked with "Komodo" in `lgtm-stack/docker-compose.yml` and `lgtm-stack/config.alloy`.

---

## Adding this store to Umbrel

1. In umbrelOS, go to **App Store → Community App Stores**
2. Paste your GitHub repository URL (e.g. `https://github.com/crislerwintler/umbrel-lgtm`)
3. Install **LGTM Stack** from the store

---

## Development

Each app lives in its own folder:

```
umbrel-app-store.yml        ← store metadata
lgtm-stack/
  umbrel-app.yml            ← app manifest
  docker-compose.yml        ← service definitions
  config.alloy              ← Alloy collector pipeline
```

App IDs follow the store prefix convention: `crisler-<app-name>` (e.g. `crisler-lgtm-stack`).

To run locally (outside Umbrel):

```bash
cd lgtm-stack
DOCKER_CONFIG=/tmp/docker-cfg docker compose up -d
```

> Grafana will be available at http://localhost:3010 (admin / admin)
