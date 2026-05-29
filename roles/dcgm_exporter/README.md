# dcgm_exporter

Installs NVIDIA's official **DCGM exporter** as a systemd unit, exposing per-GPU metrics on TCP `:9400` (temperature, utilization, ECC errors, XID errors, NVLink stats, etc.).

## Prerequisites

The target host **must already have the NVIDIA driver installed** (`nvidia-smi` must work). The role aborts cleanly if not.

## Variables

| Variable | Default | Description |
|---|---|---|
| `dcgm_exporter_version` | `3.3.5-3.4.1` | Upstream release |
| `dcgm_exporter_port` | `9400` | TCP listen port |
| `dcgm_exporter_require_nvidia` | `true` | Fail loudly if no NVIDIA driver |

## Tags

`preflight`, `install`, `service`, `verify`
