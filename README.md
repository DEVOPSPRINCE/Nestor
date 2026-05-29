# Nestor — Client Node Onboarding

Ansible automation for onboarding new client nodes into the Netsor centralized monitoring fleet. One playbook, one inventory entry, one command — and the node is scraped, log-shipped, firewalled, and registered.

The **central monitoring host is fixed** (Prometheus + Alertmanager + Loki + Grafana + nginx + msteams shim, all bare-metal systemd on Ubuntu 24.04 LTS). This repository does **not** redeploy it; it only onboards client nodes against it.

---

## Repository layout

```
ansible/
├── ansible.cfg              # inventory path, roles_path, fact cache
├── inventory.yml.example    # template inventory (copy → inventory.yml)
├── group_vars/all.yml       # central host IP, Loki URL, version pins
├── onboard.yml              # top-level orchestrating playbook
└── roles/
    ├── node_exporter/             # CPU/mem/disk/net metrics  (every host)
    ├── alloy/                     # log shipper to central Loki (every host)
    ├── dcgm_exporter/             # NVIDIA GPU metrics       (GPU hosts)
    ├── ibstat_exporter/           # InfiniBand counters       (HPC hosts)
    ├── ufw_scrape_ports/          # central-IP-only allowlist (every host)
    └── prometheus_target_register/# adds host to central SD   (every host)
```

Every role is Galaxy-standard: `meta/`, `defaults/`, `tasks/`, `handlers/`, `templates/`, `README.md`. Defaults are safe; overrides live in `group_vars/` or `host_vars/`.

---

## Prerequisites

| Where | What |
|---|---|
| Control machine | Ansible ≥ 2.14, `community.general` collection (`ansible-galaxy collection install community.general`) |
| Target nodes | Ubuntu 22.04 (jammy) or 24.04 (noble), root-equivalent SSH access |
| Central host | Already deployed (Prometheus + Loki + nginx auth-proxy reachable) |

---

## Quickstart

```bash
# 1. Clone
git clone git@github.com:<org>/Nestor.git && cd Nestor/ansible

# 2. Make your inventory
cp inventory.yml.example inventory.yml
$EDITOR inventory.yml

# 3. Vault the Alloy push password (one-time)
ansible-vault encrypt_string 'Alloy@push2026' --name alloy_basic_auth_password

# 4. Smoke test
ansible -i inventory.yml all -m ping

# 5. Onboard one host
ansible-playbook -i inventory.yml onboard.yml --limit gpu-node-01

# 6. Onboard everything
ansible-playbook -i inventory.yml onboard.yml
```

After the playbook completes:

* Prometheus auto-discovers the host via `/etc/prometheus/targets.d/*.json` (no `prometheus.yml` edits, no restart — only a hot reload).
* Grafana dashboards already filter by `client` / `host` labels, so the new node shows up immediately.
* Logs land in Loki under `client=<name>, host=<host>, job=systemd-journal|varlog`.

---

## Inventory schema

Every host needs these vars (see `inventory.yml.example`):

| Var | Example | Used by |
|---|---|---|
| `ansible_host` | `185.165.50.50` | SSH |
| `host_label` | `gpu-node-01` | All exporter labels |
| `host_ip` | `185.165.50.50` | UFW source-deny, scrape target |
| `client_name` | `daytona` | Multi-tenant routing, Grafana filter |
| `env` | `prod` / `staging` | Alert routing |
| `role` | `inference` / `training` | Dashboards |
| `node_type` | `gpu` / `cpu` | Dashboards |
| `has_gpu` | `true`/`false` | Conditional DCGM role |
| `has_infiniband` | `true`/`false` | Conditional ibstat role |

---

## Tag-based partial runs

```bash
# Just push a new node_exporter version
ansible-playbook -i inventory.yml onboard.yml --tags node_exporter

# Just re-register Prometheus targets (e.g. after host_ip change)
ansible-playbook -i inventory.yml onboard.yml --tags register

# Only firewall hardening
ansible-playbook -i inventory.yml onboard.yml --tags ufw
```

---

## Security model

* `ufw_scrape_ports` opens `9100/9400/9315` **only from the central monitor IP**. Default policy is `deny incoming` (SSH is added before `ufw enable` to avoid lockout).
* Alloy pushes logs over HTTPS to the central nginx via HTTP basic auth (`alloy_basic_auth_user` / `alloy_basic_auth_password` from `group_vars/all.yml`, vault-encrypted).
* Every systemd unit ships with hardening (`NoNewPrivileges`, `ProtectSystem=strict`, `PrivateTmp`, capability dropping, etc.).
* No secrets are committed; `.gitignore` excludes `inventory.yml`, `.vault_pass`, `*.retry`.

---

## Versions (pinned in `group_vars/all.yml`)

| Component | Version |
|---|---|
| node_exporter | 1.11.1 |
| Grafana Alloy | 1.16.1 |
| DCGM exporter | 3.3.5-3.4.1 |
| ibstat_exporter | 1.7.0 |

All binaries are pulled from the upstream GitHub releases over HTTPS — no third-party mirrors.

---

## Idempotency

Every role is safe to re-run. Target registration overwrites only the entry for the current `host_label`, leaving other clients' targets untouched. UFW rules use ufw's native idempotency. Exporter binaries are version-pinned, so re-runs are no-ops unless you bump the pin.
