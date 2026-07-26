# Changelog

## 1.0.2

- Dropped the experimental flag; the add-on now shows as stable in the store.

## 1.0.1

- The Alloy web UI is no longer published by default. It has no authentication
  and its component view discloses the Loki URL and username, so port 12345 is
  now unassigned; enable it from the add-on's Network tab when troubleshooting.
  This does not affect health reporting, which uses the container's internal
  loopback interface.

## 1.0.0

- Initial release. Ships the systemd journal to Grafana Loki using Grafana Alloy 1.18.0.
- TLS by default, with an optional custom CA certificate, SNI/server name override, and a skip-verify escape hatch for debugging.
- Optional HTTP basic auth; the password is kept out of the generated configuration file and supplied to Alloy through its process environment instead.
- Journal filtering via `journal_matches`, with a configurable `journal_max_age` retention window.
- `additional_config` and `custom_config_path` escape hatches for configuration the options UI doesn't cover.
- Alloy's web UI (component graph, live debugging) available on port 12345.
