# prometheus_target_register

Adds the freshly-onboarded host as a Prometheus scrape target on the **central monitoring host** without touching `prometheus.yml`. Uses file-based SD under `/etc/prometheus/targets.d/`, one file per `{job, client}` pair, so additions and removals are trivially diffable.

## How it works

All tasks `delegate_to: "{{ groups['monitor_server'][0] }}"` — the central host where Prometheus runs. The role:

1. Ensures `/etc/prometheus/targets.d/` exists.
2. For each applicable exporter (node, dcgm, ibstat), reads the existing JSON file, removes any prior entry for this `host_label`, appends a fresh entry with full labels, and writes back.
3. Hits `POST /-/reload` so Prometheus picks up the new target without a restart.

## Required central-side Prometheus config

`prometheus.yml` on the central host must reference these SD files. Example for the `node` job:

```yaml
- job_name: node
  file_sd_configs:
    - files: ['/etc/prometheus/targets.d/node-*.json']
```

## Variables

| Variable | Default | Description |
|---|---|---|
| `prom_targets_dir` | `/etc/prometheus/targets.d` | SD directory on central host |
| `prom_reload_url` | `http://127.0.0.1:9090/-/reload` | Prometheus reload endpoint |
| `prom_target_files.{node,dcgm,ibstat}` | `<dir>/<job>-<client>.json` | Per-job-per-client file |

## Tags

`prepare`, `node`, `dcgm`, `ibstat`, `reload`
