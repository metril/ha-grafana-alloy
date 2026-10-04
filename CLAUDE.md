# CLAUDE.md — ha-grafana-alloy

## Project Overview

Home Assistant add-on that runs Grafana Alloy to ship the host's systemd journal - Home Assistant Core, Supervisor, every add-on container, and host services - to a Grafana Loki instance, typically reached through Traefik with TLS and HTTP basic auth.

GitHub: https://github.com/metril/ha-grafana-alloy

## Architecture

```
systemd journal (read-only, via journald: true)
        |
        v
loki.source.journal "systemd"
        |
        v
loki.relabel "journal"   (unit, hostname, syslog_identifier, container_name, transport, level)
        |
        v
loki.process "journal"   (adds static label source="journald")
        |
        v
loki.write "loki"   --HTTPS (basic auth, optional custom CA / SNI)-->  Traefik  -->  Loki
```

## Key Files

| File | Purpose |
|------|---------|
| `grafana_alloy/config.yaml` | Add-on manifest: options schema, arch, ports, map |
| `grafana_alloy/Dockerfile` | Base image, Alloy download + checksum verify, build labels |
| `grafana_alloy/rootfs/etc/s6-overlay/s6-rc.d/init-alloy/run` | Oneshot: renders `/etc/alloy/config.alloy` from add-on options |
| `grafana_alloy/rootfs/etc/s6-overlay/s6-rc.d/alloy/run` | Longrun: reads `loki_password` into its own env, execs `alloy run` |
| `grafana_alloy/DOCS.md` | Full configuration reference, shown as Supervisor's Documentation tab |
| `grafana_alloy/translations/en.yaml` | Options-UI labels/descriptions |

## Critical Design Decisions

- **The Loki password never lands in `config.alloy`.** `init-alloy` only checks whether `loki_password` is set - it never reads its value - and emits the literal text `sys.env("LOKI_PASSWORD")`. The `alloy` longrun reads the actual secret itself and exports it into its own process environment before `exec`ing Alloy. Be precise about the claim: bashio caches all options (password included) under `/tmp/.bashio` while reading them, and the bundled bashio writes that cache mode 0644, so `alloy/run` calls `bashio::cache.flush_all` before exec. The secret is unavoidably in the Alloy process environment. Nothing writes it to a mapped or persistent volume. Do not restate this as "never written to disk" - that is not what was verified.
- **journald-only by design.** No Home Assistant `/config` map, no `docker_api` access. The add-on's only inputs are the journal (via `journald: true`) and its own options; this keeps the attack surface and permission footprint minimal.
- **`build.yaml` is deliberately absent.** Home Assistant retired it in the 2026-04 builder migration. The base image, OCI labels, and build args live directly in the `Dockerfile` instead.
- **`home-assistant/builder@master` is NOT used.** It is deprecated and its root `action.yaml` no longer exists. CI uses the `home-assistant/builder/actions/*` composite actions, pinned to `2026.09.0`.
- **Debian base, not Alpine.** The official Alloy binary is dynamically linked against glibc, and `loki.source.journal` needs `libsystemd`. `ghcr.io/home-assistant/base-debian:trixie` ships `ca-certificates`; `libsystemd0` is expected via apt's own dependency on it and the Dockerfile asserts `libsystemd.so.0` at build time because Alloy dlopen()s it (so `alloy --version` proves nothing about it).
- **`instance` external label only emitted when `instance_label` is set.** The journal already supplies the Home Assistant host's own name as the `hostname` label, so adding `instance` unconditionally would just duplicate it for the common case.
- **The Alloy web UI is unpublished by default** (`ports: 12345/tcp: null`), matching the `ha-ipmi-control` convention. It serves no authentication and discloses the Loki URL and username in its component view; the password is redacted (verified - the secret string appears nowhere in `/api/v0/web/components/loki.write.loki`). It is a troubleshooting convenience only: Alloy logs every push failure to stdout, so the add-on's Log tab already covers the documented failure modes.
- **Health is a Docker `HEALTHCHECK`, not the `watchdog:` key.** The add-on linter rejects `watchdog:` as obsolete, and Supervisor reads the container's Docker health status natively (it drives the reported add-on state and restarts). The probe hits `/-/ready`, deliberately not `/-/healthy`: the latter turns unhealthy whenever Loki is unreachable, and restarting Alloy cannot fix a remote outage - it would just loop and discard the journal cursor.

## Add-on Details

- Base image: `ghcr.io/home-assistant/base-debian:trixie`
- Alloy pinned to v1.20.1, downloaded from the GitHub release and SHA256-verified against that release's `SHA256SUMS`
- s6-overlay v3 `s6-rc.d`: `init-alloy` (oneshot, renders config) → `alloy` (longrun, `alloy run`), wired via `dependencies.d`
- `init: false` is mandatory - s6-overlay v3 refuses to start otherwise
- Supported architectures: `amd64`, `aarch64` only

## CI/CD

- `.github/workflows/lint.yml` - runs the Home Assistant add-on linter, an executable-bit check, and an amd64 image build plus smoke test (base image, Alloy binary, libsystemd, s6/bashio, script syntax) on every push and pull request
- `.github/workflows/release.yml` - runs on pushes to main touching `grafana_alloy/config.yaml`, on `v*` tags, and on manual dispatch: reads `version:`, skips if release `v<version>` already exists, and on a tag push asserts tag == version; builds both architectures, publishes the multi-arch manifest to ghcr.io, then creates the tag + GitHub Release via softprops. Tags pushed with `GITHUB_TOKEN` do not trigger workflows, so tag-only triggering cannot be automated.
- Single `v*` tag scheme - there is no `addon-v*` scheme, since this repo has no companion integration

## Common Gotchas

1. **`loki.source.journal` must be given an explicit `path`.** Without one, `sd_journal` opens the *local* journal, which it identifies by matching `/etc/machine-id` against the journal directory's name. Add-on containers have no `/etc/machine-id`, so nothing is ever read - and the component still reports `healthy` with "journal tailer is running", so this fails completely silently. `init-alloy` detects `/var/log/journal`, falling back to `/run/log/journal`, and writes it into the config. Verified: 0 lines read without `path`, 1064 with it.
2. s6 `run`/`up` scripts must be mode `100755` in git or the add-on silently fails to start - not visible in a normal diff. The lint workflow guards this; check it locally before pushing if scripts change.
3. `config.yaml`'s `version:` must equal the published image tag exactly, with no `v` prefix. Bumping `version:` on main is the release trigger; config.yaml edits without a bump are a no-op skip. Never hand-create the GitHub Release first or the run will skip.
4. ghcr.io package visibility. The v1.0.0 publish came out **public** with no manual step - verified by fetching the index, the amd64 child manifest, and its config blob from `ghcr.io/metril/ha-grafana-alloy` using an anonymous pull token (all HTTP 200). Older guidance says `GITHUB_TOKEN` pushes land private; that did not happen here. If Supervisor ever fails to pull with an opaque error, re-check with an anonymous token before assuming anything else is wrong.
5. `--server.http.listen-addr=0.0.0.0:12345` is needed **only** for the web UI, not for the healthcheck. Alloy defaults to `127.0.0.1:12345`; Docker port publishing DNATs to the container's IP, so a loopback bind makes the published port unreachable. The `HEALTHCHECK` runs inside the container's own network namespace and passes on loopback regardless - verified by building a `127.0.0.1`-only variant, which reported `healthy` while the published port refused connections.
6. The add-on linter errors on any `config.yaml` key set to its schema default, so `boot: auto` and `host_network: false` are deliberately omitted rather than written out explicitly.
7. When upgrading Alloy, bump `ALLOY_VERSION` in the `Dockerfile` and `version` in `config.yaml` together. Merging that to main publishes the release automatically.

## Git Conventions

- No AI attribution in commits
- GitHub user: metril
- Email: 1517921+metril@users.noreply.github.com
