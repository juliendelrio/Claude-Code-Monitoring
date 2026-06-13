# Claude-Code-Monitoring

A self-hosted Docker stack to **monitor your Claude Code usage** — cost, tokens,
sessions, cache efficiency and productivity — through OpenTelemetry, Prometheus
and Grafana.

## How it works

Claude Code can export usage metrics over OpenTelemetry. This stack receives them
and turns them into a live dashboard:

```
Claude Code ──(OTLP/gRPC :4317)──▶ otel-collector ──(Prometheus exporter :8889)──▶
prometheus (scrape every 10s, 90-day retention) ──▶ grafana (provisioned dashboard)
```

| Service          | Role                                                        | Exposed at                          |
|------------------|-------------------------------------------------------------|-------------------------------------|
| `otel-collector` | Receives OTLP/gRPC, re-exposes metrics in Prometheus format | `otel.tools.juliendelrio.fr` (TLS + basicauth) |
| `prometheus`     | Scrapes the collector, stores time series (90d retention)   | internal only                       |
| `grafana`        | Displays the auto-provisioned dashboard                     | `tokens.tools.juliendelrio.fr` (TLS) |

The dashboards live in [src/dashboards/](src/dashboards/), grouped under a
"Claude Code" folder in Grafana and cross-linked via a dropdown:

- **Vue d'ensemble** — cost, tokens, sessions, active time, cache-hit ratio, avg cost per session / commit
- **A/B testing** — cost & tokens per experiment over time, cache ratio, cost per session / commit
- **Consommation** — tokens by type and query source, cost by model, detailed token table
- **Productivité & qualité** — lines of code, edit acceptance rate, commits / PRs, edit decisions by language

Each dashboard has `$model` and `$experiment` template variables to filter the
whole view (e.g. to compare experiments).

## Prerequisites

- Docker + Docker Compose v2
- An existing **Traefik** reverse proxy attached to an external Docker network
  named `web`, with a `letsencrypt` certificate resolver configured.
- DNS records pointing `otel.tools.juliendelrio.fr` and
  `tokens.tools.juliendelrio.fr` to the host (adjust the hostnames in
  [src/compose.yaml](src/compose.yaml) for your own domain).

## Setup

1. Configure secrets:

   ```bash
   cd src
   cp .env.example .env
   ```

   Then edit `.env`:

   - `OTEL_AUTH_USERS` — Traefik basicauth credentials protecting the collector
     endpoint. Generate with `htpasswd -nb user password` and **escape every `$`
     as `$$`** so docker compose doesn't interpolate it.
   - `GRAFANA_ADMIN_PASSWORD` — admin password for Grafana (login user: `admin`).

2. Start the stack (from `src/`):

   ```bash
   docker compose up -d
   ```

3. Open `https://tokens.tools.juliendelrio.fr` and log in to Grafana. The
   datasource and dashboard are provisioned automatically.

## Point Claude Code at the collector

On the machine running Claude Code, enable telemetry and send it to your
collector endpoint, for example:

```bash
export CLAUDE_CODE_ENABLE_TELEMETRY=1
export OTEL_METRICS_EXPORTER=otlp
export OTEL_EXPORTER_OTLP_PROTOCOL=grpc
export OTEL_EXPORTER_OTLP_ENDPOINT=https://otel.tools.juliendelrio.fr
export OTEL_EXPORTER_OTLP_HEADERS="Authorization=Basic <base64 user:password>"
```

## Operations

```bash
# from src/
docker compose logs -f otel-collector   # follow collector logs
docker compose config                    # validate compose + env interpolation
docker compose down                      # stop (named volumes keep their data)
```

Prometheus and Grafana data are stored in the named volumes `prometheus-data`
and `grafana-data`; they survive `docker compose down`.

## Security notes

- All secrets live in `src/.env`, which is gitignored. Never commit credentials.
- The collector's gRPC endpoint listens in clear text inside the Docker network;
  TLS and basicauth are terminated by Traefik in front of it.
- Grafana anonymous access is disabled.

## License

See [LICENSE](LICENSE).
