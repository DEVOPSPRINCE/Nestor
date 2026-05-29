# Ansible — Onboard Client Nodes to Central Monitoring

One-command onboarding for new client servers into the central monitoring stack.

## What it does

For each target node:

| # | Action | Conditional |
|---|---|---|
| 1 | Installs `node_exporter` (host OS metrics) | Always |
| 2 | Installs **Grafana Alloy** (ships logs to central Loki with basic-auth) | Always |
| 3 | Installs **NVIDIA dcgm-exporter** (per-GPU metrics) | Only if `has_gpu: true` |
| 4 | Installs **ibstat-exporter** (InfiniBand) | Only if `has_infiniband: true` (training customers) |
| 5 | Configures UFW: SSH open, scrape ports restricted to central server IP | Always |

On the central server (delegated):

| # | Action |
|---|---|
| 6 | Adds the new host to Prometheus scrape config (`file_sd`) |
| 7 | Triggers Prometheus reload |
| 8 | Creates per-client Grafana folder + dashboard directory |

## Quick start

```bash
cd /Users/prince/server-audit/ansible

# 1. Copy and customize the inventory template
cp inventory.yml.example inventory.yml
vim inventory.yml   # add your new client's host details

# 2. Onboard a single host
ansible-playbook -i inventory.yml onboard.yml -l acme-gpu-01

# 3. Onboard an entire client group at once
ansible-playbook -i inventory.yml onboard.yml -l gpu_nodes

# 4. Dry-run first (recommended)
ansible-playbook -i inventory.yml onboard.yml -l acme-gpu-01 --check --diff
```

## Required per-host variables

| Variable | Example | Purpose |
|---|---|---|
| `host_label` | `acme-gpu-01` | Becomes `instance` label everywhere |
| `host_ip` | `10.0.10.21` | Surfaced in Slack alerts for quick SSH |
| `client_name` | `acme-corp` | MSP tenant identifier → Grafana folder + Slack channel |
| `env` | `production` | Environment label |
| `role` | `compute` | Functional role |
| `node_type` | `gpu` or `cpu` | Hardware class |
| `has_gpu` | `true` / `false` | Controls dcgm-exporter install |
| `has_infiniband` | `true` / `false` | Controls IB exporter (training customers) |

## Workflow when new client is acquired (e.g. Monday)

1. **Get credentials** from client (SSH key, IP, hostname).
2. **Add to `inventory.yml`** with the host vars above.
3. **Run** `ansible-playbook -i inventory.yml onboard.yml -l <new-hostname>`.
4. **Verify** — open Grafana → Dashboards → `<ClientName>` folder. The new host will appear in panels within 30 seconds.

## Verification after run

The playbook automatically:
- Confirms Prometheus reload succeeded
- Returns non-zero on any failure
- Re-running is idempotent (safe to repeat)

To verify manually:

```bash
# Is the new host being scraped?
ssh -i ~/.ssh/netsor -p 10345 user@80.188.223.202 \
    "curl -s -u admin:Netsor@2026 'http://127.0.0.1:9090/prometheus/api/v1/targets' | jq '.data.activeTargets[] | select(.labels.instance==\"<host_label>\")'"
```

## Files

```
ansible/
├── README.md                  ← this file
├── inventory.yml.example      ← copy → inventory.yml and edit per node
├── group_vars/
│   └── all.yml                ← central server IP, basic-auth creds, version pins
└── onboard.yml                ← the single playbook (does everything)
```

## Notes

- **The central monitoring server itself is NOT redeployed by this playbook.** It exists once. New nodes are onboarded *into* it.
- **No Docker.** Everything is bare-metal `systemd`, matching the central stack.
- **Idempotent.** Re-running against an already-onboarded host updates configs in place.
- **MSP-native.** Each new client appears as a separate Grafana folder; alerts route to that client's Slack channel automatically (when added to Alertmanager routing).
- For real H100 nodes, `dcgm-exporter` requires the NVIDIA driver to be already installed on the node. If `nvidia-smi` is not found, the playbook skips dcgm-exporter and logs a notice.
