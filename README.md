# rp-observability — a Pure Storage observability sandbox

A working, reproducible lab for monitoring Pure Storage FlashArrays and
FlashBlades with open-source observability tooling. Built incrementally,
kept as a **live notebook** — every dead-end, workaround, and design
decision is written down alongside the code so the next person (or
future-you) can pick up mid-thought.

**What this repo is:**
- An end-to-end sandbox for Pure + Prometheus + Grafana + Loki + OTel
- A reference implementation showing both `pure-*-om-exporter` (OpenMetrics
  proxy) and native Purity//FB OTel export, side-by-side
- A curated dashboard pack — canned Pure dashboards patched to work on
  current exporter versions, plus a family of unified cross-path
  dashboards we built in this repo
- A running log of what broke, why, and how we fixed it — in `RUNBOOK.md`,
  `docs/HOWTO-flashblade-monitoring.md`, and `docs/GAPS.md`

**What this repo isn't:**
- A production-ready platform. Ports are plaintext inside the lab,
  Grafana has a single admin user, there's no backup strategy, and
  certificates are off on the FB OTel export. Fine for a lab — not for
  prod. Skim `RUNBOOK.md → "What we haven't done yet"` for the honest list.
- A one-click installer. Adding an FA takes one vault edit and a playbook
  run; adding a whole new observability host is the full Ansible flow.

## Architecture at a glance

```
┌─────────────────────────────────────────────────────────────────────────┐
│ Observability host (Ubuntu + Docker Compose)                            │
│                                                                         │
│  Grafana   Prometheus   Loki   Alertmanager   OTel Collector            │
│    │           │          │          │               │                  │
│    └── UI ◄────┴──────────┴──────────┘               │                  │
│                │                                      │ OTLP/gRPC       │
│                │ scrape                               │ :4317 (plain)   │
│                ▼                                      │                 │
│          ┌──────────────┐                             │                 │
│          │ pure-fb-om-  │◄──┐                         │                 │
│          │  exporter    │   │ stateless proxy         │                 │
│          └──────────────┘   │                         │                 │
└────────┬─────────────┬──────┼─────────────────────────┼─────────────────┘
         │             │      │                         │
     HTTPS           HTTPS    HTTPS                    OTLP push
     (native)        (proxy)  (direct)                 (native)
         │             │      │                         │
         ▼             ▼      ▼                         ▼
  ┌──────────────┐ ┌────────────────┐ ┌────────────────────────────┐
  │ FlashArrays  │ │ FlashBlade A   │ │ FlashBlade B               │
  │ (7 in lab)   │ │ (OpenMetrics)  │ │ (native OTel push)         │
  │ native       │ │ purefb_* metrics│ │ flashblade_* metrics       │
  │ purefa_*     │ └────────────────┘ └────────────────────────────┘
  └──────────────┘

  Linux clients (rp-util2, rp-util3) running Grafana Alloy:
    → remote_write host metrics to Prometheus
    → push journald + syslog to Loki
```

## Repo map

```
.
├── README.md                       ← you are here
├── ARCHITECTURE.md                 ← stack topology, phased rollout
├── RUNBOOK.md                      ← deploy, day-2, troubleshooting
├── lab-manifest.yml                ← gitignored; lab inventory per site
│
├── ansible/
│   ├── ansible.cfg
│   ├── inventory/
│   │   ├── hosts.yml.example       ← template to copy
│   │   ├── hosts.yml               ← gitignored; your host list
│   │   └── group_vars/all/
│   │       ├── vars.yml            ← non-secret dashboard endpoints, retention
│   │       └── vault.yml           ← ENCRYPTED; FA/FB tokens, Grafana pw
│   ├── playbooks/
│   │   ├── prepare-storage.yml     ← mounts /dev/sdb at /var/lib/docker (run FIRST)
│   │   ├── deploy-stack.yml        ← deploys the obs stack + compose up
│   │   ├── deploy-alloy.yml        ← deploys Grafana Alloy to linux_clients
│   │   ├── generate-tfvars.yml     ← Terraform flow (unused; VMs hand-built)
│   │   └── templates/
│   │       ├── prometheus.yml.j2   ← renders FA/FB scrape jobs from vault
│   │       ├── pure-fa-exporter.yml.j2
│   │       ├── docker-compose.env.j2
│   │       └── terraform.tfvars.j2
│   └── roles/
│       └── alloy/                  ← Grafana Alloy install + config
│
├── docker-compose/
│   ├── docker-compose.yml          ← the stack definition
│   ├── prometheus/prometheus.yml   ← rendered by Ansible; example in repo
│   ├── prometheus/alerts/          ← Prom alert rules (starter set)
│   ├── grafana/
│   │   ├── provisioning/           ← datasources + dashboard provider
│   │   └── dashboards/             ← live dashboard JSON (bind-mounted)
│   ├── loki/loki-config.yml
│   ├── alertmanager/alertmanager.yml
│   ├── otel-collector/otel-collector-config.yml
│   └── pure-fa-exporter/           ← legacy; kept for reference
│
├── dashboards/                     ← PORTABLE JSON for external import
│   └── README.md                   ← import instructions
│
├── docs/
│   ├── HOWTO-flashblade-monitoring.md  ← end-to-end ingestion + dashboard guide
│   └── GAPS.md                         ← OTel vs OpenMetrics metric gap tracker
│
└── terraform/                      ← vSphere VM provisioning (unused in current flow)
```

## Where to go

| If you want to… | Start here |
|---|---|
| **Deploy the stack from scratch** | [`RUNBOOK.md`](RUNBOOK.md) → "First-time deployment" |
| **Add a new FlashArray or FlashBlade** | [`RUNBOOK.md`](RUNBOOK.md) → "Day-2 flows" |
| **Understand FB ingestion (OM + OTel) in depth** | [`docs/HOWTO-flashblade-monitoring.md`](docs/HOWTO-flashblade-monitoring.md) |
| **Build a new dashboard or fix one** | [`docs/HOWTO-flashblade-monitoring.md`](docs/HOWTO-flashblade-monitoring.md) → Parts 4–6, plus "Lessons learned" |
| **Export a dashboard to share externally** | [`dashboards/README.md`](dashboards/README.md) + HOWTO Part 7 |
| **Understand what's missing on OTel vs OM** | [`docs/GAPS.md`](docs/GAPS.md) |
| **Troubleshoot a broken panel or scrape** | [`RUNBOOK.md`](RUNBOOK.md) → "Troubleshooting cheatsheet" + HOWTO troubleshooting table |
| **See the lab wiring / component roles** | [`ARCHITECTURE.md`](ARCHITECTURE.md) |

## Current lab state

The stack is live on `rp-util1` (10.21.101.6) with:

- **7 FlashArrays** scraped via native Purity OpenMetrics (`/metrics/<namespace>`
  direct with bearer auth, no exporter container)
- **2 FlashBlades**, one via OpenMetrics proxy (`pure-fb-om-exporter`), one
  via native OTel push — intentional dual-path to compare coverage
- **2 Linux clients** (`rp-util2`, `rp-util3`) running Grafana Alloy for
  host metrics + journald/syslog
- **15+ Grafana dashboards**: 8 canned Pure dashboards (patched), 5 unified
  cross-path dashboards, 2 domain-focused (Object Storage, OTel Overview)

Everything's reachable at:
- Grafana: http://10.21.101.6:3000 (admin / see vault)
- Prometheus: http://10.21.101.6:9090
- Loki: http://10.21.101.6:3100 *(LAN-only — external ACL blocks this)*
- Alertmanager: http://10.21.101.6:9093 *(LAN-only)*

## The "live notebook" philosophy

We treat every non-trivial decision, workaround, or discovered gotcha as
content worth preserving in the repo, not just in commit messages:

- [`RUNBOOK.md → "Gotchas and lessons learned"`](RUNBOOK.md) — operational
  quirks (Prometheus reload flake, FB OTel TLS trap, storage layout
  requirement, etc.)
- [`docs/HOWTO-flashblade-monitoring.md → "Lessons learned during
   dashboard build"`](docs/HOWTO-flashblade-monitoring.md) — Grafana/PromQL
  interaction patterns, variable binding pitfalls, schema-difference
  handling
- [`docs/GAPS.md`](docs/GAPS.md) — tracker of metrics that exist on one
  telemetry path but not the other, for eventual hand-off to Pure
  engineering

When you hit something non-obvious, **add to one of those docs** before
moving on. Your future self will thank you.

## Contributing / extending

This repo is a personal lab sandbox, not a team deliverable. That said:

- **New dashboards** go into `docker-compose/grafana/dashboards/` (live
  provisioned) and, if intended for sharing, also generated into
  `dashboards/` via the script in HOWTO Part 7.
- **New metrics gaps** discovered go into `docs/GAPS.md`.
- **New operational gotchas** go into `RUNBOOK.md`.
- **Secrets stay in the encrypted vault** (`ansible/inventory/group_vars/
  all/vault.yml`) — never in plain files. `.gitignore` covers the usual
  suspects but double-check before committing.

## License & intent

Unlicensed. This is experimental / scratch work — take what's useful, but
don't expect stability guarantees. Dashboard JSON and Ansible playbooks
are easy to adapt to your own lab.
