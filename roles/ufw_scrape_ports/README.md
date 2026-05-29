# ufw_scrape_ports

Configures host-local UFW so that **only the central monitoring host** can reach exporter ports (`9100`, `9400`, `9315`). SSH on `22/tcp` is opened first to avoid lockout, defaults are set to `deny incoming` + `allow outgoing`, and UFW is enabled if currently inactive.

## Why

The provider does not offer infrastructure-level firewalling — only port forwarding. UFW gives us per-host network isolation so Prometheus scrape endpoints are not reachable from the open internet.

## Variables

| Variable | Default | Description |
|---|---|---|
| `ufw_scrape_source_ip` | `{{ central_host_ip }}` | Central monitoring IP allowed to scrape |
| `ufw_scrape_ports` | node_exporter + conditional DCGM + IB | List of `{port, proto, name, when}` |
| `ufw_enable_if_inactive` | `true` | Bring UFW up if it's not already active |

## Requirements

`community.general` collection (for the `ufw` module).

## Tags

`install`, `rules`, `enable`
