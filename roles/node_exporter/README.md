# node_exporter

Installs and runs Prometheus `node_exporter` as a systemd unit. Exposes host OS metrics (CPU, memory, disk, network, processes, systemd, filesystems) on TCP `:9100`.

## Role variables

See `defaults/main.yml`. Common overrides:

| Variable | Default | Description |
|---|---|---|
| `node_exporter_version` | `1.11.1` | Upstream release to install |
| `node_exporter_port` | `9100` | TCP listen port |
| `node_exporter_listen` | `0.0.0.0` | Listen address |
| `node_exporter_collectors` | `--collector.systemd --collector.processes` | Extra collectors to enable |

## Tags

`install`, `config`, `service`, `verify` — use `--tags` to run a subset.

## Example

```yaml
- hosts: gpu_nodes
  roles:
    - role: node_exporter
```
