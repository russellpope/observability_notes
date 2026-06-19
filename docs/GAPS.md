# FlashArray / FlashBlade Telemetry Path Gaps

Tracks capabilities available on one telemetry path but not the other. Used
to drive follow-up conversations with Pure engineering about adding coverage
on the newer native OTel exporter.

**Legend:**
- ✅ covered by both paths
- 🟦 OpenMetrics-only (`purefb_*` / `purefa_*`)
- 🟧 OTel-only (`flashblade_storage_device_*`)

Discovered during the `rp-observability` lab build (Apr 2026) against:
- OpenMetrics proxy `pure-fb-om-exporter:latest` on Purity//FB 5.x
- Native OTel push on Purity//FB with the SLC FBS200 test array

---

## FlashBlade — confirmed gaps

### 🟦 Per-user / per-group filesystem usage

OM exports these; OTel has nothing equivalent (query against
`{__name__=~"flashblade.*user.*|.*group.*|.*usage.*"}` returns empty):

| OM metric | Purpose |
|---|---|
| `purefb_file_system_usage_users_bytes` | Space consumed per user per filesystem |
| `purefb_file_system_usage_groups_bytes` | Space consumed per POSIX group per filesystem |

**Impact:** Chargeback / quota reporting for file workloads requires the OM
path. Ask Pure engineering: can the OTel exporter emit
`flashblade_storage_device_fs_usage_users_bytes` / `_groups_bytes` with
`user_id` / `group_id` / `user_name` / `group_name` resource attributes?

### 🟦 NFS export policy info

| OM metric | Purpose |
|---|---|
| `purefb_nfs_export_rule` | Labels carry `host`, `rule_policy`, `permission` |

**Impact:** Security auditing, policy visibility.

### 🟦 Hardware connector *granular* performance

OM splits per-connector throughput, packets, errors:

| OM metric | OTel equivalent |
|---|---|
| `purefb_hardware_connectors_performance_bandwidth_bytes` | Partial — `flashblade_storage_device_connector_bw_bps` |
| `purefb_hardware_connectors_performance_throughput_pkts` | Partial — `flashblade_storage_device_connector_bw_pps` |
| `purefb_hardware_connectors_performance_errors` | Partial — `flashblade_storage_device_connector_err_ps` |

**Impact:** Close enough — the OTel metrics carry equivalent data but different
naming conventions and units (pps vs pkts). Label structure may differ; verify
per-connector identification (connector name, chassis, FM, port) is preserved.

### 🟧 OTel-only useful detail

These appear in the OTel exporter but not the OpenMetrics one:

| OTel metric | Notes |
|---|---|
| `flashblade_storage_device_info` | Rich device attributes (fleet_name, realm_name, entity_owner chains) |
| `flashblade_storage_device_repl_bw_bps` | Replication bandwidth as a first-class metric |
| `flashblade_storage_device_bucket_bw_bpo` | Bytes-per-op at bucket granularity |
| `op` label granularity | OTel splits operations far more finely (metadata ops: readdir, setattr, lookup, access, etc.), while OM collapses to `dimension=read/write` |

---

### 🟧 `-1` sentinel on OTel space metrics

Pure's native OTel exporter reports `-1` as a value on certain
`flashblade_storage_device_space_bytes` series when the metric doesn't
apply to that particular array/configuration. Observed on:

| space_type | Where we saw `-1` |
|---|---|
| `total_provisioned` | SLC FB (has no configured provisioning limits) |
| `available_provisioned` | Same; buckets and array-level |

Dashboards that display bytes without filtering treat `-1` as a literal
byte count, producing the eye-catching "-1 B" display. **Workaround:**
either filter `{value != -1}` at the panel level, or pick a space_type
that's always populated (e.g. `virtual`, `total_physical`, `unique`)
depending on what you actually want to show.

**Ask for engineering:** either omit the series when the value is
inapplicable, or switch to a clearer sentinel (NaN, staleness marker,
or a `state` label) that standard dashboards can filter cleanly.

### 🟦 Data Reduction Ratio at array / per-object breakdown

| OM metric | OTel equivalent |
|---|---|
| `purefb_array_space_data_reduction_ratio` (array-wide, single series) | `flashblade_storage_device_space_reduction` **with `exported_storage_type=all`** |

OTel exposes DRR decomposed by `exported_storage_type` (`file`/`object`/`all`)
which is richer, but code that assumes a single "array DRR" number needs to
explicitly filter to `exported_storage_type="all"`.

### 🟦 Array-level IOPS read/write ratio breakdown

OM uses `dimension=read_per_sec|write_per_sec|others_per_sec` on
`purefb_array_performance_throughput_iops` — three mutually-exclusive
dimensions summing to the total. OTel's `op` label enumerates operations
(read/write + metadata ops) without an equivalent "others" bucket for
non-read-non-write activity. For faithful read-ratio panels, either filter
OTel to `op=~"read|write"` and accept it ignores metadata traffic, or
compute "others" as `total - read - write`.

## Cross-cutting schema differences

Not strictly gaps, but worth noting for anyone porting queries between paths:

### Metric name renames to watch

| OpenMetrics (old) | OpenMetrics (current) |
|---|---|
| `purefb_filesystems_*` | `purefb_file_systems_*` |
| `purefb_array_performance_iops` | `purefb_array_performance_throughput_iops` |
| `purefb_hw_status` | `purefb_hardware_health` |
| `purefb_hardware_controller_health` | `purefb_hardware_health` |

Pure's canned Grafana dashboards shipped with the older names — hence the
"No data" symptom on fresh installs. Our dashboards have been patched.

### Label shape differences

| Concept | OpenMetrics | OTel |
|---|---|---|
| Array identity | `instance` (FQDN) | `device_name` (short name, e.g. `slc6-fbs200-n3-b35-12`) |
| Filesystem identity | `name` | `entity_name` |
| Bucket identity | `name` + `account` | `entity_name` + `entity_owner` |
| I/O direction | `dimension=read_bytes_per_sec\|write_bytes_per_sec` | `op=read\|write` (plus many metadata ops) |
| Protocol | `protocol=NFS\|SMB\|S3\|HTTP\|all` (uppercase) | `proto=nfs\|smb\|s3\|http\|all` (lowercase) |
| Capacity dimension | `space=capacity\|unique\|shared\|snapshots\|virtual\|total_physical` | `space_type=capacity\|unique\|shared\|snapshots\|virtual\|total_physical` (aligned) + `exported_storage_type=all\|file\|object` |

---

## OTel-specific data-freshness quirk

Some OTel metrics push at a cadence longer than Prometheus' default 5-minute
staleness window, which makes instant queries return empty even though data
exists. Affected metrics (observed):

- `flashblade_storage_device_health`
- `flashblade_storage_device_info`
- `flashblade_storage_device_bucket_*` (all bucket-level metrics)
- `flashblade_storage_device_obj_store_account_*`

**Workaround**: wrap queries in `last_over_time(<metric>[10m])`. Our dashboard
JSONs do this consistently — see `HOWTO-flashblade-monitoring.md` Part 5.6 for
detail.

**Ask for engineering:** Either increase push cadence for slow-moving metrics
to align with a typical 5-min staleness, or emit them more often even without
value change.

---

## FlashArray — confirmed gaps

We do not yet have FlashArray OTel push enabled, so there's no second-path
comparison to draw here. Once Purity//FA adds native OTel export, repeat the
gap analysis.

Current `purefa_*` coverage (OM path) includes:
- array / performance / space / DRR / utilization
- volumes, pods, hosts, host connections, directories
- hardware (drives, controllers, temperature, voltage, network interfaces)
- alerts, info

> **Deployment note (repo config drift, not an upstream gap):** the committed
> `docker-compose/prometheus/prometheus.yml` ships a templated `flasharray`
> job pointed at a `pure-fa-exporter` container with placeholder targets. The
> **live lab** instead scrapes ~7 real arrays *directly* via Purity's native
> OpenMetrics endpoint (`https://<array>.fsa.lab/metrics/<array|directories|hosts|pods|volumes>?namespace=purefa`)
> — no exporter container. The `purefa_*` schema is identical either way, so
> dashboards are portable across both. Reconcile `prometheus.yml` to the
> native-scrape layout if/when the repo config is meant to mirror the live lab.
>
> NVMe/TCP specifics surfaced during the `pure-fa-nvme` dashboard build:
> `services="nvme-tcp"` on `purefa_network_interface_speed_bandwidth_bytes`
> cleanly identifies NVMe/TCP interfaces; per-port perf metrics aren't
> service-tagged (join via `and on(instance,name)`). **No NVMe session/connection
> metric exists** in the native exporter — `purefa_host_connectivity_info` /
> `purefa_host_connections_info` are the closest, both protocol-agnostic.

---

## Contributing to this doc

When you discover a metric on one path that doesn't exist on the other, add
it to the relevant section above with:
1. The metric name on the path where it exists
2. A short description of its purpose
3. Impact or use case that's blocked

Keep the tone factual — this doc goes upstream to Pure engineering.
