# whoknows_monitoring

GitOps-style Prometheus + Grafana stack that monitors the
[`whoknows_ripmarkus`](../whoknows_ripmarkus) Ruby/Sinatra app.

- **Monitoring server**: `65.109.162.80` — runs Prometheus + Grafana.
- **App server**: `91.100.1.101` — runs the app (port 8080) and a
  `node_exporter` sidecar (port 9100) deployed from this repo.
- **GitOps loop**: every push to `main` SSHes to both servers, pulls this
  repo, and runs `docker compose up -d`.

## Quick start (local)

```
cp .env.example .env         # edit GRAFANA_ADMIN_PASSWORD
docker compose up -d
```

- Prometheus: http://localhost:9090 (targets at /targets)
- Grafana: http://localhost:3000 (login with values from `.env`)

The `ruby-app` scrape target will be `DOWN` until the prerequisite below is
merged — expected.

## First-time server setup

### Monitoring server (65.109.162.80)

```
sudo mkdir -p /opt/docker/devops
cd /opt/docker/devops
git clone <this-repo-url> whoknows_monitoring
cd whoknows_monitoring
cp .env.example .env
# edit .env — set a strong GRAFANA_ADMIN_PASSWORD
docker compose up -d
```

Grafana is reachable at http://65.109.162.80:3000. Prometheus is bound to
`127.0.0.1:9090` — reach it via SSH tunnel:
`ssh -L 9090:localhost:9090 user@65.109.162.80`.

### App server (91.100.1.101)

```
sudo mkdir -p /opt/docker/devops
cd /opt/docker/devops
git clone <this-repo-url> whoknows_monitoring
cd whoknows_monitoring/exporters
docker compose -f compose.exporters.yaml up -d
```

Firewall: restrict port 9100 to the monitoring server's IP.

```
sudo ufw allow from 65.109.162.80 to any port 9100 proto tcp
```

## GitHub Secrets

| Secret                  | What                                     |
|-------------------------|------------------------------------------|
| `SSH_KEY`               | Private key authorised on both servers   |
| `SERVER_USER`           | SSH user (same on both)                  |
| `MONITORING_SERVER_IP`  | `65.109.162.80`                          |
| `APP_SERVER_IP`         | `91.100.1.101`                           |

## Prerequisite: app `/metrics` endpoint

The `ruby-app` scrape target in `prometheus/prometheus.yml` points at
`91.100.1.101:8080/metrics`, which does not exist yet. A follow-up PR on
`whoknows_ripmarkus` needs to:

1. Add `gem "prometheus-client"` to the Gemfile.
2. Mount `Prometheus::Client::Rack::Exporter` as Rack middleware in
   `app.rb` so `GET /metrics` returns Prometheus text format.

Once that merges, `up{job="ruby-app"}` flips to `1` automatically — no
change needed in this repo. That is the GitOps loop working as intended.

## Layout

```
compose.yaml                         Monitoring stack (prometheus + grafana)
prometheus/prometheus.yml            Scrape jobs
grafana/provisioning/                Datasource + dashboard auto-loading
exporters/compose.exporters.yaml     node_exporter for the app server
.github/workflows/deploy.yaml        SSH deploy on push to main
```

See [`PLAN.md`](./PLAN.md) for the full design rationale.
