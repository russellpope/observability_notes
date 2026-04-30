# Observability Platform Architecture

## Overview

A lab-scale observability platform providing unified metrics, logging, and alerting for Pure Storage arrays and Linux client machines. Deployed as a Docker Compose stack on a single Ubuntu VM provisioned on vSphere 8.

## Data Flow

```
┌─────────────────────────────────────────────────────────────────────┐
│                     Observability VM (Docker Compose)               │
│                                                                     │
│  ┌─────────────┐    ┌────────────┐    ┌───────────────┐            │
│  │  Prometheus  │◄───│  Pure FA    │    │  OTel         │            │
│  │  :9090       │    │  Exporter   │    │  Collector    │            │
│  │             ◄─────│  :9490      │    │  :4317 (gRPC) │            │
│  │              │    └────────────┘    │  :4318 (HTTP) │            │
│  │              │◄────────────────────│  :8889 (prom) │            │
│  │              │                      └───────┬───────┘            │
│  └──────┬───────┘                              ▲                    │
│         │                                      │ OTLP push          │
│         ▼                                      │                    │
│  ┌──────────────┐    ┌────────────┐            │                    │
│  │   Grafana    │◄───│    Loki    │            │                    │
│  │   :3000      │    │    :3100   │            │                    │
│  └──────────────┘    └─────┬──────┘            │                    │
│         │                  ▲                   │                    │
│  ┌──────────────┐          │                   │                    │
│  │ Alertmanager │          │                   │                    │
│  │   :9093      │          │                   │                    │
│  └──────────────┘          │                   │                    │
└─────────────────────────────┼───────────────────┼────────────────────┘
                              │                   │
          ┌───────────────────┼───────────────────┼───────────┐
          │                   │                   │           │
          ▼                   │                   │           ▼
   ┌─────────────┐            │                   │    ┌─────────────┐
   │ Linux       │            │                   │    │ Linux       │
   │ Clients     │            │                   │    │ Clients     │
   │ (Alloy)     │────────────┘                   │    │ (Alloy)     │
   │ metrics+logs│                                │    │ metrics+logs│
   └─────────────┘                                │    └─────────────┘
                                                  │
   ┌──────────────────────────────────────────────┘
   │
   │  ┌──────────────┐         ┌──────────────────┐
   ├──│ FlashBlade 1 │         │ FlashArray 1..7  │
   │  │ (Purity 4.8) │         │ (pull via FA     │
   │  │ OTel push    │         │  exporter)       │──► Prometheus
   │  └──────────────┘         └──────────────────┘    scrapes
   │  ┌──────────────┐
   └──│ FlashBlade 2 │
      │ (Purity 4.8) │
      │ OTel push    │
      └──────────────┘
```

## Components

### Metrics Collection

| Component | Purpose | Model |
|-----------|---------|-------|
| **Prometheus** | Central metrics store, scrapes all pull-based targets, accepts remote_write | Pull + Push |
| **FlashArray native /metrics** | Purity 6.4+ serves OpenMetrics directly — Prometheus scrapes `https://<array>/metrics/<namespace>` with bearer auth | Pull (direct) |
| **pure-fb-om-exporter** | Stateless proxy: Prometheus passes FB endpoint + API token per scrape (`flashblade_*` → `purefb_*` metrics) | Pull |
| **OpenTelemetry Collector** | Receives OTLP metrics pushed from FlashBlade native exporter (`flashblade_*` metrics); exposes as Prometheus endpoint on :8889 | Push (FB → OTel) + Pull (Prom scrapes OTel) |
| **Grafana Alloy** | Runs on each Linux client; ships host metrics via `remote_write` to Prometheus and logs via push to Loki | Push |

> **FlashBlade dual-path note:** FBs can be monitored via either path — OpenMetrics proxy (`purefb_*` series) or native OTel push (`flashblade_*` series). They produce different metric names and label schemas; canned dashboards in `docker-compose/grafana/dashboards/` use `purefb_*` names and only cover the OpenMetrics path.

### Log Aggregation

| Component | Purpose |
|-----------|---------|
| **Grafana Loki** | Log storage and query engine, indexed by labels only (lightweight) |
| **Grafana Alloy** | Ships logs from Linux clients (journald, syslog, application logs) to Loki |

### Visualization & Alerting

| Component | Purpose |
|-----------|---------|
| **Grafana** | Unified dashboards for metrics (Prometheus) and logs (Loki) |
| **Alertmanager** | Routes Prometheus alerts to email, Slack, webhooks, etc. |

## Deployment Stack

- **Infrastructure**: Terraform → vSphere 8 → Ubuntu VM
- **Configuration**: Ansible → Docker install, Compose deployment, Alloy agent on clients
- **Runtime**: Docker Compose on the VM

## Phased Rollout

### Phase 1 — Metrics (current)
- Provision VM via Terraform
- Deploy Prometheus + Grafana + Pure FA exporter + OTel Collector via Docker Compose
- Deploy Grafana Alloy to Linux clients via Ansible
- Configure FlashBlade OTel export to push to the collector
- Import Pure Storage Grafana dashboards

### Phase 2 — Logging
- Add Loki to Docker Compose stack
- Configure Alloy on Linux clients to ship logs
- Build correlated metrics + logs dashboards in Grafana

### Phase 3 — Alerting & Automation
- Configure Alertmanager with routing rules
- Define Prometheus alert rules (array capacity, performance, host health)
- Webhook receivers for automation triggers (Ansible AWX, custom services)
- Grafana OnCall for escalation (optional)

## Reference Links

### Pure Storage
- [pure-fa-openmetrics-exporter](https://github.com/PureStorage-OpenConnect/pure-fa-openmetrics-exporter) — FlashArray Prometheus exporter
- [pure-fb-openmetrics-exporter](https://github.com/PureStorage-OpenConnect/pure-fb-openmetrics-exporter) — FlashBlade Prometheus exporter (fallback for pre-4.8 firmware)
- [Pure Storage Grafana Dashboards](https://grafana.com/grafana/dashboards/?search=pure+storage) — Community dashboards

### Grafana Stack
- [Grafana](https://grafana.com/docs/grafana/latest/) — Visualization
- [Grafana Alloy](https://grafana.com/docs/alloy/latest/) — Unified telemetry collector
- [Grafana Loki](https://grafana.com/docs/loki/latest/) — Log aggregation

### Prometheus
- [Prometheus](https://prometheus.io/docs/introduction/overview/) — Metrics collection and storage
- [Alertmanager](https://prometheus.io/docs/alerting/latest/alertmanager/) — Alert routing

### OpenTelemetry
- [OpenTelemetry Collector](https://opentelemetry.io/docs/collector/) — Vendor-neutral telemetry pipeline
- [OTel Collector Contrib](https://github.com/open-telemetry/opentelemetry-collector-contrib) — Community receivers, processors, exporters
- [OTLP Specification](https://opentelemetry.io/docs/specs/otlp/) — Protocol spec

### Infrastructure
- [Terraform vSphere Provider](https://registry.terraform.io/providers/hashicorp/vsphere/latest/docs) — VM provisioning
- [Docker Compose Specification](https://docs.docker.com/compose/compose-file/) — Container orchestration

### Container Images
| Image | Registry |
|-------|----------|
| `prom/prometheus` | Docker Hub |
| `grafana/grafana` | Docker Hub |
| `grafana/loki` | Docker Hub |
| `grafana/alloy` | Docker Hub |
| `prom/alertmanager` | Docker Hub |
| `otel/opentelemetry-collector-contrib` | Docker Hub |
| `quay.io/purestorage/pure-fa-om-exporter` | Quay.io |

## Network Ports

| Service | Port | Protocol | Direction |
|---------|------|----------|-----------|
| Grafana | 3000 | HTTP | Inbound (UI) |
| Prometheus | 9090 | HTTP | Inbound (UI/API) |
| Loki | 3100 | HTTP | Inbound (from Alloy) |
| Alertmanager | 9093 | HTTP | Inbound (UI) |
| OTel Collector | 4317 | gRPC | Inbound (OTLP from FlashBlades) |
| OTel Collector | 4318 | HTTP | Inbound (OTLP HTTP) |
| OTel Collector | 8889 | HTTP | Internal (Prometheus scrapes) |
| Pure FA Exporter | 9490 | HTTP | Internal (Prometheus scrapes) |
