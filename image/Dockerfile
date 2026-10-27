# syntax=docker/dockerfile:1

# Pinned by digest so a rebuild without repo changes gives the same base.
FROM ghcr.io/getpaseo/paseo:latest@sha256:fde5ff1192933bf4feebaa19ef072ff8b96d9c1078978cc0ee75c5541ff16581

ARG TARGETARCH
ARG MISE_VERSION=v2026.10.20

# Stay root: the upstream entrypoint drops to the `paseo` user itself.
USER root

COPY image/apt.txt /tmp/apt.txt
RUN set -eux; \
    apt-get update; \
    grep -vE '^\s*(#|$)' /tmp/apt.txt | xargs apt-get install -y --no-install-recommends; \
    rm -rf /var/lib/apt/lists/* /tmp/apt.txt

RUN set -eux; \
    case "$TARGETARCH" in amd64) arch=x64 ;; arm64) arch=arm64 ;; *) exit 1 ;; esac; \
    curl -fsSL -o /usr/local/bin/mise \
      "https://github.com/jdx/mise/releases/download/${MISE_VERSION}/mise-${MISE_VERSION}-linux-${arch}"; \
    chmod +x /usr/local/bin/mise

# System-wide toolchain. /opt/mise is handed to `paseo` so agents can still
# `mise use` extra tool versions inside their projects.
ENV MISE_DATA_DIR=/opt/mise \
    MISE_GLOBAL_CONFIG_FILE=/etc/mise/config.toml \
    MISE_YES=1 \
    PATH=/usr/local/lib/devbox/bin:/opt/mise/shims:/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
COPY image/mise.toml /etc/mise/config.toml
# Changed by scripts/rebuild on every run so "latest" is re-resolved instead of
# reusing the cached layer.
ARG TOOLS_CACHE_BUST
RUN set -eux; \
    mise install; \
    # mise installs npm tools without lifecycle scripts, but claude-code needs
    # its postinstall to put the native binary in place.
    cd "$(mise where npm:@anthropic-ai/claude-code)/lib/node_modules/@anthropic-ai/claude-code"; \
    node install.cjs; \
    cd /; \
    mise reshim; \
    rm -rf /root/.cache; \
    chown -R paseo:paseo /opt/mise

COPY config/ /etc/devbox/
COPY image/rootfs/ /
RUN chmod +x /usr/local/bin/devbox-* /usr/local/lib/devbox/bin/*

ENTRYPOINT ["/usr/bin/tini", "--", "/usr/local/bin/devbox-entrypoint"]
