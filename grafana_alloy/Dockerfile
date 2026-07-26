# Grafana Alloy add-on for Home Assistant.
#
# There is deliberately no build.yaml: Home Assistant retired it in the
# 2026-04 builder migration. The base image, labels and build args live here.
# BUILD_ARCH and BUILD_VERSION are supplied by
# home-assistant/builder/actions/build-image; the defaults below only apply
# to a manual `docker build`.
#
# The base must be glibc-based: Grafana's Alloy binaries are dynamically
# linked against glibc, and loki.source.journal needs libsystemd. Both
# libsystemd0 and ca-certificates ship in base-debian already.
ARG BUILD_FROM=ghcr.io/home-assistant/base-debian:bookworm
FROM ${BUILD_FROM}

ARG BUILD_ARCH=amd64
ARG BUILD_VERSION=0.0.0-dev
ARG ALLOY_VERSION=1.18.0

LABEL \
    io.hass.name="Grafana Alloy" \
    io.hass.description="Ship the Home Assistant system journal to Grafana Loki" \
    io.hass.type="addon" \
    io.hass.arch="${BUILD_ARCH}" \
    io.hass.version="${BUILD_VERSION}" \
    org.opencontainers.image.title="Grafana Alloy" \
    org.opencontainers.image.description="Ship the Home Assistant system journal to Grafana Loki" \
    org.opencontainers.image.source="https://github.com/metril/ha-grafana-alloy" \
    org.opencontainers.image.licenses="MIT" \
    org.opencontainers.image.version="${BUILD_VERSION}"

# Download, checksum-verify and install the official Alloy release binary.
# `grep ... | sha256sum -c -` avoids process substitution: RUN uses /bin/sh
# (dash) on Debian, which has no <(...).
RUN set -eu; \
    apt-get update; \
    apt-get install -y --no-install-recommends unzip; \
    case "${BUILD_ARCH}" in \
        amd64)   alloy_arch="amd64" ;; \
        aarch64) alloy_arch="arm64" ;; \
        *) echo "Unsupported BUILD_ARCH: ${BUILD_ARCH}" >&2; exit 1 ;; \
    esac; \
    zip_name="alloy-linux-${alloy_arch}.zip"; \
    base_url="https://github.com/grafana/alloy/releases/download/v${ALLOY_VERSION}"; \
    mkdir -p /tmp/alloy; \
    cd /tmp/alloy; \
    curl -fsSL -O "${base_url}/${zip_name}"; \
    curl -fsSL -O "${base_url}/SHA256SUMS"; \
    grep " ${zip_name}\$" SHA256SUMS | sha256sum -c -; \
    unzip -q "${zip_name}"; \
    install -m 0755 "alloy-linux-${alloy_arch}" /usr/bin/alloy; \
    cd /; \
    rm -rf /tmp/alloy; \
    apt-get purge -y unzip; \
    apt-get autoremove -y; \
    rm -rf /var/lib/apt/lists/*

# Fails the build immediately on a wrong-architecture or corrupt binary,
# rather than at the user's first install.
RUN alloy --version

# Supervisor reads the container's Docker health status directly: it drives the
# add-on's reported state and its watchdog restarts. This replaces the obsolete
# `watchdog:` key in config.yaml.
#
# /-/ready is deliberate. /-/healthy reports component health, which turns
# unhealthy whenever Loki is unreachable - restarting Alloy cannot fix a remote
# outage, and doing so in a loop would discard its journal cursor. /-/ready only
# reports that Alloy itself is up with a loaded configuration.
HEALTHCHECK --interval=30s --timeout=5s --start-period=30s --retries=3 \
    CMD curl -fsS http://127.0.0.1:12345/-/ready || exit 1

COPY rootfs /
