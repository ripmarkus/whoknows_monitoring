# Plan: `whoknows_monitoring/` — minimal Prometheus + Grafana stack

## Context

DevOps elective. Two servers:

- **`vss-vm-1`** (`65.109.162.80`) — runs Prometheus + Grafana. This repo.
- **`tivieserver`** (`mathiasmortensen.dk`, `91.100.1.101`) — runs the Ruby/Sinatra web app, PostgreSQL, and `node_exporter`, all fronted by Nginx with TLS. Deployed from `whoknows_ripmarkus`.

GitOps: pushing to `main` here redeploys the monitoring stack on `vss-vm-1`. The `node_exporter` on `tivieserver` is deployed alongside the app from `whoknows_ripmarkus` and is not managed by this repo.

The web app's `/metrics` endpoint is exposed by `whoknows_ripmarkus` (via `gem "prometheus-client"` + `Prometheus::Client::Rack::Exporter`).

---

## Metric flow

```
Prometheus on vss-vm-1 (65.109.162.80)
        │ HTTPS scrape, every 15s
        ▼
Nginx on tivieserver (443, TLS)
        │ IP allow-list: only 65.109.162.80
        ├── /metrics        → web:8080/metrics             (app metrics)
        └── /node-metrics   → node_exporter:9100/metrics   (host metrics)
```

Both `/metrics` paths on `tivieserver` are gated by `allow 65.109.162.80; deny all;` in Nginx, so they're inaccessible from anywhere else on the internet.

Decisions: Grafana public on `:3000` with admin auth, no Alertmanager, no Blackbox.

---

## Repo layout

```
whoknows_monitoring/
├── README.md                         # Setup, secrets
├── PLAN.md                           # This file
├── .gitignore                        # .env
├── compose.yaml                      # Monitoring stack (vss-vm-1)
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
│   └── monitoring.mathiasmortensen.dk  # Vhost for Grafana — present in repo, NOT deployed yet
└── .github/workflows/
    └── deploy.yaml                   # SSH + git pull + docker compose up
```

No `compose.exporters.yaml` here — exporters live in `whoknows_ripmarkus` now.

---

## What each piece does

**`prometheus/prometheus.yml`** — three scrape jobs:

- `prometheus` → `localhost:9090` (self)
- `node-app` → `https://mathiasmortensen.dk/node-metrics` (host metrics, via Nginx)
- `ruby-app` → `https://mathiasmortensen.dk/metrics` (app metrics, via Nginx)

`scrape_interval: 15s`. No rules.

**`compose.yaml` (on `vss-vm-1`)** — two services:

- `prometheus` (`prom/prometheus:v2.54.1`) — mounts `./prometheus`, publishes `9090:9090`, flag `--web.enable-lifecycle`.
- `grafana` (`grafana/grafana:11.2.0`) — publishes `3000:3000`, reads `GF_SECURITY_ADMIN_USER/_PASSWORD` from `.env`, provisioning tree mounted read-only.

Named volumes `prometheus_data`, `grafana_data`.

**`grafana/provisioning/`** — Prometheus datasource auto-wired to `http://prometheus:9090`; file provider loads Node Exporter Full (dashboard 1860). One JSON, nothing hand-rolled.

**`nginx/monitoring.mathiasmortensen.dk`** — vhost that proxies `monitoring.mathiasmortensen.dk` → `127.0.0.1:3000`. HTTP only, no TLS. Present in the repo but Nginx is not installed on `vss-vm-1` yet, so Grafana is reached directly on `:3000` for now. Activating it would mean: install Nginx on `vss-vm-1`, drop the vhost in `/etc/nginx/sites-enabled/`, change Grafana to bind `127.0.0.1:3000`, and run certbot for TLS. Out of scope before the exam.

**`.github/workflows/deploy.yaml`** — single job, triggered on push to `main`. SSH-key + heredoc pattern copied from `whoknows_ripmarkus/.github/workflows/CD.yaml:18-30`. One step:

On `MONITORING_SERVER_IP` (`vss-vm-1`):

```
cd /opt/docker/devops/whoknows_monitoring && git fetch origin \
  && git reset --hard origin/main \
  && docker compose pull \
  && docker compose up -d \
  && curl -fsS -X POST http://127.0.0.1:9090/-/reload
```

No second step for the app server — `node_exporter` ships with the app via `whoknows_ripmarkus`'s own CD.

**GitHub secrets:** `SSH_KEY`, `SERVER_USER`, `MONITORING_SERVER_IP`.

**Server `.env` file** (placed by hand once, never in Git):

- `vss-vm-1`: `GRAFANA_ADMIN_USER`, `GRAFANA_ADMIN_PASSWORD`.

---

## Verification

1. `docker compose config` locally — no errors.
2. `docker compose up -d` locally; open `http://localhost:9090/targets` — `prometheus` UP, remote targets DOWN (expected: laptop isn't on the allow-list).
3. `http://localhost:3000`, log in with `.env` creds, Node dashboard auto-listed.
4. Push to `main` → GitHub Actions deploys `vss-vm-1`.
5. After deploy: `up{job="node-app"} == 1` and `up{job="ruby-app"} == 1` in Prometheus; host and app metrics visible in Grafana.

---

## Known drift from the plan (post-exam TODOs)

These are conscious gaps between this document and what's actually running. Documenting so it's not a surprise.

- **Nginx vhost not deployed on `vss-vm-1`.** The vhost file is in the repo, but Nginx isn't installed, so Grafana is reached directly on `:3000`. The vhost is also HTTP-only — needs certbot before going live.

- **Prometheus bound to `0.0.0.0:9090`** rather than loopback. Anyone on the internet can query metrics. Read-only and no app secrets in there, but worth tightening to `127.0.0.1:9090` + SSH tunnel later.
