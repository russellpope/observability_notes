# Pure FlashArray — NVMe/TCP Front-End Dashboard

**Date:** 2026-06-19
**Status:** Approved (design)
**Author:** Russell Pope (with Claude Code)

## Goal

A Grafana dashboard focused on **front-end NVMe-over-TCP** for the Pure FlashArrays
in Prometheus: which ports serve NVMe/TCP, their link state, RX/TX throughput,
packet rate, errors, plus the closest available connection/connectivity signals.
Back-end drive/component health is explicitly **out of scope**.

## Context / Discovery

Live discovery against the lab Prometheus (`10.21.101.6:9090`, host `rp-util1`)
established ground truth — every series below was verified to exist:

- FlashArrays are scraped **directly via Purity's native `/metrics/` endpoint**
  (`https://<array>.fsa.lab/metrics/<category>?namespace=purefa`), not through the
  `pure-fa-exporter` container the repo's `prometheus.yml` still describes. Same
  `purefa_*` schema, so the dashboard is portable. (Config drift noted separately;
  not addressed by this work.)
- ~7 arrays scraped; **NVMe/TCP is live on 2**: `sn1-c60-e12-16` (2 ports) and
  `sn1-x90r2-f07-27` (4 ports), all `enabled="true"`.

### Key metric facts

- **NVMe/TCP is cleanly identifiable at the interface level** via
  `purefa_network_interface_speed_bandwidth_bytes`, which carries a `services`
  label with explicit values incl. `nvme-tcp` (also `nvme-fc`, `iscsi`, `scsi-fc`,
  `file`, `replication`, …), plus `enabled`, `type` (`eth`/`fc`), `ethsubtype`.
  Filter: `services=~".*nvme-tcp.*"` (services can be a comma list).
- `purefa_network_port_info{nqn!="",portal!=""}` identifies NVMe/TCP **target
  portals** (NQN present + IP portal present → TCP listener; FC ports have no portal).
- **Per-port performance is NOT service-tagged.** `purefa_network_interface_performance_*`
  (`bandwidth_bytes`, `throughput_pkts`, `errors`) carry only `name`/`type`/`dimension`.
  To restrict to NVMe/TCP, filter by a label-match join on `(instance,name)` against
  the services-labeled speed metric, using `and on(instance,name)` so LHS values are
  **filtered, not scaled** (using `*` would multiply throughput by link speed — wrong).
- **No NVMe session-count metric exists** in Purity's native exporter. Closest
  protocol-agnostic substitutes:
  - `purefa_host_connectivity_info{status,details,host}` — per-host connectivity
    status (fleet-wide at discovery: 755 critical / 395 healthy / 31 unhealthy / 1 unused).
  - `purefa_host_connections_info{host,volume,hostgroup}` — host↔volume connection
    mappings (value 1 each; `count` = connection count).
  Both span all transports and **cannot** be isolated to NVMe.

## Non-Goals

- Back-end NVMe drive / DirectFlash / component health (separate concern).
- True NVMe-oF session/controller-state counts (not exposed by the data source).
- Per-protocol IOPS/latency split (Purity does not break array/host perf by transport).
- Fixing the `prometheus.yml` exporter-vs-native config drift.

## Design

**File:** `pure-fa-nvme.json`, written to both `dashboards/` and
`docker-compose/grafana/dashboards/` (mirrors existing dashboard convention).
**UID:** `pure-fa-nvme`. **Tags:** `pure`, `storage`, `flasharray`, `nvme`, `nvme-tcp`.
**Default time:** `now-3h`. **Panel datasource:** `uid: $datasource`.

**Templating** (reused from `grafana-purefa-flasharray-overview.json`):
- `datasource` — Prometheus datasource picker.
- `env` — `query_result(purefa_info)`.
- `instance` — `query_result(purefa_info{instance=~".+"})`, multi.

A top-of-dashboard **text panel** names the NVMe/TCP-capable arrays so empty panels
on non-NVMe arrays read as expected, not as a bug.

### R1 — NVMe/TCP Front-End (core; NVMe/TCP-filtered)

| Panel | Type | Query core |
|---|---|---|
| Enabled NVMe/TCP ports | stat | `count by(instance)(purefa_network_interface_speed_bandwidth_bytes{services=~".*nvme-tcp.*",enabled="true",instance=~"$instance",env=~"$env"})` |
| NVMe/TCP target portals | table | `purefa_network_port_info{nqn!="",portal!="",instance=~"$instance",env=~"$env"}` → instance, name, nqn, portal |
| RX/TX throughput per NVMe/TCP port | timeseries (binBps) | `purefa_network_interface_performance_bandwidth_bytes{dimension=~"received_bytes_per_sec|transmitted_bytes_per_sec"} and on(instance,name) purefa_network_interface_speed_bandwidth_bytes{services=~".*nvme-tcp.*"}` |
| Packets/sec per NVMe/TCP port | timeseries | `purefa_network_interface_performance_throughput_pkts{...} and on(instance,name) (...nvme-tcp...)` |
| Errors/discards per NVMe/TCP port | timeseries | `purefa_network_interface_performance_errors{...} and on(instance,name) (...nvme-tcp...)` |
| NVMe/TCP port speed & state | table | `purefa_network_interface_speed_bandwidth_bytes{services=~".*nvme-tcp.*",instance=~"$instance",env=~"$env"}` → name, speed, enabled, ethsubtype |

All R1 timeseries are filtered to NVMe/TCP ports via `and on(instance,name)` and
templated by `$instance`/`$env`.

### R2 — Connection / Connectivity State (protocol-agnostic)

Each panel description states explicitly that it spans **all transports**, not just NVMe.

| Panel | Type | Query core |
|---|---|---|
| Host connectivity by status | piechart/stat | `count by(status)(purefa_host_connectivity_info{instance=~"$instance",env=~"$env"})` |
| Critical/unhealthy hosts | table | `purefa_host_connectivity_info{status=~"critical|unhealthy",instance=~"$instance",env=~"$env"}` → host, status, details |
| Host↔volume connections per array | stat/timeseries | `count by(instance)(purefa_host_connections_info{instance=~"$instance",env=~"$env"})` |

### R3 — Array Front-End Performance (context)

Array-wide; cannot be isolated to NVMe. Mirrors the overview's panel style.

| Panel | Query core |
|---|---|
| Latency by op type | `purefa_array_performance_latency_usec{instance=~"$instance",env=~"$env",dimension=~"..."}` (µs) |
| IOPS by type | `purefa_array_performance_throughput_iops{...,dimension}` |
| Bandwidth by type | `purefa_array_performance_bandwidth_bytes{...,dimension}` (binBps) |
| Queue depth | `purefa_array_performance_queue_depth_ops{...}` |

## Testing / Acceptance

- Dashboard JSON imports cleanly into Grafana (valid schema, no datasource errors).
- On `sn1-c60-e12-16` and `sn1-x90r2-f07-27`, R1 panels render NVMe/TCP ports with
  non-empty throughput/speed/portal data.
- R1 panels are empty (not erroring) on non-NVMe/TCP arrays.
- `and on(instance,name)` throughput values match raw per-port throughput magnitude
  (sanity: not scaled by link speed).
- R2 connectivity status counts roughly match the discovery snapshot.
- Validate JSON with `python3 -m json.tool` and confirm both file copies are identical.

## Open Questions

None blocking. Config drift (`prometheus.yml` exporter vs native scrape) noted for
possible future cleanup, out of scope here.
