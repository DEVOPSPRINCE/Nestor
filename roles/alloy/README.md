# alloy

Installs Grafana Alloy and configures it to ship systemd journal + `/var/log/*.log` to the central Loki via HTTP basic auth.

## Variables

| Variable | Default | Description |
|---|---|---|
| `central_loki_url` | (set in `group_vars/all.yml`) | Loki push endpoint via central nginx |
| `alloy_basic_auth_user` | `alloy` | nginx basic-auth user |
| `alloy_basic_auth_password` | (in group_vars / ansible-vault) | nginx basic-auth password |
| `alloy_ship_journal` | `true` | Ship systemd journal |
| `alloy_ship_varlogs` | `true` | Ship `/var/log/*.log` |
| `alloy_varlog_pattern` | `/var/log/*.log` | Glob for file tailing |

## Labels attached to every log stream

`client`, `env`, `host` from inventory vars; `job=systemd-journal` or `job=varlog`.

## Tags

`repo`, `install`, `config`, `service`
