# Runbook — rp-observability

Lessons learned, context, and procedures from the initial deployment. Treat
this as living documentation — update when you discover something that would
help the next person (or next you).

---

## What's actually deployed

Three Ubuntu 22.04 VMs in the FSA lab:

| Host | IP | Role | Storage layout |
|------|-----|------|----------------|
| `rp-util1` | 10.21.101.6 | Observability stack (all services) | 40 GB boot (LVM/XFS) + 1 TB XFS at `/var/lib/docker` |
| `rp-util2` | 10.21.101.14 | Grafana Alloy agent | 40 GB boot; 1 TB disk unused |
| `rp-util3` | 10.21.101.18 | Grafana Alloy agent | 40 GB boot; 1 TB disk unused |

Stack on `rp-util1` (Docker Compose): Prometheus, Grafana, Loki, Alertmanager,
OTel Collector, pure-fb-om-exporter. **No pure-fa-exporter** — Purity 6.4+ serves
OpenMetrics natively.

**Data paths:**
- FlashArrays: Prometheus → `https://<fa>/metrics/<ns>` direct with bearer auth.
- FlashBlades (choose one per array):
  - OpenMetrics proxy → Prometheus → `http://pure-fb-exporter:9491/metrics/<ns>` with bearer auth + `params.endpoint=<fb>` (→ `purefb_*` metrics).
  - Native OTel push → `obs-vm:4317` → OTel collector → Prom scrape on `:8889` (→ `flashblade_*` metrics).
- Linux hosts: Alloy → remote_write metrics + push logs.

Canned FB dashboards from Pure's exporter repo use `purefb_*` names and only
cover the OpenMetrics path. The `Pure Storage FlashBlade - OTel Overview`
dashboard in this repo covers the OTel path.

---

## Prerequisites (on your Mac)

- `ansible-core` ≥ 2.16 with collections `community.general`, `ansible.posix`
- `ssh-agent` loaded with the lab key (passphrase is entered once per shell):
  ```
  ssh-add ~/.ssh/lab_key
  ssh-add -l               # should list the key
  ```
  macOS keychain persistence: `ssh-add --apple-use-keychain ~/.ssh/lab_key` +
  `AddKeysToAgent yes` / `UseKeychain yes` in `~/.ssh/config`.
- VMs exist, reachable by SSH as `root` from your workstation (no prompt needed
  thanks to agent forwarding). Ansible's `ansible.cfg` already enables
  `ControlMaster=auto` + `ForwardAgent=yes`.

---

## First-time deployment (from scratch)

```bash
cd ansible

# 1. Populate inventory (already committed as hosts.yml; re-generate from example if needed).
#    Hosts in `observability:` get the full stack; hosts in `linux_clients:` get Alloy.

# 2. Fill in vault. Password file lives at ansible/.vault-password-file (gitignored).
ansible-vault edit inventory/group_vars/all/vault.yml
#   Required fields:
#     vault_observability_ip / hostname / domain
#     vault_grafana_admin_user / _password
#     vault_pure_fa_tokens:  list of {address, api_token}
#     vault_pure_fb_tokens:  list of {address, api_token}   (OpenMetrics path — leave [] if all FBs use OTel push)

# 3. Prep the data disk on obs-vm (only once per fresh VM).
ansible-playbook playbooks/prepare-storage.yml
#   Formats /dev/sdb as XFS and mounts it at /var/lib/docker.
#   MUST run before Docker is installed so the mount point is empty.

# 4. Deploy the stack.
ansible-playbook playbooks/deploy-stack.yml

# 5. Deploy Alloy agents.
ansible-playbook playbooks/deploy-alloy.yml

# 6. Enable FlashBlade telemetry (manual, GUI or API on each FB):
#    Settings → External Telemetry → Add export target → 10.21.101.6:4317, TLS OFF
#    Test delivery before saving. See "Gotchas" for the TLS trap.
```

**Verify:**

```bash
# From Mac (LAN ports 3000/9090 work from anywhere; 3100/9093 are LAN-only)
curl -s http://10.21.101.6:9090/-/healthy
curl -s -u admin:'<pw>' http://10.21.101.6:3000/api/search?type=dash-db | jq length

# From anywhere in the lab
ssh root@10.21.101.6 'curl -s http://localhost:9090/api/v1/targets?state=active' | jq '.data.activeTargets[] | {pool: .scrapePool, health}'
```

Expected after full deploy: one job per `(FA, namespace)` × 5, per `(FB, namespace)` × 6 (OpenMetrics path), one `otel-collector` job, plus Prom/Grafana self-monitoring.

---

## Day-2 flows

### Add a FlashArray
1. Generate API token on the FA: `pureadmin create --api-token --user pureuser`.
2. Append `{address, api_token}` to `vault_pure_fa_tokens` (`ansible-vault edit ...`).
3. `ansible-playbook playbooks/deploy-stack.yml` — renders new scrape jobs and hits `/-/reload`.
4. If new jobs don't appear in `/api/v1/targets` within ~30 s, hard-restart the container:
   `ssh root@obs-vm 'docker restart prometheus'`. See "Prometheus reload quirk".

### Add a FlashBlade (OpenMetrics path)
Same as FA but append to `vault_pure_fb_tokens`. The exporter is stateless —
no config reload needed inside the exporter container.

### Add a FlashBlade (OTel path)
FB GUI → Settings → External Telemetry → add target `obs-vm:4317`, TLS off,
enable the telemetry types you want (metrics/health/events). Nothing to change
on our side.

### Add a Linux client
1. Add host under `linux_clients:` in `ansible/inventory/hosts.yml`.
2. `ansible-playbook playbooks/deploy-alloy.yml` (safe to re-run — idempotent).

### Add a dashboard
Drop a JSON file in `docker-compose/grafana/dashboards/` and run
`deploy-stack.yml` (syncs files + Grafana picks them up within 10 s via
provisioning). To force-refresh without a full deploy:
`curl -u admin:'<pw>' -X POST http://obs-vm:3000/api/admin/provisioning/dashboards/reload`.

**Datasource references must use the legacy string form** (`"datasource": "Prometheus"`)
or the dashboards won't bind to the provisioned datasource. The canned dashboards
from Pure's repos come with `${DS_PROMETHEUS}` variables — those must be stripped
(see download pattern in `docker-compose/grafana/dashboards/`).

### Edit secrets
```
ansible-vault edit inventory/group_vars/all/vault.yml
```
Don't `git add` the decrypted form. `.vault-password-file` is gitignored; keep it
that way.

---

## Gotchas and lessons learned

### FlashArray native vs. legacy exporter
Pure publishes a container at `quay.io/purestorage/pure-fa-om-exporter` that
*historically* proxied FA API → Prometheus. **Modern Purity (6.4+) exposes
OpenMetrics natively at `/metrics/<namespace>`**, so the exporter container is
dead weight for us. Prometheus scrapes arrays directly with a bearer token from
the vault. If you hit an older FA that doesn't serve `/metrics/...` natively,
add back the `pure-fa-om-exporter` service and relabel scrape jobs through it.

### FlashBlade: pick your poison
Two working paths, both documented in ARCHITECTURE.md:
- **OpenMetrics proxy (`pure-fb-om-exporter`)**: 52 metric families, traditional
  Pure dashboards just work. Exporter is stateless — token comes from Prometheus
  scrape config, not from any config file inside the container. Be careful not
  to expect a `tokens.yaml` — it doesn't use one.
- **Native OTel push**: 30 metric families under a different schema
  (`flashblade_storage_device_*`). Different labels (`device_name`, `fleet_id`,
  `component_name/type`, `entity_name`). Our custom dashboard
  `pure-fb-otel-overview.json` covers the basics; the canned Pure dashboards
  don't work with these names.

Dual-path is supported but **metrics don't unify** — don't try to average
`purefb_*` and `flashblade_*` on one panel.

### OTel TLS trap on FlashBlade
When you configure the OTel export target in the FB GUI, **TLS is on by default
and there is no `--insecure` toggle that ignores cert trust**. If you leave it
on without uploading a trusted cert, the FB's connectivity test passes (raw nc
TCP connect works) but the dummy OTLP delivery fails silently. The OTel
collector logs show zero connection attempts because the TLS client hello dies
before any gRPC traffic reaches it.

**Fix:** either turn off TLS on the export target (lab default) or generate a
proper cert and upload it to the FB. Our collector currently listens in
plaintext on 4317/4318.

### Prometheus reload quirk
The deploy playbook's handler hits `/-/reload` via HTTP. We observed at least
one case where `/-/reload` returned 200 but newly templated scrape jobs didn't
appear in `/api/v1/status/config`. A `docker restart prometheus` fixed it
immediately.

**If jobs don't show up after a deploy:**
```
ssh root@obs-vm 'docker restart prometheus'
```
Cheap and lossless (TSDB is on the persistent volume).

### Storage layout
The 40 GB boot volume has a very small `/var` (~4 GB). Do **not** install Docker
without mounting `/dev/sdb` at `/var/lib/docker` first — Docker will fill `/var`
and the whole VM wedges. `prepare-storage.yml` handles this idempotently.
The 1 TB disks on the Alloy clients are unused; leave them alone or repurpose.

### Alloy config Jinja escaping
`config.alloy` is templated by Ansible's Jinja. Alloy itself uses `{{ ... }}`
syntax for its own templating (e.g. `{{ env("HOSTNAME") }}`). Mixing them leads
to either Ansible rendering Alloy's template (bad) or weird-looking escaped
output. **Template dynamic values in Ansible instead** — we set
`hostname = "{{ inventory_hostname }}"` so Ansible bakes the real hostname at
deploy time, and Alloy sees a static string.

### Grafana dashboards and `${DS_PROMETHEUS}`
Pure's downloaded dashboard JSONs have a `__inputs` block expecting the user to
bind a datasource at import time. For file-based provisioning we have to strip
`__inputs` and substitute `${DS_PROMETHEUS}` with the literal string
`Prometheus` (the datasource name). Script pattern lives in the shell history
of the "import FB dashboards" step — repeat it whenever new dashboards are
pulled from upstream.

### Ansible secret hygiene
`inventory/hosts.yml`, `.vault-password-file`, and `lab-manifest.yml` are all
gitignored. The vault itself (encrypted) **is** committed. If you ever see
decrypted YAML in `git diff`, stop and re-encrypt before committing:
`ansible-vault encrypt inventory/group_vars/all/vault.yml`.

### External network ACLs
From outside the lab: ports 3000 (Grafana) and 9090 (Prometheus) reach through
the perimeter; 3100 (Loki), 9093 (Alertmanager), 4317/4318 (OTel) are blocked
at a network ACL upstream of the lab. Everything works inside the lab. This is
a network config concern, not a stack concern — don't chase it in UFW or Docker.

---

## Troubleshooting cheatsheet

```bash
# Stack state
ssh root@rp-util1 'docker ps --format "table {{.Names}}\t{{.Status}}"'

# Prometheus: what's it scraping?
curl -s http://10.21.101.6:9090/api/v1/targets?state=active \
  | jq '.data.activeTargets[] | {pool: .scrapePool, health, err: .lastError}'

# Prometheus: what config is actually loaded?
ssh root@rp-util1 'curl -s http://localhost:9090/api/v1/status/config' \
  | jq -r '.data.yaml' | grep job_name

# Hard-reload Prometheus (for stubborn reload behavior)
ssh root@rp-util1 'docker restart prometheus'

# Loki from inside the VM (external is firewalled)
ssh root@rp-util1 'curl -s http://localhost:3100/ready; echo
  curl -s http://localhost:3100/loki/api/v1/labels'

# OTel collector traffic volume
ssh root@rp-util1 'docker logs otel-collector 2>&1 | grep Metrics | tail -5'

# FB OpenMetrics proxy sanity
ssh root@rp-util1 'docker exec pure-fb-exporter wget -qO- "http://localhost:9491/metrics/array?endpoint=<fb>" --header "Authorization: Bearer <token>" | head'

# Alloy health on a client
ssh root@rp-util2 'systemctl status alloy'

# Redeploy (idempotent; safe)
cd ansible && ansible-playbook playbooks/deploy-stack.yml
cd ansible && ansible-playbook playbooks/deploy-alloy.yml
```

---

## What we haven't done yet (intentionally)

- Terraform / automated VM provisioning (VMs are hand-built in the lab; the
  `terraform/` directory exists but isn't in the current deploy flow).
- Alertmanager routing beyond defaults (no Slack/SMTP wiring yet).
- Prometheus alert rules beyond the placeholder in
  `docker-compose/prometheus/alerts/pure_storage.yml`.
- TLS anywhere (plaintext inside the lab; add TLS when exposing externally).
- Backup of Prometheus TSDB / Loki chunks / Grafana DB.
- Authentication beyond Grafana's admin user.
