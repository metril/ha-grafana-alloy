<p align="center"><img src="https://raw.githubusercontent.com/metril/ha-grafana-alloy/main/brands/logo.png" alt="Grafana Alloy" width="320"></p>

# Grafana Alloy for Home Assistant

Ship the Home Assistant systemd journal to a Grafana Loki instance using Grafana Alloy.

[![Open your Home Assistant instance and add this repository to your Supervisor add-on store.](https://my.home-assistant.io/badges/supervisor_add_addon_repository.svg)](https://my.home-assistant.io/redirect/supervisor_add_addon_repository/?repository_url=https%3A%2F%2Fgithub.com%2Fmetril%2Fha-grafana-alloy)

## Features

- Ships every journald entry visible to Home Assistant OS - HA Core, Supervisor, every add-on container, and host services - to Grafana Loki
- TLS by default, with an optional private CA, SNI override, or skip-verify for debugging
- Optional HTTP basic auth; the password is kept out of the generated Alloy configuration file
- Filter what gets shipped with systemd journal match expressions and a configurable retention window
- `additional_config` and `custom_config_path` escape hatches for anything the options UI doesn't cover
- Alloy's own web UI (component graph, live debugging) exposed on port 12345

## Installation

This is a Home Assistant **add-on**, not a HACS integration - HACS cannot install add-ons, so it doesn't apply here.

1. In Home Assistant, go to **Settings → Add-ons → Add-on Store**.
2. Click the **⋮** menu in the top right corner and choose **Repositories**.
3. Paste `https://github.com/metril/ha-grafana-alloy` and click **Add**.
4. Find **Grafana Alloy** in the store and click **Install**.

Or use the badge above to open the repository dialog directly in your own Home Assistant instance.

## Configuration

At minimum, set `loki_url` to your Loki push endpoint (e.g. `https://loki.example.com/loki/api/v1/push`). Basic auth, TLS options, journal filtering, and advanced overrides are all optional. See [`grafana_alloy/DOCS.md`](grafana_alloy/DOCS.md) for the full configuration reference, an example configuration, the label reference, and a troubleshooting guide - the same content is also shown as the add-on's **Documentation** tab in Supervisor.

## Requirements

- Home Assistant OS or Supervised, with Supervisor
- `amd64` or `aarch64` host - Grafana publishes no 32-bit ARM Alloy binaries, so `armhf`/`armv7` are not supported
- A reachable Loki push endpoint (`/loki/api/v1/push`)
