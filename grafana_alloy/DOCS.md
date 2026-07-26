# Grafana Alloy

Ships the Home Assistant host's `systemd` journal to a Grafana Loki instance using [Grafana Alloy](https://grafana.com/docs/alloy/latest/) v1.18.0.

Because `journald: true` in this add-on's `config.yaml` maps the host's journal read-only into the container, that single journal already contains everything HAOS logs through systemd: Home Assistant Core, Supervisor, every other add-on's container, and host-level services. One add-on, one collector, the whole system's logs.

## Installation

1. In Home Assistant, go to **Settings → Add-ons → Add-on Store**.
2. Click the **⋮** menu (top right) → **Repositories**, and add `https://github.com/metril/ha-grafana-alloy`.
3. Install **Grafana Alloy** from the store.
4. Open the **Configuration** tab, set at least `loki_url`, and start the add-on.

## Configuration

| Option | Required | Default | Description |
|---|---|---|---|
| `loki_url` | Yes | — | Loki push endpoint, e.g. `https://loki.example.com/loki/api/v1/push`. |
| `loki_username` | No | — | Username for HTTP basic auth against Loki. Must be set together with `loki_password` - the add-on refuses to start if only one is set. |
| `loki_password` | No | — | Password for HTTP basic auth against Loki. Never written to the on-disk configuration; see [Web UI](#web-ui) below. |
| `loki_tenant_id` | No | — | Sent as the Loki tenant/org ID (`X-Scope-OrgID`), for multi-tenant Loki deployments. |
| `loki_ca_cert` | No | — | PEM-encoded CA certificate. Set this if Loki (or the reverse proxy in front of it) presents a certificate from a private/internal certificate authority. |
| `loki_server_name` | No | — | Overrides the TLS SNI/server name used to validate the certificate, for when it doesn't match the hostname in `loki_url`. |
| `loki_tls_skip_verify` | No | `false` | Disables TLS certificate verification entirely. Debugging only - never leave this on in production. |
| `instance_label` | No | — | Adds an `instance` external label to every stream. The journal already supplies the HA host's own name as `hostname`, so this is only needed to override that. |
| `journal_max_age` | No | `24h` | How far back into the journal Alloy reads when the add-on starts. |
| `journal_matches` | No | — | A systemd journal match expression restricting which entries are shipped. See [Filtering what gets shipped](#filtering-what-gets-shipped). |
| `log_level` | No | `info` | Alloy's own log level: `debug`, `info`, `warn`, or `error`. This is Alloy's operational logging, not a filter on shipped journal entries. |
| `additional_config` | No | — | Raw Alloy configuration syntax, appended verbatim to the generated `config.alloy`. See [Advanced: additional_config](#advanced-additional_config). |
| `custom_config_path` | No | — | Filename of a complete, hand-written `config.alloy`, used instead of the generated one. See [Advanced: custom_config_path](#advanced-custom_config_path). |

## Example configuration

The common case: Loki reachable through Traefik, terminating TLS with a Let's Encrypt certificate, gated behind HTTP basic auth.

```yaml
loki_url: "https://loki.example.com/loki/api/v1/push"
loki_username: "homeassistant"
loki_password: "a-strong-password"
journal_max_age: "24h"
log_level: "info"
```

Because the certificate is issued by a public CA (Let's Encrypt), no `loki_ca_cert`, `loki_server_name`, or `loki_tls_skip_verify` is needed - Alloy validates it against the system trust store automatically.

## Labels

Every log line shipped by this add-on carries:

- `job="systemd-journal"` - constant, set on the journal source itself.
- `source="journald"` - constant, added by the processing stage.
- `unit` - the systemd unit that logged the entry (from `_SYSTEMD_UNIT`).
- `hostname` - the journal's own idea of the machine's hostname. On HAOS this is the **Home Assistant host's** hostname, the same for every entry - it is not an add-on identifier. This is exactly why `instance` is *not* added unless you set `instance_label`: it would just duplicate `hostname`.
- `syslog_identifier` - the program name as reported to syslog (`SYSLOG_IDENTIFIER`).
- `container_name` - the Docker container that produced the entry, when the entry came from a container. **This is the most useful label on HAOS**: entries from Home Assistant Core show `container_name="homeassistant"`, Supervisor shows `hassio_supervisor`, and every add-on shows `addon_<slug>` (this add-on itself would appear as something like `addon_grafana_alloy`).
- `transport` - how the entry reached the journal (`stdout`, `syslog`, `journal`, `kernel`, ...).
- `level` - the syslog priority keyword (`info`, `warning`, `err`, ...).

Example LogQL queries:

```logql
# Everything Home Assistant Core has logged
{job="systemd-journal", container_name="homeassistant"}

# Every add-on's logs, across the whole system
{job="systemd-journal", container_name=~"addon_.+"}

# Only warnings and errors from the Supervisor
{job="systemd-journal", container_name="hassio_supervisor", level=~"warning|err"}
```

## Filtering what gets shipped

`journal_matches` is passed straight through to Alloy's `loki.source.journal` as its `matches` argument, which uses the same `FIELD=value` match syntax as `journalctl -M` / `sd_journal_add_match`. A couple of useful examples:

```
# Only the Supervisor's own unit
_SYSTEMD_UNIT=hassio-supervisor.service

# Only entries from a specific container
CONTAINER_NAME=homeassistant
```

Combine this with `journal_max_age` to control both *what* and *how much history* is shipped - handy while testing a Loki connection before shipping the full journal.

## Advanced: additional_config

`additional_config` is appended verbatim to the end of the generated `config.alloy`. It can define new Alloy components that plug into the pipeline this add-on already builds, by referencing:

- `loki.process.journal.receiver` - the input of the labeling/processing stage, upstream of `loki.write`.
- `loki.write.loki.receiver` - the input of the configured Loki write client, with `loki_url`, auth, and TLS already applied.

For example, to accept logs pushed from another local process and forward them straight to the already-configured Loki endpoint, reusing its credentials and TLS settings without duplicating them:

```alloy
loki.source.api "extra" {
    http {
        listen_address = "0.0.0.0"
        listen_port    = 9999
    }
    forward_to = [loki.write.loki.receiver]
}
```

If you want this reachable from outside the container, add its port to `ports:` and `ports_description:` in `config.yaml` and rebuild the add-on - by default only 12345 (the web UI) is published.

**A common request this cannot satisfy: tailing `/config/home-assistant.log`.** `additional_config` cannot add a `loki.source.file` block that reads `/config/home-assistant.log`, because the Home Assistant configuration directory is not mapped into this container at all - `map:` in `config.yaml` only maps `addon_config:rw`, this add-on's *own* configuration folder, not `homeassistant_config`. Making Home Assistant's own log file readable would require adding `homeassistant_config:ro` to `map:` and rebuilding the add-on image; it is not something `additional_config` (which only appends configuration text, not container mounts) can work around.

## Advanced: custom_config_path

For full control, place a complete `config.alloy` in this add-on's own configuration folder - `/addon_configs/<hash>_grafana_alloy/` on the host, mapped to `/config` inside the container - and set `custom_config_path` to its filename (relative to that folder). When set, the add-on skips its own config generation entirely and runs your file instead.

`sys.env("LOKI_PASSWORD")` still works in a custom config: if `loki_password` is set, the `alloy` process still has `LOKI_PASSWORD` in its environment regardless of which configuration file it runs.

## Web UI

Alloy has a built-in web UI showing the live component graph, per-component debug info, and counters such as `sent_entries`. **It is not published by default.**

To reach it, open the add-on's **Network** tab, assign a host port to `12345/tcp` (12345 is fine), and save. Clear the field again when you're done.

It is left off by default because the UI has **no authentication** - anything that can reach the port can read it - and its component view shows your `loki_url` and `loki_username` in plain text. Your `loki_password` is *not* shown: Alloy redacts secret-typed values, and the password reaches Alloy through an environment variable rather than the configuration file.

You usually don't need it. Alloy writes every push failure to its own log, so the add-on's **Log** tab already covers everything in the troubleshooting table below. The UI is most useful when logs are silently not arriving and you want to see whether the journal source is producing entries at all.

Enabling the port has no effect on the add-on's health reporting, which uses a Docker healthcheck on the container's internal loopback interface and works whether or not the port is published.

## How the Loki password is handled

`config.alloy` never contains the password. It contains the literal expression `sys.env("LOKI_PASSWORD")`, and the add-on's service script reads the password from Supervisor and puts it in Alloy's process environment just before starting it. So the generated configuration file - the one thing that persists and that you might read, copy, or paste into a bug report - holds no secret.

Two honest caveats:

- `bashio`, the standard Home Assistant add-on helper library, caches the add-on's options under `/tmp/.bashio` inside the container while reading them, and the version bundled in the base image writes that cache world-readable. The add-on deletes this cache immediately before starting Alloy, so it does not persist for the container's lifetime, but it does exist briefly during startup. This affects every add-on that reads a password option, not just this one.
- The password is, necessarily, in the `alloy` process's environment for as long as it runs, and is therefore readable via `/proc` by anything with root inside that container.

Neither is a path to the secret from outside the container, and nothing writes it to a mapped or persistent volume.

## Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| `status=401` in the Alloy log | Wrong HTTP basic auth credentials | Check `loki_username` / `loki_password` against what Traefik/Loki expects |
| `x509: certificate signed by unknown authority` | Loki (or the proxy in front of it) presents a certificate from a private/internal CA | Set `loki_ca_cert` to that CA's PEM certificate |
| `x509: certificate is valid for X, not Y` | The certificate's SAN/CN doesn't match the hostname in `loki_url` (e.g. connecting by IP, or via an internal name while the cert covers only the public one) | Set `loki_server_name` to the hostname the certificate is actually issued for |
| Add-on won't start; fatal log about username/password | Only one of `loki_username` / `loki_password` is set | Set both to enable basic auth, or clear both to disable it |
| No logs arriving; `sent_entries` stuck at zero | The journal source isn't producing entries, or `journal_matches` is filtering everything out | Open the web UI and inspect the `loki.source.journal` component's debug info; double-check `journal_matches` syntax |
