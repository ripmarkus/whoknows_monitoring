# Plan: `whoknows_monitoring/` — minimal Prometheus + Grafana stack

## Context

DevOps elective: run Prometheus + Grafana on `65.109.162.80` and monitor the Ruby/Sinatra app and host metrics on `mathiasmortensen.dk` (`91.100.1.101`). GitOps means all config is in Git; pushing to `main` redeploys the server.
Metrics are now exposed securely over HTTPS via an Nginx reverse proxy on the app server (`/metrics` and `/node-metrics`), instead of exposing the raw ports.
The target app does **not** expose `/metrics` yet — adding it is a separate PR on `whoknows_ripmarkus` and is not part of this plan.

Decisions: Grafana public on :3000 with admin auth, no alerting, app-side `/metrics` flagged as prerequisite only.

---

## Repo layout

```
whoknows_monitoring/
├── README.md                         # Setup, secrets, prerequisite note
├── PLAN.md                           # This file
├── .gitignore                        # .env
├── compose.yaml                      # Monitoring stack (monitoring server)
├── .env.example                      # GRAFANA_ADMIN_USER / _PASSWORD
├── prometheus/
│   └── prometheus.yml                # Scrape jobs
├── grafana/provisioning/
│   ├── datasources/prometheus.yml    # Wires Prometheus datasource
│   ├── dashboards/dashboards.yml     # File provider
│   └── dashboards-json/
│       ├── node-exporter-1860.json   # Community dashboard (host metrics)
│       └── ...                       # Other dashboards
├── nginx/
│   └── monitoring.mathiasmortensen.dk # Nginx virtual host for the monitoring dashboard proxy
└── .github/workflows/
    └── deploy.yaml                   # SSH + git pull + docker compose up
```

That's it. No blackbox, no Alertmanager, no postgres_exporter, no separate CI workflow.

---

## What each piece does

**`prometheus/prometheus.yml`** — three scrape jobs:

- `prometheus` → `localhost:9090` (self)
- `node-app` → `https://mathiasmortensen.dk/node-metrics` (host metrics)
- `ruby-app` → `https://mathiasmortensen.dk/metrics` (stays DOWN until app PR lands)

`scrape_interval: 15s`. No rules.

**`compose.yaml` (monitoring server)** — two services:

- `prometheus` (`prom/prometheus`) — mounts `./prometheus`, bound to `127.0.0.1:9090` (reach via SSH tunnel), flag `--web.enable-lifecycle`.
- `grafana` (`grafana/grafana`) — publishes `3000:3000`, reads `GF_SECURITY_ADMIN_USER/_PASSWORD` from `.env`, provisioning tree mounted read-only.

Named volumes `prometheus_data`, `grafana_data`.

**`grafana/provisioning/`** — Prometheus datasource auto-wired to `http://prometheus:9090`; file provider loads one community dashboard (Node Exporter Full, ID 1860). One JSON, nothing hand-rolled.

**`.github/workflows/deploy.yaml`** — single job, triggered on push to `main`. Copies the SSH-key + heredoc pattern from `whoknows_ripmarkus/.github/workflows/CD.yaml:18-30`. Two steps, both via SSH:

1. On `MONITORING_SERVER_IP`:
   ```
   cd /opt/docker/devops/whoknows_monitoring && git fetch origin \
     && git reset --hard origin/main \
     && docker compose pull \
     && docker compose up -d \
     && curl -fsS -X POST http://127.0.0.1:9090/-/reload
   ```
2. On `APP_SERVER_IP`:
   ```
   cd /opt/docker/devops/whoknows_ripmarkus \
     && docker compose -f compose.exporters.yaml pull \
     && docker compose -f compose.exporters.yaml up -d
   ```

**GitHub secrets:** `SSH_KEY`, `SERVER_USER`, `MONITORING_SERVER_IP`, `APP_SERVER_IP`.

**Server `.env` files** (placed by hand once, never in Git):

- Monitoring server: `GRAFANA_ADMIN_USER`, `GRAFANA_ADMIN_PASSWORD`.
- App server: nothing needed for node_exporter.

---

## Prerequisite (separate PR on `whoknows_ripmarkus`)

Add `gem "prometheus-client"` + mount `Prometheus::Client::Rack::Exporter` so `/metrics` returns Prometheus text. Until that merges, the `ruby-app` scrape target shows DOWN — expected and documented in README.

---

## Verification

1. `docker compose config` locally — no errors.
2. `docker compose up -d` locally; open `http://localhost:9090/targets` — `prometheus` UP, remote targets DOWN (expected).
3. `http://localhost:3000`, log in with `.env` creds, Node dashboard auto-listed.
4. Push to `main` → GitHub Actions deploys both servers.
5. After deploy: `up{job="node-app"} == 1` in Prometheus; host metrics visible in Grafana.
6. When `whoknows_ripmarkus` `/metrics` PR merges, `up{job="ruby-app"}` flips to 1 with no change to this repo — proves the GitOps loop works.
