# Pure FlashArray NVMe/TCP Dashboard Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build a Grafana dashboard (`pure-fa-nvme.json`) focused on front-end NVMe-over-TCP for the Pure FlashArrays in Prometheus.

**Architecture:** A single Grafana dashboard JSON, hand-authored to match the existing `grafana-purefa-flasharray-overview.json` conventions (templating vars, `$datasource`, tags). Three panel rows: NVMe/TCP front-end (NVMe/TCP-filtered via the `services` label and `and on(instance,name)` joins), protocol-agnostic connection/connectivity state, and array front-end performance context. Verification is done by running each panel's PromQL against the live lab Prometheus over SSH *before* embedding it, then validating the final JSON.

**Tech Stack:** Grafana dashboard JSON (schemaVersion ~39), Prometheus / Purity native `purefa_*` metrics. Lab access: `ssh root@10.21.101.6` (host `rp-util1`), Prometheus at `http://localhost:9090` on that host.

**Spec:** `docs/superpowers/specs/2026-06-19-pure-fa-nvme-tcp-dashboard-design.md`

---

## Notes for the implementer

- **Live query helper** (run from your workstation; validates a PromQL expr returns data):
  ```bash
  q(){ ssh -o ConnectTimeout=8 -o BatchMode=yes root@10.21.101.6 \
    "curl -gs 'http://localhost:9090/api/v1/query?query=$1'" \
    | python3 -c "import sys,json;r=json.load(sys.stdin)['data']['result'];print('rows:',len(r));[print(' ',x['metric'],'=',x['value'][1]) for x in r[:5]]"; }
  ```
  URL-encode the expr (`{`=`%7B`, `}`=`%7D`, space=`%20`, `"`=`%22`, `=`=`%3D` inside values is optional). Expected: `rows: >0` on NVMe/TCP arrays.
- **NVMe/TCP arrays** (where R1 panels show data): `sn1-c60-e12-16.fsa.lab`, `sn1-x90r2-f07-27.fsa.lab`. Other arrays render empty R1 panels — that is correct, not a bug.
- **Filter idiom for NVMe/TCP perf:** `<perf_metric>{dimension=...} and on(instance,name) purefa_network_interface_speed_bandwidth_bytes{services=~".*nvme-tcp.*"}` — `and on` filters, never `*` (which would scale by link speed).
- **Style to copy** from `docker-compose/grafana/dashboards/grafana-purefa-flasharray-overview.json`: timeseries `fieldConfig.defaults` (palette-classic, smooth line, fillOpacity 10, legend table w/ mean/min/max/last), panel `datasource = {type: prometheus, uid: "$datasource"}`, templating list for `datasource`/`env`/`instance`.
- Author the JSON in `dashboards/pure-fa-nvme.json` first; the final task copies it byte-for-byte to `docker-compose/grafana/dashboards/pure-fa-nvme.json`.

---

## Task 1: Validate all panel queries against live Prometheus

**Files:** none (verification only — establishes every panel binds to real data before authoring)

- [ ] **Step 1: Validate R1 NVMe/TCP queries return data**

Run each via the `q` helper; expected `rows > 0`:
```
count by(instance)(purefa_network_interface_speed_bandwidth_bytes{services=~".*nvme-tcp.*",enabled="true"})
purefa_network_port_info{nqn!="",portal!=""}
purefa_network_interface_performance_bandwidth_bytes{dimension="received_bytes_per_sec"} and on(instance,name) purefa_network_interface_speed_bandwidth_bytes{services=~".*nvme-tcp.*"}
purefa_network_interface_performance_throughput_pkts and on(instance,name) purefa_network_interface_speed_bandwidth_bytes{services=~".*nvme-tcp.*"}
purefa_network_interface_performance_errors and on(instance,name) purefa_network_interface_speed_bandwidth_bytes{services=~".*nvme-tcp.*"}
```
Expected: throughput rows show plausible bytes/sec magnitudes (NOT 1e16 — that would mean a `*` join slipped in).

- [ ] **Step 2: Validate R2 connectivity/connection queries**

```
count by(status)(purefa_host_connectivity_info)
purefa_host_connectivity_info{status=~"critical|unhealthy"}
count by(instance)(purefa_host_connections_info)
```
Expected: status counts return (`healthy`/`critical`/`unhealthy`), connection counts per array > 0.

- [ ] **Step 3: Validate R3 array perf + templating source**

```
purefa_array_performance_latency_usec
purefa_array_performance_throughput_iops
purefa_array_performance_bandwidth_bytes
purefa_array_performance_queue_depth_ops
purefa_info
```
Capture the distinct `dimension` values for latency/iops/bandwidth (use `... by(dimension)`), and confirm `purefa_info` exposes `env` + `instance` labels for templating.

- [ ] **Step 4: Record findings** — note any query returning 0 rows; adjust the expr in this plan before proceeding. No commit (verification task).

---

## Task 2: Author dashboard skeleton + templating

**Files:**
- Create: `dashboards/pure-fa-nvme.json`

- [ ] **Step 1: Write the JSON skeleton**

Top-level keys mirroring the overview dashboard:
```json
{
  "annotations": {"list": [{"builtIn": 1, "datasource": {"type": "grafana", "uid": "-- Grafana --"}, "enable": true, "hide": true, "iconColor": "rgba(0, 211, 255, 1)", "name": "Annotations & Alerts", "type": "dashboard"}]},
  "description": "Front-end NVMe/TCP for Pure FlashArrays: port state, throughput, errors, and connectivity.",
  "editable": true, "fiscalYearStartMonth": 0, "graphTooltip": 0, "links": [], "liveNow": false,
  "panels": [],
  "refresh": "", "schemaVersion": 39, "style": "dark",
  "tags": ["pure", "storage", "purestorage", "flasharray", "fa", "nvme", "nvme-tcp"],
  "templating": {"list": []},
  "time": {"from": "now-3h", "to": "now"},
  "timepicker": {}, "timezone": "", "title": "Pure FlashArray · NVMe/TCP", "uid": "pure-fa-nvme", "version": 1, "weekStart": ""
}
```

- [ ] **Step 2: Add templating list** (`datasource`, `env`, `instance`), copied/adapted from the overview dashboard:
```json
{"current": {}, "hide": 0, "includeAll": false, "label": "Datasource", "name": "datasource", "options": [], "query": "prometheus", "refresh": 1, "regex": "", "type": "datasource"},
{"current": {}, "datasource": {"type": "prometheus", "uid": "$datasource"}, "definition": "query_result(purefa_info)", "hide": 0, "includeAll": true, "multi": true, "name": "env", "query": {"query": "query_result(purefa_info)", "refId": "StandardVariableQuery"}, "refresh": 2, "regex": "/.*env=\"([^\"]+)\".*/", "type": "query"},
{"current": {}, "datasource": {"type": "prometheus", "uid": "$datasource"}, "definition": "query_result(purefa_info{instance=~\".+\"})", "hide": 0, "includeAll": true, "multi": true, "name": "instance", "query": {"query": "query_result(purefa_info{instance=~\".+\"})", "refId": "StandardVariableQuery"}, "refresh": 2, "regex": "/.*instance=\"([^\"]+)\".*/", "type": "query"}
```
(Confirm the `regex` extraction matches the overview dashboard's exact pattern in Task 1's `purefa_info` output; adjust if labels differ.)

- [ ] **Step 3: Add a header text panel** (gridPos at top, `y:0`) naming NVMe/TCP arrays:
```json
{"type": "text", "title": "About this dashboard", "gridPos": {"h": 3, "w": 24, "x": 0, "y": 0}, "options": {"mode": "markdown", "content": "Front-end **NVMe/TCP** view. NVMe/TCP is currently enabled on **sn1-c60-e12-16** and **sn1-x90r2-f07-27** — R1 panels are empty on other arrays by design. R2 (connectivity/connections) spans **all transports**, not just NVMe; Purity exposes no NVMe-specific session metric."}}
```

- [ ] **Step 4: Validate JSON** — `python3 -m json.tool dashboards/pure-fa-nvme.json > /dev/null && echo OK`. Expected: `OK`.

- [ ] **Step 5: Commit**
```bash
git add dashboards/pure-fa-nvme.json docs/superpowers/plans/2026-06-19-pure-fa-nvme-tcp-dashboard.md
git commit -m "feat(dashboard): NVMe/TCP dashboard skeleton + templating"
```

---

## Task 3: R1 — NVMe/TCP Front-End panels

**Files:**
- Modify: `dashboards/pure-fa-nvme.json` (append to `panels`)

- [ ] **Step 1: Add a row header** `"NVMe/TCP Front-End"` (`type: row`, `gridPos.y: 3`).

- [ ] **Step 2: Add the six R1 panels** with these exprs (all templated `{instance=~"$instance",env=~"$env"}` where the metric carries those labels; perf metrics use the `and on` join which carries instance via LHS):

  1. **Enabled NVMe/TCP ports** (stat):
     `count by(instance)(purefa_network_interface_speed_bandwidth_bytes{services=~".*nvme-tcp.*",enabled="true",instance=~"$instance",env=~"$env"})`
  2. **NVMe/TCP target portals** (table, `instant: true`, transform: organize → keep `instance,name,nqn,portal`):
     `purefa_network_port_info{nqn!="",portal!="",instance=~"$instance",env=~"$env"}`
  3. **RX/TX throughput per NVMe/TCP port** (timeseries, unit `binBps`), two targets:
     `purefa_network_interface_performance_bandwidth_bytes{dimension="received_bytes_per_sec",instance=~"$instance",env=~"$env"} and on(instance,name) purefa_network_interface_speed_bandwidth_bytes{services=~".*nvme-tcp.*"}` (legend `{{instance}} {{name}} RX`)
     `...{dimension="transmitted_bytes_per_sec"...} and on(...) ...` (legend `... TX`)
  4. **Packets/sec per NVMe/TCP port** (timeseries, unit `pps`):
     `purefa_network_interface_performance_throughput_pkts{instance=~"$instance",env=~"$env"} and on(instance,name) purefa_network_interface_speed_bandwidth_bytes{services=~".*nvme-tcp.*"}` (legend `{{instance}} {{name}} {{dimension}}`)
  5. **Errors/discards per NVMe/TCP port** (timeseries):
     `purefa_network_interface_performance_errors{instance=~"$instance",env=~"$env"} and on(instance,name) purefa_network_interface_speed_bandwidth_bytes{services=~".*nvme-tcp.*"}` (legend `{{name}} {{dimension}}`)
  6. **NVMe/TCP port speed & state** (table, `instant: true`, transform keep `instance,name,enabled,ethsubtype,Value`; unit `binBps` on Value):
     `purefa_network_interface_speed_bandwidth_bytes{services=~".*nvme-tcp.*",instance=~"$instance",env=~"$env"}`

  Each timeseries panel copies the overview's `fieldConfig.defaults` + `options.legend` block. Assign sequential `gridPos` (two-wide layout, `w:12`).

- [ ] **Step 3: Validate JSON** — `python3 -m json.tool dashboards/pure-fa-nvme.json > /dev/null && echo OK`.

- [ ] **Step 4: Commit**
```bash
git add dashboards/pure-fa-nvme.json
git commit -m "feat(dashboard): add NVMe/TCP front-end panels (R1)"
```

---

## Task 4: R2 — Connection / Connectivity State panels

**Files:**
- Modify: `dashboards/pure-fa-nvme.json`

- [ ] **Step 1: Add row header** `"Connection / Connectivity State (all transports)"`.

- [ ] **Step 2: Add three panels**, each with a `description` stating it spans all transports:

  1. **Host connectivity by status** (piechart):
     `count by(status)(purefa_host_connectivity_info{instance=~"$instance",env=~"$env"})` (legend `{{status}}`)
  2. **Critical / unhealthy hosts** (table, `instant: true`, transform keep `instance,host,status,details`):
     `purefa_host_connectivity_info{status=~"critical|unhealthy",instance=~"$instance",env=~"$env"}`
  3. **Host↔volume connections per array** (timeseries, unit `short`):
     `count by(instance)(purefa_host_connections_info{instance=~"$instance",env=~"$env"})` (legend `{{instance}}`)

- [ ] **Step 3: Validate JSON** — `python3 -m json.tool ... && echo OK`.

- [ ] **Step 4: Commit**
```bash
git add dashboards/pure-fa-nvme.json
git commit -m "feat(dashboard): add connectivity/connection panels (R2)"
```

---

## Task 5: R3 — Array Front-End Performance (context) panels

**Files:**
- Modify: `dashboards/pure-fa-nvme.json`

- [ ] **Step 1: Add row header** `"Array Front-End Performance (context — all transports)"`.

- [ ] **Step 2: Add four timeseries panels** (dimension values confirmed in Task 1; use the actual values found):

  1. **Latency by op type** (unit `µs`): `purefa_array_performance_latency_usec{instance=~"$instance",env=~"$env"}` (legend `{{instance}} {{dimension}}`)
  2. **IOPS by type** (unit `iops`): `purefa_array_performance_throughput_iops{instance=~"$instance",env=~"$env"}`
  3. **Bandwidth by type** (unit `binBps`): `purefa_array_performance_bandwidth_bytes{instance=~"$instance",env=~"$env"}`
  4. **Queue depth** (unit `short`): `purefa_array_performance_queue_depth_ops{instance=~"$instance",env=~"$env"}`

- [ ] **Step 3: Validate JSON** — `python3 -m json.tool ... && echo OK`.

- [ ] **Step 4: Commit**
```bash
git add dashboards/pure-fa-nvme.json
git commit -m "feat(dashboard): add array front-end perf context panels (R3)"
```

---

## Task 6: Mirror to provisioning dir, final validation, README

**Files:**
- Create: `docker-compose/grafana/dashboards/pure-fa-nvme.json`
- Modify: `dashboards/README.md` (if it lists dashboards)

- [ ] **Step 1: Copy the dashboard** to the provisioning directory:
```bash
cp dashboards/pure-fa-nvme.json docker-compose/grafana/dashboards/pure-fa-nvme.json
diff dashboards/pure-fa-nvme.json docker-compose/grafana/dashboards/pure-fa-nvme.json && echo IDENTICAL
```
Expected: `IDENTICAL`.

- [ ] **Step 2: Final JSON validation on both copies**:
```bash
for f in dashboards/pure-fa-nvme.json docker-compose/grafana/dashboards/pure-fa-nvme.json; do python3 -m json.tool "$f" > /dev/null && echo "OK $f"; done
```

- [ ] **Step 3: Update `dashboards/README.md`** — add a `pure-fa-nvme.json` row in the same format as existing entries (check the file's table/list style first; if no listing exists, skip).

- [ ] **Step 4: (Optional) Live import smoke test** — if Grafana is reachable, confirm the dashboard provisions without datasource errors; otherwise note manual import as the acceptance step.

- [ ] **Step 5: Commit**
```bash
git add docker-compose/grafana/dashboards/pure-fa-nvme.json dashboards/README.md
git commit -m "feat(dashboard): provision NVMe/TCP dashboard + update README"
```

---

## Acceptance (from spec)

- [ ] JSON valid in both locations; copies identical.
- [ ] R1 panels render NVMe/TCP data on `sn1-c60-e12-16` and `sn1-x90r2-f07-27`, empty (not erroring) elsewhere.
- [ ] Throughput magnitudes are real bytes/sec (no link-speed scaling artifact).
- [ ] R2 status counts match the discovery snapshot ballpark; descriptions note all-transport scope.
- [ ] Templating (`$datasource`/`$env`/`$instance`) drives all panels.
