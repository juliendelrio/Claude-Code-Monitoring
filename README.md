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

| Service          | Role                                                        | Exposed at                     |
|------------------|-------------------------------------------------------------|--------------------------------|
| `otel-collector` | Receives OTLP/gRPC, re-exposes metrics in Prometheus format | `$OTEL_HOST` (TLS + basicauth) |
| `prometheus`     | Scrapes the collector, stores time series (90d retention)   | internal only                  |
| `grafana`        | Displays the auto-provisioned dashboards                    | `$GRAFANA_HOST` (TLS)          |

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
- An existing **Traefik** reverse proxy attached to an external Docker network,
  with a certificate resolver configured (defaults assume a network named `web`
  and a `letsencrypt` resolver — both overridable via `.env`).
- DNS records pointing your chosen `OTEL_HOST` and `GRAFANA_HOST` to the host.

The stack is host-agnostic: everything instance-specific (hostnames, Traefik
network and cert resolver) is set through environment variables, so the same
`compose.yaml` works for any deployment.

## Setup

1. Create your config:

   ```bash
   cd src
   cp .env.example .env
   ```

   Then edit `.env`:

   | Variable                | Required | Description                                                                 |
   |-------------------------|----------|-----------------------------------------------------------------------------|
   | `OTEL_HOST`             | yes      | Public hostname for the OTLP collector (Claude Code sends metrics here).     |
   | `GRAFANA_HOST`          | yes      | Public hostname for the Grafana UI.                                         |
   | `TRAEFIK_NETWORK`       | no       | External Docker network Traefik is on. Default: `web`.                       |
   | `TRAEFIK_CERTRESOLVER`  | no       | Traefik certificate resolver name. Default: `letsencrypt`.                   |
   | `OTEL_AUTH_USERS`       | yes      | Traefik basicauth protecting the collector. Generate with `htpasswd -nb user password`; **escape every `$` as `$$`**. |
   | `GRAFANA_ADMIN_PASSWORD`| yes      | Grafana admin password (login user: `admin`).                              |

2. Start the stack (from `src/`):

   ```bash
   docker compose up -d
   ```

3. Open `https://$GRAFANA_HOST` and log in to Grafana. The datasource and
   dashboards are provisioned automatically.

## Point Claude Code at the collector

On the machine running Claude Code, enable telemetry and send it to your
collector endpoint, for example:

```bash
export CLAUDE_CODE_ENABLE_TELEMETRY=1
export OTEL_METRICS_EXPORTER=otlp
export OTEL_EXPORTER_OTLP_PROTOCOL=grpc
export OTEL_EXPORTER_OTLP_ENDPOINT=https://$OTEL_HOST
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
