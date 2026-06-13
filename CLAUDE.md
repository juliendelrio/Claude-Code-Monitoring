# CLAUDE.md

Guidance for Claude Code when working in this repository.

## Project

Docker monitoring stack for **Claude Code usage telemetry**. Claude Code emits
OpenTelemetry metrics (OTLP/gRPC); this stack collects, stores and visualizes
them in Grafana, exposed behind an existing Traefik reverse proxy with TLS.

## Architecture

```
Claude Code --(OTLP/gRPC :4317)--> otel-collector --(Prometheus exporter :8889)-->
prometheus (scrape 10s, 90d retention) --> grafana (provisioned dashboard)
```

All deployment files live in [src/](src/):

- [src/compose.yaml](src/compose.yaml) — service definitions (otel-collector, prometheus, grafana) + Traefik labels
- [src/otel-config.yaml](src/otel-config.yaml) — OTLP receiver + Prometheus exporter pipeline
- [src/prometheus.yml](src/prometheus.yml) — scrape config targeting the collector
- [src/datasource.yml](src/datasource.yml) — Grafana Prometheus datasource (auto-provisioned)
- [src/dashboards.yml](src/dashboards.yml) — Grafana dashboard provider (auto-provisioned, points at src/dashboards/)
- [src/dashboards/](src/dashboards/) — one JSON per themed dashboard (overview, A/B testing, consumption, productivity), grouped under a "Claude Code" folder in Grafana

## Conventions

- Run docker compose from the `src/` directory (where `compose.yaml` and `.env` live).
- **Secrets** go only in `src/.env` (gitignored). Never hardcode credentials; reference
  them via `${VAR}` in `compose.yaml` and document new vars in `src/.env.example`.
- **No instance-specific values in `compose.yaml`**: hostnames (`OTEL_HOST`, `GRAFANA_HOST`),
  Traefik network (`TRAEFIK_NETWORK`) and cert resolver (`TRAEFIK_CERTRESOLVER`) all come
  from `.env`. Keep it host-agnostic — if you add a new instance-specific value, externalize it.
- The stack assumes an external Traefik on a Docker network (default `web`, TLS via the
  configured cert resolver, default `letsencrypt`).
- Prometheus/Grafana state lives in named docker volumes (`prometheus-data`, `grafana-data`),
  not in the repo.

## Common commands

```bash
# from src/
docker compose up -d
docker compose logs -f otel-collector
docker compose down
docker compose config        # validate compose + env interpolation
```
