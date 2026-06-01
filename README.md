# whoknows_monitoring

GitOps-style Prometheus + Grafana stack that monitors the
[`whoknows_ripmarkus`](../whoknows_ripmarkus) Ruby/Sinatra app.

- **Monitoring server**: `vss-vm-1` (`65.109.162.80`) — runs Prometheus + Grafana.
- **App server**: `tivieserver` (`mathiasmortensen.dk`, `91.100.1.101`) — runs the app,
  database, and `node_exporter`, all fronted by Nginx with TLS. Deployed from
  `whoknows_ripmarkus`, not from this repo.
- **GitOps loop**: every push to `main` SSHes to the monitoring server, pulls this
  repo, and runs `docker compose up -d`.

## Architecture Note

The app server securely exposes its metrics via the Nginx reverse proxy:

- `https://mathiasmortensen.dk/metrics` — proxies to the Ruby web container
- `https://mathiasmortensen.dk/node-metrics` — proxies to the `node_exporter` container

Nginx restricts access to both endpoints to the monitoring server IP
(`allow 65.109.162.80; deny all;`), so the rest of the internet can't scrape them.

## Quick start (local)

```bash
cp .env.example .env
docker compose up -d
```

- Prometheus: http://localhost:9090 (targets at `/targets`)
- Grafana: http://localhost:3000 (login with values from `.env`)

The remote scrape targets (`node-app`, `ruby-app`) will show DOWN locally — expected,
your laptop isn't on Nginx's IP allow-list.

## First-time server setup (monitoring server)

```
sudo mkdir -p /opt/docker/devops
cd /opt/docker/devops
git clone <this-repo-url> whoknows_monitoring
cd whoknows_monitoring
cp .env.example .env
# edit .env — set a strong GRAFANA_ADMIN_PASSWORD
docker compose up -d
```

- Grafana: http://65.109.162.80:3000
- Prometheus: http://65.109.162.80:9090

No setup needed on the app server — `node_exporter` ships with the app from
`whoknows_ripmarkus`'s own CD, and Nginx on `tivieserver` handles access control.

## GitHub Secrets

| Secret                   | What                                            |
| ------------------------ | ----------------------------------------------- |
| `SSH_KEY`                | Private key authorised on the monitoring server |
| `MONITORING_SERVER_USER` | SSH user on the monitoring server               |
| `MONITORING_SERVER_IP`   | `65.109.162.80`                                 |

## Layout

```
compose.yaml                         Monitoring stack (prometheus + grafana)
prometheus/prometheus.yml            Scrape jobs
grafana/provisioning/                Datasource + dashboard auto-loading
.github/workflows/deploy.yaml        SSH deploy on push to main
```

See [`PLAN.md`](./PLAN.md) for the full design rationale.
