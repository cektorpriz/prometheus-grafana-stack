# Prometheus + Grafana monitoring stack (reference template)

A production-style, Docker Compose monitoring stack for a single Linux host:

- **Prometheus** — metrics collection and alert rules
- **node_exporter** — host metrics (CPU, memory, disk, network)
- **blackbox_exporter** — HTTP probes / uptime checks
- **Alertmanager** — alert routing to Telegram
- **Grafana** — dashboards (Prometheus datasource auto-provisioned)

Self-contained and reusable — a common pattern for monitoring small-to-medium
deployments (one host, several services) without Kubernetes.

## Architecture

```
┌─────────────────┐   scrape (pull)   ┌────────────┐   rules   ┌──────────────┐
│  node_exporter  │◄──────────────────│            │──────────►│ Alertmanager │──► Telegram
└─────────────────┘                   │ Prometheus │           └──────────────┘
┌─────────────────┐   scrape (pull)   │            │
│ blackbox_exporter│◄─────────────────│            │
└─────────────────┘                   └─────┬──────┘
        │ probes http://…/health           │ query (PromQL)
        ▼                                  ▼
   your app / sites                   ┌─────────┐
                                      │ Grafana │
                                      └─────────┘
```

The important mental model: exporters only **expose** metrics; **Prometheus
pulls** them on `scrape_interval`. When a rule fires, Prometheus **pushes** the
alert to Alertmanager. **Grafana pulls** from Prometheus via PromQL — it never
receives anything directly from Prometheus.

## Services & ports

| Service          | Port | Purpose                       |
|------------------|------|-------------------------------|
| Prometheus       | 9090 | scraping + UI + API           |
| node_exporter    | 9100 | host metrics                  |
| blackbox_exporter| 9115 | HTTP probes (`/probe`)        |
| Alertmanager     | 9093 | alert routing (UI + API)      |
| Grafana          | 3000 | dashboards                    |

## Quick start

```bash
cp .env.example .env   # optional: set TELEGRAM_BOT_TOKEN / TELEGRAM_CHAT_ID
docker compose up -d
docker compose ps
```

- Prometheus: http://localhost:9090 → Status → Targets (everything should be UP)
- Grafana: http://localhost:3000 (login from `.env`, default `admin` / `change-me`)

## Configuration

### What to monitor
Edit `prometheus/prometheus.yml` → `blackbox_health` → `targets`. Replace the
example URLs with your real health endpoints (external sites, or internal
services reachable from the `monitoring` network).

### Alert rules
`prometheus/alerts.yml` ships with `SiteDown`, `HighCPU`, `HighMemory`.
Adjust thresholds or add rules as needed.

### Telegram alerts
Set `TELEGRAM_BOT_TOKEN` and `TELEGRAM_CHAT_ID` in `.env`. Alertmanager reads
them via environment expansion (`--config.expand-env=true`). Without them the
stack still runs; only Telegram delivery is inactive.

### Grafana dashboards
The Prometheus datasource is auto-provisioned. Import community dashboards, e.g.
"Node Exporter Full" (ID `1860`) → datasource = Prometheus.

## Notes & security

- `node_exporter` requires a **Linux** host (`pid: host` + `/proc`, `/sys` mounts).
- Containers talk to each other by service name on the `monitoring` bridge.
- By default the ports are bound to `0.0.0.0`. In production, restrict them with
  a firewall / reverse proxy, or bind to `127.0.0.1` and reach them via an SSH
  tunnel:

  ```bash
  ssh -L 9090:localhost:9090 -L 3000:localhost:3000 user@host
  ```

## License

MIT
