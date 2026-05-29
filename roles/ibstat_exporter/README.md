# ibstat_exporter

Installs **ibstat_exporter** as a hardened systemd unit, exposing InfiniBand port + counter metrics on TCP `:9315`. Apply this role only to nodes with InfiniBand HCAs (set `has_infiniband: true` in inventory).

## Prerequisites

`/sys/class/infiniband` must be present (i.e. an IB driver/HCA on the host). The role aborts cleanly if not, unless `ibstat_exporter_require_ib: false`.

## Variables

| Variable | Default | Description |
|---|---|---|
| `ibstat_exporter_version` | `1.7.0` | Upstream release |
| `ibstat_exporter_port` | `9315` | TCP listen port |
| `ibstat_exporter_require_ib` | `true` | Fail loudly if no IB device |

## Tags

`preflight`, `install`, `service`, `verify`
