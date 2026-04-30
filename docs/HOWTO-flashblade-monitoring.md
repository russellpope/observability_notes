# HOWTO: FlashBlade monitoring — OpenMetrics & OTel

End-to-end guide for getting Pure Storage FlashBlade telemetry into Prometheus
+ Grafana. Covers both ingestion paths available today:

1. **OpenMetrics** — `pure-fb-om-exporter` stateless proxy that converts the
   FB REST API into Prometheus-format scrape responses. Works on all Purity//FB
   versions that expose the REST API (2.x and later). Metric prefix: `purefb_*`.
2. **Native OTel push** — FB itself streams OTLP/gRPC to a collector.
   Requires modern Purity//FB firmware with the native OTel exporter. Metric
   prefix: `flashblade_*`.

They are independent; run both if you want to compare. This guide assumes the
baseline stack from `RUNBOOK.md` is already up: Prometheus, Grafana, OTel
collector, and the `pure-fb-exporter` container on the observability host.

## Related artifacts in this repo

| File | What it is |
|---|---|
| [`../RUNBOOK.md`](../RUNBOOK.md) | Stack deployment, day-2 operations, troubleshooting |
| [`GAPS.md`](GAPS.md) | Metrics available on one telemetry path but not the other — feedback track for Pure engineering |
| [`../dashboards/`](../dashboards/) | Portable JSON exports of every unified/custom dashboard (see Part 7 below) |
| [`../docker-compose/grafana/dashboards/`](../docker-compose/grafana/dashboards/) | Live provisioned dashboard copies (file-provisioning source of truth) |
| [`../ansible/playbooks/templates/prometheus.yml.j2`](../ansible/playbooks/templates/prometheus.yml.j2) | Prometheus scrape config template — adds `env="lab"` static label that Pure's canned dashboards filter on |
| [`../ansible/playbooks/templates/pure-fa-exporter.yml.j2`](../ansible/playbooks/templates/pure-fa-exporter.yml.j2) | FlashArray token list template |

---

## Reference links

| Resource | URL |
|---|---|
| FB OpenMetrics exporter (source + dashboards) | https://github.com/PureStorage-OpenConnect/pure-fb-openmetrics-exporter |
| FB OpenMetrics exporter (container image) | `quay.io/purestorage/pure-fb-om-exporter` |
| FA OpenMetrics exporter (dashboards referenced here) | https://github.com/PureStorage-OpenConnect/pure-fa-openmetrics-exporter |
| Prometheus scrape config reference | https://prometheus.io/docs/prometheus/latest/configuration/configuration/#scrape_config |
| OpenTelemetry Collector OTLP receiver | https://github.com/open-telemetry/opentelemetry-collector/tree/main/receiver/otlpreceiver |
| OTel Collector Contrib Prometheus exporter | https://github.com/open-telemetry/opentelemetry-collector-contrib/tree/main/exporter/prometheusexporter |
| Grafana file-based dashboard provisioning | https://grafana.com/docs/grafana/latest/administration/provisioning/#dashboards |
| Pure support portal (Purity release notes, OTel docs) | https://support.purestorage.com/ |

---

## System requirements

| Component | Requirement |
|---|---|
| Observability host | Docker + Compose plugin; TCP egress to each FB on 443; TCP ingress on 4317 (OTel), 9090 (Prom UI), 3000 (Grafana UI) |
| Network | Observability host must reach FB management IP(s) on 443 (OpenMetrics) and each FB must reach observability host on 4317/gRPC (OTel push) |
| FlashBlade (both paths) | An API token bound to an account with at least **readonly** role on the FB |
| FlashBlade (OpenMetrics) | Purity//FB 2.x or newer (any version exposing the REST API that the exporter targets) |
| FlashBlade (OTel) | Purity//FB version with the native OTel exporter (available via GUI → Settings → External Telemetry). Verify in your release notes. |

Resource sizing for the observability host scales with cardinality. A single
FB at 30 s scrape interval generates ~6–7k active series in Prometheus
(OpenMetrics path, all namespaces enabled) and <100 MB/day of TSDB growth.

---

## Part 1 — OpenMetrics exporter setup

### 1.1  Generate an API token on the FlashBlade

Either path works. Use a **readonly** user; don't use `pureuser` unless you
have to.

**GUI:**
1. FB GUI → **Settings** → **Users and Roles** → pick a user with `readonly`,
   or create one.
2. Click the user → **API Token** → **Create**.
3. Copy the token (format: `T-<uuid>`). It is shown only once.

**CLI (SSH to the FB as an admin):**
```
pureadmin create --api-token --user <readonly-user>
```
Copy the `T-…` token from the output.

### 1.2  Run the exporter container

The exporter is **stateless** — it holds no credentials. Prometheus passes the
FB endpoint and token on every scrape. This is what makes it safe to run a
single exporter for a whole fleet.

If you're using this repo's Docker Compose stack, the exporter is already
defined in `docker-compose/docker-compose.yml`:

```yaml
pure-fb-exporter:
  image: quay.io/purestorage/pure-fb-om-exporter:latest
  container_name: pure-fb-exporter
  restart: unless-stopped
  networks:
    - observability
```

Deploy / reconcile:
```
cd ansible
ansible-playbook playbooks/deploy-stack.yml
```

Standalone (no Compose):
```
docker run -d --name pure-fb-exporter \
  -p 9491:9491 \
  quay.io/purestorage/pure-fb-om-exporter:latest
```

### 1.3  Add the FB to Prometheus scrape config

Put the FB in `vault_pure_fb_tokens` so the playbook renders scrape jobs. Each
FB produces six scrape jobs — one per metrics namespace (`array`, `clients`,
`filesystems`, `objectstore`, `policies`, `usage`).

```
cd ansible
ansible-vault edit inventory/group_vars/all/vault.yml
# add under vault_pure_fb_tokens:
#   - address: <fb-fqdn>
#     api_token: "T-<uuid>"
ansible-playbook playbooks/deploy-stack.yml
```

Each scrape job looks like this in the rendered `prometheus.yml` (from
`ansible/playbooks/templates/prometheus.yml.j2`):

```yaml
- job_name: "flashblade_<fqdn>_<namespace>"
  metrics_path: /metrics/<namespace>
  authorization:
    credentials: "T-<uuid>"
  params:
    endpoint: ['<fb-fqdn>']
  static_configs:
    - targets: ['pure-fb-exporter:9491']
      labels:
        instance: "<fb-fqdn>"
        storage_type: flashblade
        namespace: "<namespace>"
        source: openmetrics
```

**Available metric paths** (for reference when customizing):

| Path | Covers |
|---|---|
| `/metrics` | All namespaces at once (heavy; prefer per-namespace jobs) |
| `/metrics/array` | Array-level performance + space |
| `/metrics/clients` | NFS/SMB client stats |
| `/metrics/filesystems` | Per-filesystem performance + space |
| `/metrics/objectstore` | Buckets + S3 accounts |
| `/metrics/policies` | NFS export/snapshot policies |
| `/metrics/usage` | Quota usage per user/group/account |

### 1.4  Validate the OpenMetrics path

From the observability host:

```bash
# Hit the exporter directly with a valid endpoint + token.
# Should return OpenMetrics text starting with HELP/TYPE lines.
docker exec pure-fb-exporter wget -qO- \
  --header "Authorization: Bearer T-<uuid>" \
  "http://localhost:9491/metrics/array?endpoint=<fb-fqdn>" | head -20

# Prometheus targets: all six jobs should be 'up'.
curl -s 'http://localhost:9090/api/v1/targets?state=active' \
  | jq '.data.activeTargets[] | select(.scrapePool|test("flashblade_"))
        | {pool: .scrapePool, health, err: .lastError}'

# Metric count sanity check.
curl -sG 'http://localhost:9090/api/v1/query' \
  --data-urlencode 'query=count({__name__=~"purefb_.+"})' | jq .

# Representative query — array-level write latency.
curl -sG 'http://localhost:9090/api/v1/query' \
  --data-urlencode 'query=purefb_array_performance_latency_usec{dimension="write"}' | jq .
```

Expected orders of magnitude for a lightly loaded FB:
- 6 scrape jobs `up`
- ~5–10 k `purefb_*` series
- ~40–60 distinct metric families

If jobs show `down` with `401 Unauthorized`: token is wrong or doesn't have
readonly access. If `400 Bad Request`: `params.endpoint` is missing or
malformed.

---

## Part 2 — Native OTel push setup

### 2.1  Verify Purity//FB has native OTel export

In the FB GUI: **Settings** → **System** → **External Telemetry** (or similar
— menu label varies by Purity version). If the menu doesn't exist, your Purity
is too old; use OpenMetrics instead.

### 2.2  Configure the export target on the FB

1. FB GUI → External Telemetry → **Add** a new export target.
2. **Endpoint:** `<observability-host-ip>:4317`
3. **Protocol:** OTLP / gRPC.
4. **TLS:** **Off** for a lab. Pure's current UI enables TLS by default and
   there is no "insecure/skip-verify" mode — if TLS is on, the FB requires a
   trusted cert uploaded. See *Gotcha* below.
5. **Test network connectivity** — validates TCP reachability to 4317 (`nc`
   under the hood). Should return a success banner.
6. **Test delivery** — sends a single `flashblade_test_connectivity_check_ratio`
   metric. Should return a success banner.
7. Save. Then enable the telemetry types you want (metrics at a minimum;
   optionally health and events).

**Gotcha — the TLS trap:** The "Test network connectivity" button uses raw nc
and passes fine even when TLS is misconfigured. The "Test delivery" button
sends OTLP/gRPC and will fail silently if the FB tries a TLS handshake against
our plaintext listener. If delivery fails with no log entry on our end, the
OTel collector will show **zero** new connection attempts — that's your
signature for a TLS mismatch. Disable TLS on the FB target for lab use, or
properly terminate TLS at the collector.

### 2.3  OTel collector receiver config

Already wired in `docker-compose/otel-collector/otel-collector-config.yml`:

```yaml
receivers:
  otlp:
    protocols:
      grpc:
        endpoint: 0.0.0.0:4317
      http:
        endpoint: 0.0.0.0:4318

processors:
  batch:
    timeout: 10s
  resource:
    attributes:
      - key: collector.name
        value: "lab-otel-collector"
        action: upsert

exporters:
  prometheus:
    endpoint: 0.0.0.0:8889
    namespace: flashblade             # becomes the metric prefix
    resource_to_telemetry_conversion:
      enabled: true                   # copies OTel resource attrs to labels
  debug:
    verbosity: basic

service:
  pipelines:
    metrics:
      receivers: [otlp]
      processors: [batch, resource]
      exporters: [prometheus, debug]
```

Key settings:
- `namespace: flashblade` — every OTel metric becomes `flashblade_<name>` in
  Prometheus. This is what gives you `flashblade_storage_device_iops` etc.
- `resource_to_telemetry_conversion: enabled: true` — copies OTel resource
  attributes (`device_name`, `fleet_id`, `component_name`, …) onto each
  metric as Prometheus labels. Without this you lose the ability to slice by
  FB.

### 2.4  Prometheus scrape of the collector's Prom exposition

Already in `prometheus.yml.j2`:
```yaml
- job_name: "otel-collector"
  scrape_interval: 30s
  static_configs:
    - targets: ["otel-collector:8889"]
      labels:
        storage_type: flashblade
        source: otel
```

### 2.5  Validate the OTel path

```bash
# OTel collector should log received metric batches (rate grows as the FB
# starts streaming real telemetry).
docker logs otel-collector --tail 20 | grep -i Metrics

# Prometheus exposition direct from the collector.
curl -s http://localhost:8889/metrics | grep '^flashblade_' | head

# Prometheus target for the collector.
curl -s 'http://localhost:9090/api/v1/targets?state=active' \
  | jq '.data.activeTargets[] | select(.scrapePool=="otel-collector")
        | {pool: .scrapePool, health, err: .lastError}'

# Which FBs are reporting via OTel?
curl -sG 'http://localhost:9090/api/v1/label/device_name/values' | jq .

# Sample query.
curl -sG 'http://localhost:9090/api/v1/query' \
  --data-urlencode 'query=count by (device_name) (flashblade_storage_device_health)' | jq .
```

Expected:
- Collector logs show steady `Metrics` batches, growing from `data points: 1`
  (connectivity test) to hundreds per minute once telemetry enables.
- ~20–30 distinct `flashblade_*` metric families.
- `device_name` label exposes each pushing FB.

---

## Part 3 — Import the canned Pure Grafana dashboards

Pure publishes Grafana dashboards alongside each OpenMetrics exporter repo.
They use `purefb_*` / `purefa_*` metric names and bind to a datasource at
import time.

### 3.1  Download the dashboards

The dashboards in this repo live in `docker-compose/grafana/dashboards/` and
are already patched for provisioning. If you're syncing with upstream, re-run
the fetch:

```bash
cd docker-compose/grafana/dashboards

# FlashArray (one dashboard)
curl -sLO https://raw.githubusercontent.com/PureStorage-OpenConnect/pure-fa-openmetrics-exporter/master/extra/grafana/grafana-purefa-flasharray-overview.json

# FlashBlade (seven dashboards)
BASE=https://raw.githubusercontent.com/PureStorage-OpenConnect/pure-fb-openmetrics-exporter/main/extra/grafana
for f in grafana-purefb-flashblade-overview.json \
         pure-fb-filesystem-detail.json \
         pure-fb-hardware-health.json \
         pure-fb-landscape.json \
         pure-fb-storage-usage-details.json \
         pure-fb-storage-usage.json \
         pure-fb-system-detail.json; do
  curl -sLO "$BASE/$f"
done
```

### 3.2  Patch them for file-based provisioning

Upstream dashboards ship with a `__inputs` block that expects you to bind a
datasource when importing through the GUI. File-based provisioning can't
satisfy `__inputs`, so the dashboards load but show *"Datasource ${DS_PROMETHEUS}
not found"* on every panel.

Strip `__inputs` / `__requires` and substitute `${DS_PROMETHEUS}` with the
literal datasource name:

```bash
python3 <<'EOF'
import json, glob
for f in glob.glob('docker-compose/grafana/dashboards/*.json'):
    d = json.load(open(f))
    d.pop('__inputs', None)
    d.pop('__requires', None)
    s = json.dumps(d, indent=2).replace('${DS_PROMETHEUS}', 'Prometheus')
    open(f, 'w').write(s)
EOF
```

The datasource name `Prometheus` must match the provisioning config at
`docker-compose/grafana/provisioning/datasources/datasources.yml`.

### 3.3  Wire provisioning

The repo already has:
- `docker-compose/grafana/provisioning/datasources/datasources.yml` — declares
  `Prometheus` (primary), `Loki`, `Alertmanager` datasources.
- `docker-compose/grafana/provisioning/dashboards/dashboards.yml` — points
  Grafana at `/var/lib/grafana/dashboards` for file-based loading.
- `docker-compose/docker-compose.yml` — bind-mounts `./grafana/dashboards`
  into the Grafana container.

If you're not using this repo's Compose stack, add an equivalent provider
config pointing to whichever directory holds your JSON files.

### 3.4  Reload Grafana provisioning

```bash
ansible-playbook playbooks/deploy-stack.yml     # syncs files + restarts compose
# or for an instant reload without a full redeploy:
curl -s -u admin:'<pw>' -X POST \
  http://<obs-host>:3000/api/admin/provisioning/dashboards/reload
```

### 3.5  Verify

```bash
curl -s -u admin:'<pw>' \
  'http://<obs-host>:3000/api/search?type=dash-db' | jq '.[].title'
```

Expected to list all nine dashboards (8 from Pure + 1 custom OTel). Open
`http://<obs-host>:3000` → Dashboards → browse. Panels should populate
immediately for FBs monitored via the OpenMetrics path.

---

## Part 4 — Build a custom OTel FlashBlade dashboard (from scratch)

The canned Pure dashboards use `purefb_*` names and won't light up for FBs
monitored only via OTel push. This section walks through building a
minimal-but-useful overview dashboard for the OTel path. A finished version
lives at `docker-compose/grafana/dashboards/pure-fb-otel-overview.json`.

### 4.1  Understand the OTel schema

Key metric families (prefix `flashblade_storage_device_`):

| Family | Granularity |
|---|---|
| `health` | Per-component (chassis, blade, bay, fabric module, ETH port, fan, PSU) |
| `alerts` | Per-open-alert (each alert is a series with `severity`, `summary`, `action`) |
| `space_bytes` / `space_reduction` | Per-array (by `exported_storage_type` and `space_type`) |
| `iops` / `latency_us` / `bw_bps` / `bw_bpo` | Per-array performance, by `op` (read/write/access/…) |
| `fs_iops` / `fs_latency_us` / `fs_bw_bps` / `fs_space_bytes` | Per-filesystem (`entity_name`) |
| `bucket_iops` / `bucket_latency_us` / `bucket_bw_bps` / `bucket_space_bytes` | Per-bucket (`entity_name`, `entity_owner`) |
| `connector_bw_bps` / `connector_bw_pps` / `connector_err_ps` | Per-ethernet-connector |
| `obj_store_account_*` | Per-object-store-account |
| `info` | Device metadata (version, model) |

Labels worth knowing:
- `device_name` — the FB identity (e.g. `slc6-fbs200-n3-b35-12`)
- `component_name` / `component_type` — physical components for health/hardware
- `entity_name` / `entity_owner` — logical entity (filesystem, bucket, account)
- `op` — operation type (`read`, `write`, `access`, …)
- `space_type` — `total_physical`, `capacity`, `unique`, `virtual`, `snapshots`, …
- `exported_storage_type` — `all`, `file`, `object`
- `severity` (alerts only) — `info`, `warning`, `critical`

Health value convention (observed, not documented): `5` = healthy.

### 4.2  Create the dashboard skeleton

In Grafana UI: **Dashboards** → **New** → **New dashboard** → **Add visualization**.

Alternatively, author the JSON directly — use
`docker-compose/grafana/dashboards/pure-fb-otel-overview.json` as a template
and drop a new file into that directory; Grafana picks it up on
provisioning reload.

Set the dashboard-level variable so every panel filters by FB:

- Name: `device`
- Type: Query
- Datasource: Prometheus
- Query: `label_values(flashblade_storage_device_health, device_name)`
- Include All: on
- Multi-value: on

### 4.3  Panels — top row (stats, 4 × (w=6, h=4))

| Panel | Viz | PromQL | Unit |
|---|---|---|---|
| Unhealthy Components | Stat | `count(flashblade_storage_device_health{device_name=~"$device"} != 5) or vector(0)` | short; value mapping `0 → "All OK"` (green), threshold red at 1 |
| Open Alerts by Severity | Stat | `count by (severity) (flashblade_storage_device_alerts{device_name=~"$device"}) or on() vector(0)` | short; threshold yellow at 1 |
| Space Used | Stat | `sum by (device_name) (flashblade_storage_device_space_bytes{device_name=~"$device", space_type="total_physical", exported_storage_type="all"})` | bytes |
| Data Reduction | Stat | `avg by (device_name) (flashblade_storage_device_space_reduction{device_name=~"$device"})` | none, 2 decimals |

### 4.4  Panels — middle row (time series, 2 × (w=12, h=8))

| Panel | PromQL | Unit |
|---|---|---|
| Array IOPS | `sum by (device_name, op) (flashblade_storage_device_iops{device_name=~"$device"})` | iops |
| Array Latency | `avg by (device_name, op) (flashblade_storage_device_latency_us{device_name=~"$device"})` | µs |

Legend format: `{{device_name}} / {{op}}`.

### 4.5  Panels — bottom row

| Panel | PromQL | Unit |
|---|---|---|
| Array Bandwidth | `sum by (device_name, op) (flashblade_storage_device_bw_bps{device_name=~"$device"})` | Bps |
| Filesystem IOPS (top 10) | `topk(10, sum by (device_name, entity_name, op) (flashblade_storage_device_fs_iops{device_name=~"$device"}))` | iops |

### 4.6  Critical gotcha — datasource reference form

Grafana 10+ uses a UID-based datasource reference by default in JSON. That form
does **not** bind to file-provisioned datasources unless you set the UID
explicitly in the provisioning config. Simplest fix: use the legacy string
form in every panel and template variable:

```json
"datasource": "Prometheus"
```

**Not:**
```json
"datasource": {"type": "prometheus", "uid": "prometheus"}
```

If you author in the Grafana UI then export JSON, search-and-replace the UID
form to the string form before committing to `dashboards/`. Pure's canned
dashboards already use the string form.

### 4.7  Save and reload

Save in the Grafana UI *or* write the JSON file directly. For file-based
workflow:

```bash
# Drop the JSON into docker-compose/grafana/dashboards/
ansible-playbook playbooks/deploy-stack.yml
# Or instant reload:
curl -s -u admin:'<pw>' -X POST \
  http://<obs-host>:3000/api/admin/provisioning/dashboards/reload
```

Verify the dashboard appears and every panel renders. The Unhealthy
Components stat should show "All OK" (green), Open Alerts reflects the actual
count per severity, Space Used is non-zero in bytes, Data Reduction is
around 1–3×.

---

## Part 5 — Build an object-storage-focused dashboard

This section scopes a single dashboard to S3/object workloads, drawing from
**both** ingestion paths so it works regardless of how any given FB reports
in. Save it as `docker-compose/grafana/dashboards/pure-fb-object-storage.json`
(file committed alongside this guide if you use this repo).

### 5.1  Metrics on each path

**OpenMetrics (`purefb_*`):**

| Metric | Scope |
|---|---|
| `purefb_array_s3_performance_throughput_iops` | Array-wide S3 IOPS (by read/write) |
| `purefb_array_s3_performance_latency_usec` | Array-wide S3 latency (by op) |
| `purefb_buckets_performance_throughput_iops` | Per-bucket IOPS |
| `purefb_buckets_performance_latency_usec` | Per-bucket latency |
| `purefb_buckets_performance_bandwidth_bytes` | Per-bucket bandwidth |
| `purefb_buckets_s3_specific_performance_throughput_iops` | Per-bucket S3 IOPS |
| `purefb_buckets_s3_specific_performance_latency_usec` | Per-bucket S3 latency |
| `purefb_buckets_space_bytes` | Per-bucket capacity (by `space=total_physical|unique|snapshots|virtual|destroyed`) |
| `purefb_buckets_object_count` | Objects per bucket |
| `purefb_buckets_space_data_reduction_ratio` | DRR per bucket |
| `purefb_buckets_quota_space_bytes` | Per-bucket quotas |
| `purefb_object_store_accounts_space_bytes` | Per-account total space |
| `purefb_object_store_accounts_object_count` | Per-account object count |
| `purefb_object_store_accounts_data_reduction_ratio` | Per-account DRR |

Key labels: `instance` (FB FQDN), `name` (bucket or account), `account`,
`space` (space dimension).

**OTel (`flashblade_storage_device_bucket_*`):**

| Metric | Scope |
|---|---|
| `flashblade_storage_device_bucket_iops` | Per-bucket IOPS (by `op`) |
| `flashblade_storage_device_bucket_latency_us` | Per-bucket latency |
| `flashblade_storage_device_bucket_bw_bps` | Per-bucket bandwidth (bytes/s) |
| `flashblade_storage_device_bucket_bw_bpo` | Per-bucket bytes-per-op |
| `flashblade_storage_device_bucket_space_bytes` | Per-bucket space (by `space_type`) |
| `flashblade_storage_device_bucket_space_reduction` | Per-bucket DRR |
| `flashblade_storage_device_bucket_obj_count` | Per-bucket object count |
| `flashblade_storage_device_obj_store_account_space_bytes` | Per-account space |
| `flashblade_storage_device_obj_store_account_obj_count` | Per-account object count |
| `flashblade_storage_device_obj_store_account_space_reduction` | Per-account DRR |

Key labels: `device_name` (FB), `entity_name` (bucket/account), `entity_owner`
(account for bucket metrics), `op`, `space_type`.

### 5.2  Template variables

Two filters — one per path, plus a bucket selector that spans both:

| Variable | Type | Query | Purpose |
|---|---|---|---|
| `fb_om` | Query | `label_values(purefb_array_s3_performance_throughput_iops, instance)` | FB from OpenMetrics |
| `fb_otel` | Query | `label_values(flashblade_storage_device_bucket_iops, device_name)` | FB from OTel |
| `bucket` | Query | `label_values({__name__=~"purefb_buckets_.*\|flashblade_storage_device_bucket_.*"}, entity_name \|\| name)` | Bucket selector (both schemas) |

Grafana's `label_values()` can't OR two labels natively, so for `bucket` use
two variables if you want clean binding, or accept that the variable shows
bucket names from one side only and manually free-type the other.

A pragmatic default: two variables, `bucket_om` and `bucket_otel`, with
`Include All = on`, so panels on each side use their matching variable.

### 5.3  Recommended panel layout

**Row 1 — Account-level summary (stats):**

| Panel | PromQL (OpenMetrics) | PromQL (OTel) | Unit |
|---|---|---|---|
| Total object count | `sum(purefb_object_store_accounts_object_count{instance=~"$fb_om"})` | `sum(flashblade_storage_device_obj_store_account_obj_count{device_name=~"$fb_otel"})` | short |
| Total space used | `sum(purefb_object_store_accounts_space_bytes{instance=~"$fb_om",space="total_physical"})` | `sum(flashblade_storage_device_obj_store_account_space_bytes{device_name=~"$fb_otel",space_type="total_physical"})` | bytes |
| Avg DRR | `avg(purefb_object_store_accounts_data_reduction_ratio{instance=~"$fb_om"})` | `avg(flashblade_storage_device_obj_store_account_space_reduction{device_name=~"$fb_otel"})` | none |
| Array S3 IOPS | `sum by (dimension) (purefb_array_s3_performance_throughput_iops{instance=~"$fb_om"})` | `sum by (op) (flashblade_storage_device_iops{device_name=~"$fb_otel", proto="s3"})` if `proto` label exists, else omit | iops |

Create two stat panels side-by-side per row — one OpenMetrics, one OTel — or
use a single panel with both queries and legend formatting to distinguish.

**Row 2 — Top-N buckets (bar gauge or table):**

| Panel | PromQL (OpenMetrics) | PromQL (OTel) |
|---|---|---|
| Top 10 buckets by size | `topk(10, sum by (name) (purefb_buckets_space_bytes{instance=~"$fb_om", space="total_physical"}))` | `topk(10, sum by (entity_name) (flashblade_storage_device_bucket_space_bytes{device_name=~"$fb_otel", space_type="total_physical"}))` |
| Top 10 buckets by objects | `topk(10, sum by (name) (purefb_buckets_object_count{instance=~"$fb_om"}))` | `topk(10, sum by (entity_name) (flashblade_storage_device_bucket_obj_count{device_name=~"$fb_otel"}))` |

**Row 3 — Performance (time series):**

| Panel | PromQL (OpenMetrics) | PromQL (OTel) | Unit |
|---|---|---|---|
| Bucket IOPS over time | `sum by (name) (purefb_buckets_performance_throughput_iops{instance=~"$fb_om", name=~"$bucket_om"})` | `sum by (entity_name, op) (flashblade_storage_device_bucket_iops{device_name=~"$fb_otel", entity_name=~"$bucket_otel"})` | iops |
| Bucket latency over time | `avg by (name, dimension) (purefb_buckets_performance_latency_usec{instance=~"$fb_om", name=~"$bucket_om"})` | `avg by (entity_name, op) (flashblade_storage_device_bucket_latency_us{device_name=~"$fb_otel", entity_name=~"$bucket_otel"})` | µs |
| Bucket bandwidth | `sum by (name, dimension) (purefb_buckets_performance_bandwidth_bytes{instance=~"$fb_om", name=~"$bucket_om"})` | `sum by (entity_name, op) (flashblade_storage_device_bucket_bw_bps{device_name=~"$fb_otel", entity_name=~"$bucket_otel"})` | Bps |

**Row 4 — Space breakdown (stacked time series):**

| Panel | PromQL (OpenMetrics) | PromQL (OTel) |
|---|---|---|
| Bucket space breakdown | `sum by (space) (purefb_buckets_space_bytes{instance=~"$fb_om", name=~"$bucket_om"})` with stacked time-series + legend `{{space}}` | `sum by (space_type) (flashblade_storage_device_bucket_space_bytes{device_name=~"$fb_otel", entity_name=~"$bucket_otel"})` |
| Bucket DRR | `purefb_buckets_space_data_reduction_ratio{instance=~"$fb_om", name=~"$bucket_om"}` | `flashblade_storage_device_bucket_space_reduction{device_name=~"$fb_otel", entity_name=~"$bucket_otel"}` |

### 5.4  Authoring pattern — author in UI, export, patch

1. Create the dashboard in the Grafana UI, iterating on queries until panels
   look right.
2. **Dashboard settings → JSON Model** → copy.
3. Paste into `docker-compose/grafana/dashboards/pure-fb-object-storage.json`.
4. Patch the datasource references to legacy string form:

   ```bash
   python3 <<'EOF'
   import json
   f = 'docker-compose/grafana/dashboards/pure-fb-object-storage.json'
   d = json.load(open(f))
   def fix(o):
       if isinstance(o, dict):
           ds = o.get('datasource')
           if isinstance(ds, dict) and ds.get('type') == 'prometheus':
               o['datasource'] = 'Prometheus'
           for v in o.values(): fix(v)
       elif isinstance(o, list):
           for v in o: fix(v)
   fix(d)
   # Strip editing-time cruft that causes noisy diffs.
   for k in ['__inputs','__requires','version','iteration']:
       d.pop(k, None)
   json.dump(d, open(f,'w'), indent=2)
   EOF
   ```

5. Commit, run `ansible-playbook playbooks/deploy-stack.yml`, then reload:
   `curl -s -u admin:'<pw>' -X POST http://<obs>:3000/api/admin/provisioning/dashboards/reload`.

### 5.5  Validate

In the dashboard:
- Both `fb_om` and `fb_otel` dropdowns populate with their respective FBs.
- `bucket_om` / `bucket_otel` dropdowns populate with bucket names.
- Row 1 stats non-zero for accounts with any data.
- Row 2 shows the actual largest buckets per FB.
- Row 3 draws time-series with live data when traffic is present (try a
  synthetic `mc` or `aws s3` workload if idle).
- Row 4 space breakdown stack sums to the bucket's total space.

If a panel is empty:
- Check the relevant label cardinality: `curl -sG http://prometheus:9090/api/v1/label/<label>/values`.
- Confirm the FB's path is actually reporting that metric family
  (`purefb_buckets_*` requires the `objectstore` scrape job on the
  OpenMetrics side; `flashblade_storage_device_bucket_*` only appears when
  the FB has at least one bucket and OTel is pushing).

### 5.6  Critical Grafana + OTel-specific gotchas (from the lab build)

These are tripwires the first time you build a unified dashboard. The reference
implementation at `docker-compose/grafana/dashboards/pure-fb-object-storage.json`
has all of these baked in.

**Gotcha 1 — OTel staleness window exceeds Prometheus' default.** Pure's native
OTel exporter pushes slow-cadence metrics at intervals longer than the
Prometheus 5-minute staleness window. Affected metrics include:

- `flashblade_storage_device_health`
- `flashblade_storage_device_info`
- `flashblade_storage_device_bucket_space_bytes`
- `flashblade_storage_device_bucket_obj_count`
- `flashblade_storage_device_bucket_space_reduction`
- `flashblade_storage_device_obj_store_account_*`

An instant query (`curl .../api/v1/query?query=flashblade_storage_device_health`)
returns **zero series**, while `/api/v1/series?match[]=...` confirms the series
exist. Wrap every such query in `last_over_time(<metric>[10m])`. This applies
to:
- Stat panel queries
- Top-N (topk) queries
- Table panel queries
- **Template variable queries** — easy to miss; if you use a stale OTel metric
  to populate a dropdown, your OTel FBs won't appear there at all

Performance metrics (`_iops`, `_latency_us`, `_bw_bps`) push at 1-minute cadence
and don't need this wrapping.

**Gotcha 2 — multi-value "All" variable binding.** A Grafana query variable
with `"multi": true` and `"includeAll": true` but **no explicit `allValue`**
expands `$__all` to a literal alternation like `(val1|val2|val3)`. That works
with direct `instance=~"$fb"` selectors, but breaks when used in a query that
unions via `or` with `label_replace` (the alternation doesn't bind into the
`label_replace` regex match as expected, and you get 0 series).

**Fix:** set `"allValue": ".*"` on every variable that's referenced in a
unioned query. `$__all` then expands to `.*` which matches everything
regardless of path.

**Gotcha 3 — bucket vs account-level metrics.** Pure's OTel exporter emits
both:

- Per-bucket: `flashblade_storage_device_bucket_*` (with `entity_name`,
  `entity_owner` labels)
- Per-account: `flashblade_storage_device_obj_store_account_*` (with
  `entity_name` label holding the account name)

For a billing report, account-level totals are often more accurate because
they aggregate across bucket lifecycle state (destroyed buckets, snapshots).
For per-bucket customer drill-down, use the bucket metrics but remember
individual buckets may be near-empty.

**Gotcha 4 — `count()` fails with duplicate identical-label series.** The
OTel collector's `resource_to_telemetry_conversion` can produce series with
the same label set (appearing twice in the underlying data). `count(foo)`
then returns `[]`, and `topk`/`instant` queries break. **Always aggregate
first** with `sum by (...)` or `max by (...)` to collapse duplicates before
counting.

Example: `count(sum by (device_name, entity_name) (last_over_time(flashblade_storage_device_bucket_obj_count[10m])))`
returns 19; a bare `count(flashblade_storage_device_bucket_obj_count)`
returns empty.

---

## Part 6 — Billing export from the dashboard

The object-storage dashboard ships with an **All buckets** table at the
bottom (panel id 19) that lists every bucket across every FB with columns
for FlashBlade, Account, Bucket, Size (bytes), Objects, and DRR.

### 6.1  Interactive export (no scripting)

1. Open the Object Storage dashboard.
2. Optionally filter via the `FlashBlade` or `Bucket` variables, or use the
   per-column text filters in the table header.
3. Click the **All buckets** panel title → **Inspect** → **Data**.
4. Toggle **Formatted data** if you want units applied (readable GB/TB vs raw
   bytes).
5. Click **Download CSV**.

The footer row shows `sum` totals across whatever's displayed, useful for a
quick sanity check before export.

### 6.2  Scripted export from the Prometheus HTTP API

For scheduled or automated exports (cron, CI, backup):

```bash
#!/usr/bin/env bash
# Dumps a billing-ready CSV of every bucket across all monitored FBs.
set -euo pipefail

PROM="${PROM:-http://10.21.101.6:9090}"
DATE="$(date -u +%Y-%m-%d)"
OUT="fb-billing-${DATE}.csv"

# Unified size query (same as dashboard table, flattened to CSV via jq).
Q='sum by (fb, bucket, account) (
  label_replace(label_replace(label_replace(
    max by (instance, account, name) (purefb_buckets_space_bytes{space="total_physical"}),
    "fb", "$1", "instance", "(.+)"),
    "bucket", "$1", "name", "(.+)"),
    "account", "$1", "account", "(.+)")
  or
  label_replace(label_replace(label_replace(
    max by (device_name, entity_name, entity_owner) (last_over_time(flashblade_storage_device_bucket_space_bytes{space_type="total_physical"}[10m])),
    "fb", "$1", "device_name", "(.+)"),
    "bucket", "$1", "entity_name", "(.+)"),
    "account", "$1", "entity_owner", "(.+)")
)'

{
  echo "date,flashblade,account,bucket,size_bytes"
  curl -sG "$PROM/api/v1/query" --data-urlencode "query=$Q" \
    | jq -r --arg d "$DATE" '.data.result[]
        | [$d, .metric.fb, .metric.account, .metric.bucket, .value[1]]
        | @csv'
} > "$OUT"

echo "wrote $OUT ($(wc -l <"$OUT") lines)"
```

Combine with `cron` or a CI trigger and push to your billing system. Add
similar queries for object count and DRR if needed; the labels match so
you can join on (fb, account, bucket).

### 6.3  Caveats

- **Billing semantics:** "Size" uses `space="total_physical"` (OM) /
  `space_type="total_physical"` (OTel) — the actual space consumed on the
  array after reduction. If you charge for logical/virtual space instead,
  swap to `virtual` or `total_provisioned`.
- **Snapshots:** `total_physical` typically excludes snapshots; add
  `space="snapshots"` if you need to include them.
- **Destroyed buckets:** values exist in `space="destroyed"` for a retention
  window. Filter those out or bill them depending on policy.
- **Time anchor:** queries are instantaneous. For monthly billing, run the
  export on the billing-cutoff date at a consistent time, or take a daily
  snapshot and pick the closing value.

---

## Troubleshooting cheatsheet

| Symptom | Likely cause | Fix |
|---|---|---|
| Scrape job `down`, `401 Unauthorized` | Bad/expired API token, or user lacks readonly | Regenerate token against a readonly user, update vault, redeploy |
| Scrape job `down`, `400 Bad Request` | `params.endpoint` missing or FQDN doesn't resolve from exporter container | Verify DNS inside container: `docker exec pure-fb-exporter nslookup <fb-fqdn>` |
| New scrape jobs don't appear after `/-/reload` | Known Prometheus reload flake | `docker restart prometheus` |
| OTel "Test delivery" fails silently, nothing in collector logs | TLS mismatch (FB sending TLS, collector plaintext) | Disable TLS on FB export target |
| Dashboards load but panels say "Datasource not found" | JSON has `${DS_PROMETHEUS}` or UID-form datasource ref | Strip `__inputs`; replace with `"datasource": "Prometheus"` |
| OTel path metrics exist but no `device_name` label | `resource_to_telemetry_conversion` disabled in collector exporter | Set `enabled: true` on the prometheus exporter |
| Object-storage panels empty on OpenMetrics side | `objectstore` scrape job not rendered | Verify playbook included `objectstore` in the namespace loop in `prometheus.yml.j2` |
| `purefb_array_s3_performance_*` present but `purefb_buckets_*` absent | FB has no buckets yet, or `objectstore` scrape job removed | Create a test bucket or restore the job |
| OTel metric returns `0 series` to instant query but `/api/v1/series` finds it | Pure's OTel exporter pushes slow-cadence metrics (health, bucket space, obj counts, DRR, info) at intervals longer than Prometheus' 5-min staleness window | Wrap query in `last_over_time(<metric>[10m])`. Affects stat panels, topk, table panels, **and template variables** — use `last_over_time` in variable queries too |
| Grafana dashboard: "All" on a multi-select variable returns no data | With `multi: true` + `includeAll: true` and no explicit `allValue`, `$__all` expands to a concrete-value alternation that breaks the `or`-unioned queries this dashboard uses | Set `"allValue": ".*"` on the variable so default "All" becomes a regex wildcard |
| OTel path shows tiny/zero bucket sizes but account has real data | Pure emits both `flashblade_storage_device_bucket_*` (per bucket) and `flashblade_storage_device_obj_store_account_*` (per account). Some buckets are genuinely empty; the account-level metric aggregates | Query account-level metrics for billing/summary views; only use bucket-level when you need per-bucket granularity |

---

## Appendix — Minimal Prometheus scrape example (no Ansible)

If you're reproducing this outside this repo's Ansible workflow, this is the
minimum `prometheus.yml` fragment you need for one FB on each path:

```yaml
scrape_configs:
  # OpenMetrics path — one job per namespace
  - job_name: "flashblade_om_array"
    metrics_path: /metrics/array
    authorization:
      credentials: "T-<uuid>"
    scrape_interval: 30s
    params:
      endpoint: ['<fb-fqdn>']
    static_configs:
      - targets: ['<exporter-host>:9491']
        labels:
          instance: "<fb-fqdn>"

  # (repeat for clients, filesystems, objectstore, policies, usage)

  # OTel path — scrape the collector's Prometheus exposition
  - job_name: "otel-collector"
    scrape_interval: 30s
    static_configs:
      - targets: ["<otel-collector-host>:8889"]
```

And the minimum OTel collector config:

```yaml
receivers:
  otlp:
    protocols:
      grpc:
        endpoint: 0.0.0.0:4317

exporters:
  prometheus:
    endpoint: 0.0.0.0:8889
    namespace: flashblade
    resource_to_telemetry_conversion:
      enabled: true

service:
  pipelines:
    metrics:
      receivers: [otlp]
      exporters: [prometheus]
```

---

## Part 7 — Exporting & sharing dashboards

Two distinct JSON forms for our dashboards, each serving a different audience:

| Form | Location | Datasource reference | Who uses it |
|---|---|---|---|
| **Provisioned** (live) | `docker-compose/grafana/dashboards/` | `"datasource": "Prometheus"` (literal string) | This repo's Ansible stack only. File-provisioning binds to the datasource named `Prometheus`. |
| **Portable** (shareable) | `dashboards/` | `"datasource": {"type": "prometheus", "uid": "${DS_PROMETHEUS}"}` | Anyone with a Grafana + Prometheus scraping the same schemas. Grafana prompts the importer to bind their own datasource. |

See [`../dashboards/README.md`](../dashboards/README.md) for the portable
bundle and import instructions.

### 7.1 Generating the portable form from the provisioned form

Edit the provisioned JSON in `docker-compose/grafana/dashboards/` (or author
in the Grafana UI against our stack, then save). To regenerate the portable
versions:

```bash
python3 <<'EOF'
import json, os

# list whichever dashboards you changed (or iterate over all of ours)
OURS = [
  'pure-fb-overview-unified.json',
  'pure-fb-capacity-space-unified.json',
  'pure-fb-filesystem-detail-unified.json',
  'pure-fb-hardware-health-unified.json',
  'pure-fb-system-detail-unified.json',
  'pure-fb-object-storage.json',
  'pure-fb-otel-overview.json',
]

for fname in OURS:
    src = f'docker-compose/grafana/dashboards/{fname}'
    dst = f'dashboards/{fname}'
    d = json.load(open(src))

    d['__inputs'] = [{
        "name": "DS_PROMETHEUS", "label": "Prometheus", "description": "",
        "type": "datasource", "pluginId": "prometheus", "pluginName": "Prometheus"
    }]
    d['__requires'] = [
        {"type": "grafana", "id": "grafana", "name": "Grafana", "version": "10.0.0"},
        {"type": "datasource", "id": "prometheus", "name": "Prometheus", "version": "1.0.0"}
    ]

    def walk(o):
        if isinstance(o, dict):
            if o.get('datasource') == 'Prometheus':
                o['datasource'] = {"type": "prometheus", "uid": "${DS_PROMETHEUS}"}
            for v in o.values():
                if isinstance(v, (dict, list)): walk(v)
        elif isinstance(o, list):
            for v in o: walk(v)
    walk(d)
    d.pop('id', None); d['version'] = 1
    json.dump(d, open(dst, 'w'), indent=2)
    print(f'  {dst}')
EOF
```

### 7.2 Exporting from the Grafana UI (ad-hoc)

For one-off sharing without touching the repo:

1. Open the dashboard.
2. **Share** → **Export** tab.
3. Tick **"Export for sharing externally"** — this rewrites datasource refs
   and adds `__inputs`.
4. **Save to file** or **View JSON** → copy.

### 7.3 Importing into a different Grafana

Whoever you're sharing with:

1. Save the JSON file.
2. In their Grafana: **Dashboards → New → Import** → **Upload JSON file**.
3. Grafana shows the `__inputs` bindings — they pick their Prometheus
   datasource and click **Import**.

Panels light up immediately if their Prometheus is scraping `purefb_*`
(OpenMetrics path) or `flashblade_storage_device_*` (OTel path). If neither
schema is present the panels show "No data" but the dashboard loads cleanly.

---

## Lessons learned during dashboard build

Collected from the sessions where we built and iterated the unified dashboard
family. Each has a more detailed explanation inline where relevant, but this
is the consolidated list for quick reference.

### Grafana / PromQL interaction gotchas

- **`label_values(<complex-expr>, <label>)` causes a UI toast error.**
  Grafana's Prometheus plugin first hits `/api/v1/label/<label>/values?match[]=<expr>`
  which only accepts series selectors — not full expressions. Anything with
  `or`, `label_replace()`, or arithmetic gets a 400 ("1:14: parse error"-style)
  and surfaces as a persistent toast even though Grafana silently falls back
  to the query endpoint and dropdowns still populate. **Fix:** use
  `query_result(<expr>)` + `regex` extraction — same pattern Pure's canned
  dashboards use. Forces the query endpoint and avoids the spurious 400.

- **Multi-select "All" needs explicit `allValue`.** `"includeAll": true,
  "multi": true` without an `allValue` makes `$__all` expand to a
  value-alternation like `(fb1|fb2|fb3)`. That's fine for direct
  `instance=~"$fb"` selectors, but breaks panels that inject `$fb` inside
  `label_replace(...)` or union expressions. Always set `"allValue": ".+"`
  on variables that feed unified `or`-expressions.

- **Legacy string datasource form is required for file-provisioned dashboards.**
  `"datasource": "Prometheus"` works under file provisioning because Grafana
  resolves by name. The Grafana-10+ UID-object form (`{"type": "...", "uid": "..."}`)
  does **not** bind cleanly under provisioning unless you also set the UID
  explicitly in the provisioning config. Keep it as a string in provisioned
  copies; swap to UID+`${DS_PROMETHEUS}` only for exportable copies.

- **Dashboard `current` defaults shipped by upstream are poison.** Pure's
  canned dashboards have hardcoded `current.value` fields from their test
  lab (e.g. `instance: "10.21.241.90"`, `job: "pure_flashblade1"`). These
  values persist across Grafana versions and break any dashboard on first
  load until the user manually repicks. Our patch clears those to `{}` so
  variables resolve fresh. If you import from upstream, strip these.

- **`count()` fails on duplicate-label series.** OTel exporters sometimes
  emit series with identical label sets (different scope, same exposed
  labels). `count(metric)` returns `[]` instead of the expected count.
  Aggregate first: `count(sum by (key_labels) (metric))`.

- **Chained variable queries need `=~` not `=`.** `label_values(m{job="$job"}, ...)`
  breaks the moment `$job` is multi-value because the alternation regex
  doesn't match exact. Always `job=~"$job"`, `instance=~"$instance"`.

### OTel-specific gotchas

- **Slow-cadence OTel metrics go stale within Prometheus' 5-minute default.**
  Affected families (observed): `flashblade_storage_device_health`,
  `_info`, `_bucket_*`, `_obj_store_account_*`. Instant queries return
  empty even though `/api/v1/series?match[]=...` shows data. Wrap with
  `last_over_time(metric[10m])`. This applies to panels **and** variable
  queries that feed on these metrics.

- **TLS on by default in the FB OTel export config, with no "insecure"
  toggle.** The "Test connectivity" nc check passes on plaintext; the
  "Test delivery" gRPC send fails silently when the collector listener
  is plaintext. Our collector is plaintext, so disable TLS on the FB
  export target in the GUI. This is noted in GAPS.md as "needs product
  fix" — Pure should either offer an insecure mode or make TLS
  handshake failures more visible.

- **Pure emits duplicate-label OTel series periodically.** Harmless in
  aggregation queries (`sum by (...)`), but `count()` without aggregation
  misfires. Noted in GAPS.md.

- **`exported_storage_type` filter matters for cross-path comparability.**
  OTel's space metrics split by `all`/`file`/`object`; OM's don't. When
  writing a "total space" stat, always filter OTel to
  `exported_storage_type="all"` or your number double-counts.

### Pure canned dashboard gotchas

- **Metric renames broke canned FB dashboards.** Upstream renamed
  `purefb_filesystems_*` → `purefb_file_systems_*`, `purefb_hw_status` →
  `purefb_hardware_health`, etc. The dashboards ship referencing the old
  names. Our patch script in Part 3 remaps them. Always validate against
  your actual exporter version before trusting a downloaded dashboard.

- **Pure dashboards filter by `env` label that our stack didn't emit.**
  Added as a static scrape label in `prometheus.yml.j2` (`env: "lab"`) to
  make canned dashboards functional out-of-the-box.

- **Variable refresh=1 means "on dashboard load."** Changes to metrics
  won't update dropdowns until a dashboard reload. Hard-refresh
  (Cmd+Shift+R) is the quickest user-side fix.

### Prometheus operational quirks

- **`/-/reload` flake.** Returns 200 but occasionally does not reload the
  scrape config. `docker restart prometheus` reliably fixes it. Data
  isn't lost — TSDB is on the persistent `/var/lib/docker` mount.

- **Static scrape labels beat relabel_configs for cross-cutting labels**
  like `env`. `static_configs.labels.env: lab` in `prometheus.yml.j2`
  is one line vs. a multi-stage relabel rule.

### Schema design lessons for unified dashboards

- **Two queries per panel (`refId A` for OM, `refId B` for OTel) with
  a clear legend prefix** is more maintainable than trying to unify via
  `label_replace()` + `or`. Exceptions: stat/gauge panels where you want
  a single headline number — unify there via `or vector(0)` arithmetic.

- **Filter variables at the query level, not in Grafana transformations.**
  Transformation-level filtering happens after data transfer; query-level
  filtering uses Prometheus to shrink the payload. For tables scoped by a
  variable, always `instance=~"$fb"` / `name=~"$filesystem"` in the expr.

- **Tag every dashboard.** Our conventions: `pure`, `flashblade` or
  `flasharray`, `unified` (for dashboards that query both paths), `otel`
  (for dashboards that query OTel metrics). Tags drive Grafana search and
  help isolate schema-specific dashboards.

---

## Appendix C — Full artifact inventory

After all dashboards land, `docker-compose/grafana/dashboards/` contains:

| File | Source | Tagged |
|---|---|---|
| `grafana-purefa-flasharray-overview.json` | Pure FA exporter repo (patched) | pure, flasharray |
| `grafana-purefb-flashblade-overview.json` | Pure FB exporter repo (patched) | pure, flashblade |
| `pure-fb-filesystem-detail.json` | Pure FB exporter repo (patched) | pure, flashblade |
| `pure-fb-hardware-health.json` | Pure FB exporter repo (patched) | pure, flashblade |
| `pure-fb-landscape.json` | Pure FB exporter repo (patched) | pure, flashblade |
| `pure-fb-storage-usage.json` | Pure FB exporter repo (patched) | pure, flashblade |
| `pure-fb-storage-usage-details.json` | Pure FB exporter repo (patched) | pure, flashblade |
| `pure-fb-system-detail.json` | Pure FB exporter repo (patched) | pure, flashblade |
| `pure-fb-overview-unified.json` | **This repo** | pure, flashblade, unified, otel |
| `pure-fb-capacity-space-unified.json` | **This repo** | pure, flashblade, capacity, unified, otel |
| `pure-fb-filesystem-detail-unified.json` | **This repo** | pure, flashblade, filesystem, unified, otel |
| `pure-fb-hardware-health-unified.json` | **This repo** | pure, flashblade, hardware, unified, otel |
| `pure-fb-system-detail-unified.json` | **This repo** | pure, flashblade, system, unified, otel |
| `pure-fb-object-storage.json` | **This repo** | pure, flashblade, object-storage, s3, otel |
| `pure-fb-otel-overview.json` | **This repo** | pure, flashblade, otel |

The `dashboards/` directory at the repo root contains the 7 portable versions
of the "this repo" dashboards for external import.

