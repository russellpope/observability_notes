# Pure FlashArray — Per-Client (Host) Dashboard

**Date:** 2026-06-19
**Status:** Approved (design)
**Author:** Russell Pope (with Claude Code)

## Goal

A Grafana dashboard showing **per-client (per-host) statistics** for the Pure
FlashArrays in Prometheus, so we can see exactly what host-level visibility
exists today — and make the NVMe-attribution gap explicit to drive conversations
with Pure engineering. Companion to the front-end NVMe/TCP dashboard
(`pure-fa-nvme`), which is target-side; this one is client-side.

## Context / Discovery

Verified live against lab Prometheus (`10.21.101.6:9090`, host `rp-util1`):

- **811 distinct hosts** across the fleet — too many to list, so the design uses
  a searchable `$host` picker **and** fleet top-N ranking panels.
- The **`host` label is consistent** across all per-host metrics (one host showed
  3 perf + 378 connection + 7 space + 1 connectivity series), so a single `$host`
  variable filters every panel.
- Per-host performance metrics are **gauges** (instantaneous IOPS/BW/latency) —
  used directly, no `rate()`.
- `purefa_host_space_bytes` `space` values: `unique`, `snapshots`, `virtual`,
  `total_physical`, `total_provisioned`, `total_reduction`, `thin_provisioning`.
- `purefa_host_space_data_reduction_ratio` has **extreme outliers** (observed
  `67,587,584` on a near-empty host) → DRR belongs in per-host detail, never in a
  top-N ranking.
- IOPS dimensions: `reads_per_sec`, `writes_per_sec`, `mirrored_writes_per_sec`.

### The NVMe attribution gap (central to this dashboard)

No per-host metric carries a `protocol`, `nqn`, or `transport` label. The only
labels on host metrics are `host`, `instance`, `dimension`, `space`, `volume`,
`hostgroup`, `status`, `details` (and `details` is port-redundancy, not protocol).
So per-host IOPS/BW/latency is **aggregate across all transports** (FC, iSCSI,
NVMe/TCP, NVMe/FC) and **cannot be isolated to NVMe**. NVMe-specific visibility
exists only target-side (interface `services="nvme-tcp"`, the `pure-fa-nvme`
dashboard).

## Non-Goals

- Per-client per-protocol/NVMe traffic split (not exposed by the data source).
- Per-client NVMe session counts (no such metric).
- Joining host→NQN from the REST API (possible future work, not this dashboard).
- Volume-level performance (this is host-scoped).

## Design

**File:** `pure-fa-host-client.json` — provisioned copy in
`docker-compose/grafana/dashboards/`, plus a portable `${DS_PROMETHEUS}` export in
`dashboards/` (mirrors the `pure-fa-nvme` delivery pattern).
**UID:** `pure-fa-host-client`. **Tags:** `pure`, `storage`, `flasharray`, `fa`,
`host`, `client`. **Default time:** `now-3h`. **Panel datasource:** `uid: $datasource`.

**Templating:** `datasource`; `env` (`query_result(purefa_info)`); `instance`
(multi, `query_result(purefa_info{instance=~".+"})`); **`host`** (multi + All,
`label_values(purefa_host_performance_throughput_iops, host)`); `TopN` (custom:
`5,10,15,25`).

### R0 — NVMe attribution gap (text panel, top)

Markdown panel stating: per-host stats are aggregate across all transports
(FC/iSCSI/NVMe-TCP/NVMe-FC); no `protocol`/`nqn`/`transport` label exists, so NVMe
traffic cannot be isolated per client. Two lists — **what we see** (IOPS, BW,
latency, IO size, space, DRR, connections, connectivity per host) vs **what we
don't** (per-client protocol split, NVMe session counts). Points to `pure-fa-nvme`
for target-side NVMe/TCP visibility.

### R1 — Fleet: busiest clients (top-N)

| Panel | Type | Query core |
|---|---|---|
| Top-$TopN hosts by total IOPS | timeseries | `topk($TopN, sum by(instance,host)(purefa_host_performance_throughput_iops{dimension=~"reads_per_sec\|writes_per_sec",instance=~"$instance",env=~"$env"}))` |
| Top-$TopN hosts by bandwidth | timeseries (binBps) | `topk($TopN, sum by(instance,host)(purefa_host_performance_bandwidth_bytes{dimension=~"read_bytes_per_sec\|write_bytes_per_sec",instance=~"$instance",env=~"$env"}))` |
| Top-$TopN hosts by latency | timeseries (µs) | `topk($TopN, max by(instance,host)(purefa_host_performance_latency_usec{dimension=~"usec_per_read_op\|usec_per_write_op",instance=~"$instance",env=~"$env"}))` |
| Top-$TopN hosts by connection count | table/bargauge | `topk($TopN, count by(instance,host)(purefa_host_connections_info{instance=~"$instance",env=~"$env"}))` |

### R2 — Selected client detail (`$host`)

| Panel | Type | Query core |
|---|---|---|
| IOPS by op type | timeseries (iops) | `purefa_host_performance_throughput_iops{host=~"$host",instance=~"$instance",env=~"$env"}` (legend `{{host}} {{dimension}}`) |
| Bandwidth by op type | timeseries (binBps) | `purefa_host_performance_bandwidth_bytes{host=~"$host",instance=~"$instance",env=~"$env"}` |
| Latency by op type | timeseries (µs) | `purefa_host_performance_latency_usec{host=~"$host",instance=~"$instance",env=~"$env"}` |
| Avg IO size | timeseries (bytes) | `purefa_host_performance_average_bytes{host=~"$host",instance=~"$instance",env=~"$env"}` |
| Space breakdown | piechart (bytes) | `purefa_host_space_bytes{host=~"$host",instance=~"$instance",env=~"$env",space=~"unique\|snapshots\|virtual\|total_physical"}` (legend `{{space}}`) |
| Data reduction ratio | stat | `purefa_host_space_data_reduction_ratio{host=~"$host",instance=~"$instance",env=~"$env"}` — panel description warns about near-empty-host outliers |
| Connectivity status | table | `purefa_host_connectivity_info{host=~"$host",instance=~"$instance",env=~"$env"}` → instance, host, status, details |
| Host → volume connections | table | `purefa_host_connections_info{host=~"$host",instance=~"$instance",env=~"$env"}` → instance, volume, hostgroup |

### R3 — Connectivity overview (fleet, all transports)

| Panel | Type | Query core |
|---|---|---|
| Host connectivity by status | piechart | `count by(status)(purefa_host_connectivity_info{instance=~"$instance",env=~"$env"})` |

## Testing / Acceptance

- Dashboard JSON valid (`python3 -m json.tool`); provisioned + portable copies
  identical except datasource binding (`$datasource` vs `${DS_PROMETHEUS}` + `__inputs`).
- All queries return data live (validated during the plan's Task 1).
- `$host` picker drives every R2 panel; selecting a known busy host (e.g. an Oracle
  host on `sn1-x90r2-f06-27`) populates all R2 panels; the host→volume table is
  empty (not erroring) for hosts with no current connections.
- Top-N panels rank hosts and respect `$instance`/`$env`/`$TopN`.
- R0 gap text is present and accurate.
- Provisioned copy imports cleanly into Grafana 13 (no datasource errors).

## Open Questions

None blocking. Host→NQN enrichment from the REST API is a possible future
follow-up if per-client NVMe attribution becomes a hard requirement.
