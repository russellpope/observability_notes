# Exportable Grafana dashboards

Portable versions of the unified / custom FlashBlade and FlashArray
dashboards we built in this repo. These JSON files are wrapped with a
`__inputs` block and use `${DS_PROMETHEUS}` placeholders so any Grafana
instance can import them and bind its own Prometheus datasource at import
time.

**This directory is for sharing.** For the Ansible-provisioned live stack
copies (which use a hardcoded `"datasource": "Prometheus"` and are rewritten
in place by Grafana's file provisioning), see
`docker-compose/grafana/dashboards/`.

## What's here

| File | Dashboard | Covers |
|---|---|---|
| `pure-fb-overview-unified.json` | FlashBlade Overview (Unified) | Top-level view mirroring Pure's canned Overview — system detail, alerts, space, IOPS, latency, bandwidth, top-N filesystems/buckets |
| `pure-fb-capacity-space-unified.json` | Capacity & Space (Unified) | Per-FB capacity gauge, DRR, free space, growth, per-FS and per-bucket tables |
| `pure-fb-filesystem-detail-unified.json` | Filesystem Detail (Unified) | Per-FS drill-down. User/group usage flagged OpenMetrics-only |
| `pure-fb-hardware-health-unified.json` | Hardware Health (Unified) | Chassis / blades / bays / FMs / ETH / fans / PSUs. Unhealthy-component tables per path |
| `pure-fb-system-detail-unified.json` | System Detail (Unified) | Per-FB system view (firmware, capacity, performance, top-N filesystems) |
| `pure-fb-object-storage.json` | Object Storage | Bucket-focused dashboard for chargeback/billing (total size, object count, top-N, full table CSV export) |
| `pure-fb-otel-overview.json` | OTel Overview | Designed for the OTel-path FB specifically — uses `flashblade_storage_device_*` schema only |
| `pure-fa-nvme.json` | FlashArray NVMe/TCP | FlashArray front-end NVMe-over-TCP — enabled ports, target portals, RX/TX throughput, packets, errors, plus all-transport connectivity/connection context. Uses `purefa_*` (native Purity OpenMetrics) |
| `pure-fa-host-client.json` | FlashArray Per-Client (Host) | Per-host IOPS / bandwidth / latency / IO-size / space / DRR / connections / connectivity, with fleet top-N + `$host` drill-down. Includes an explicit note that host stats are all-transport (no per-client NVMe attribution). Uses `purefa_host_*` |

The FlashBlade dashboards query **both** the OpenMetrics path (`purefb_*` via
`pure-fb-om-exporter`) and the native OTel path
(`flashblade_storage_device_*` via the OpenTelemetry collector). Panels that
exist on only one path are labeled in their title. See
[../docs/GAPS.md](../docs/GAPS.md) for the cross-path metric gap tracker.

`pure-fa-nvme.json` is the lone **FlashArray** dashboard here; it queries
`purefa_*` only. NVMe/TCP is isolated via the `services="nvme-tcp"` interface
label. Connectivity/connection panels span all transports (Purity exposes no
NVMe-specific session metric); R1 throughput/error panels populate only on
arrays with NVMe/TCP enabled.

## Importing into a different Grafana

1. In your Grafana: **Dashboards → New → Import**.
2. Click **Upload JSON file** and pick the file from this directory.
3. Grafana parses the `__inputs` and prompts you to bind a Prometheus
   datasource — pick the one scraping your Pure data.
4. Click **Import**.

The dashboards assume your Prometheus is scraping the same metric schemas
we use in this repo:
- `purefb_*` via `pure-fb-om-exporter` (see
  [../docs/HOWTO-flashblade-monitoring.md](../docs/HOWTO-flashblade-monitoring.md)
  Part 1 for setup)
- `flashblade_storage_device_*` via the OpenTelemetry collector receiving
  OTLP push from Purity//FB (HOWTO Part 2)

Without those metrics in your Prometheus the panels will show "No data" but
won't error.

## Refreshing the exports

The live provisioned copies in `docker-compose/grafana/dashboards/` are the
source of truth. If you edit those, regenerate these portable versions via
the helper script in `HOWTO-flashblade-monitoring.md` Part 7, or by hand:

```bash
python3 <<'EOF_'
import json
for fname in [...]:  # list the files you changed
    src = f'docker-compose/grafana/dashboards/{fname}'
    dst = f'dashboards/{fname}'
    d = json.load(open(src))
    d['__inputs'] = [{"name": "DS_PROMETHEUS", "label": "Prometheus", "type": "datasource", "pluginId": "prometheus", "pluginName": "Prometheus"}]
    def walk(o):
        if isinstance(o, dict):
            if o.get('datasource') == 'Prometheus':
                o['datasource'] = {"type": "prometheus", "uid": "${DS_PROMETHEUS}"}
            for v in o.values():
                if isinstance(v, (dict, list)): walk(v)
        elif isinstance(o, list):
            for v in o: walk(v)
    walk(d); d.pop('id', None); d['version'] = 1
    json.dump(d, open(dst, 'w'), indent=2)
EOF_
```

Or export from the Grafana UI with the **"Export for sharing externally"**
checkbox enabled (Dashboard → Share → Export → JSON with `__inputs`).
