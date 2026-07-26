# Changelog

## 1.0.0

- Initial release. Ships the systemd journal to Grafana Loki using Grafana Alloy 1.18.0.
- TLS by default, with an optional custom CA certificate, SNI/server name override, and a skip-verify escape hatch for debugging.
- Optional HTTP basic auth; the password is kept out of the generated configuration file and supplied to Alloy through its process environment instead.
- Journal filtering via `journal_matches`, with a configurable `journal_max_age` retention window.
- `additional_config` and `custom_config_path` escape hatches for configuration the options UI doesn't cover.
- Alloy's web UI (component graph, live debugging) exposed on port 12345.
