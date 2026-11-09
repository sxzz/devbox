# syntax=docker/dockerfile:1

# Tracks latest: scripts/rebuild builds with --pull, and scripts/rollback
# returns to an older image if an upgrade breaks something.
FROM ghcr.io/getpaseo/paseo:latest

ARG TARGETARCH

# Stay root: the upstream entrypoint drops to the `paseo` user itself.
USER root

COPY image/apt.txt /tmp/apt.txt
RUN set -eux; \
    apt-get update; \
    grep -vE '^\s*(#|$)' /tmp/apt.txt | xargs apt-get install -y --no-install-recommends; \
    rm -rf /var/lib/apt/lists/* /tmp/apt.txt

# Changed by scripts/rebuild on every run so "latest" (mise itself and the
# tools below) is re-resolved instead of reusing the cached layers.
ARG TOOLS_CACHE_BUST
# GitHub release assets carry the version in their name, so take the
# unversioned latest build from mise's own site.
RUN set -eux; \
    case "$TARGETARCH" in amd64) arch=x64 ;; arm64) arch=arm64 ;; *) exit 1 ;; esac; \
    curl -fsSL -o /usr/local/bin/mise "https://mise.jdx.dev/mise-latest-linux-${arch}"; \
    chmod +x /usr/local/bin/mise

# System-wide toolchain. /opt/mise is handed to `paseo` so agents can still
# `mise use` extra tool versions inside their projects.
ENV MISE_DATA_DIR=/opt/mise \
    MISE_GLOBAL_CONFIG_FILE=/etc/mise/config.toml \
    MISE_YES=1 \
    PATH=/usr/local/lib/devbox/bin:/opt/mise/shims:/opt/cargo/bin:/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
COPY image/mise.toml /etc/mise/config.toml
# The cache bust reruns this step every time, so keep downloads in BuildKit
# cache mounts: a tool whose latest version hasn't changed is installed from
# the local copy. Only mise's download cache is shared, not the rest of
# ~/.cache/mise, whose cached version lists could make "latest" stale.
RUN --mount=type=cache,target=/root/.cache/mise/downloads-cas \
    --mount=type=cache,target=/root/.npm \
    set -eux; \
    mise install; \
    # mise installs npm tools without lifecycle scripts, but claude-code needs
    # its postinstall to put the native binary in place.
    cd "$(mise where npm:@anthropic-ai/claude-code)/lib/node_modules/@anthropic-ai/claude-code"; \
    node install.cjs; \
    cd /; \
    mise reshim; \
    # rm -rf /root/.cache would empty the mounted cache and then fail on the
    # mount point; clear everything around it instead.
    for p in /root/.cache/* /root/.cache/mise/*; do \
      case "$p" in /root/.cache/mise | /root/.cache/mise/downloads-cas) ;; *) rm -rf "$p" ;; esac; \
    done; \
    chown -R paseo:paseo /opt/mise

# Rust via rustup, latest stable as the default toolchain. Owned by `paseo` so
# agents can add toolchains, targets and components, and rust-toolchain.toml
# files work as usual.
ENV RUSTUP_HOME=/opt/rustup \
    CARGO_HOME=/opt/cargo
# rustup deletes its downloads after installing, so cache the whole install
# instead: when stable hasn't moved, the cache bust above costs only a manifest
# check and a local copy. rustup and the toolchain are relocatable.
RUN --mount=type=cache,target=/var/cache/devbox-rust,sharing=locked \
    set -eux; \
    export RUSTUP_HOME=/var/cache/devbox-rust/rustup CARGO_HOME=/var/cache/devbox-rust/cargo; \
    if [ ! -x "$CARGO_HOME/bin/rustup" ]; then \
      curl --proto '=https' --tlsv1.2 -fsSL https://sh.rustup.rs | \
        sh -s -- -y --no-modify-path --profile minimal --default-toolchain none; \
    fi; \
    rustup="$CARGO_HOME/bin/rustup"; \
    "$rustup" self update; \
    "$rustup" set profile minimal; \
    # Updates the toolchain in place when it is already in the cache.
    "$rustup" toolchain install stable --component clippy,rustfmt; \
    "$rustup" default stable; \
    mkdir -p /opt/rustup /opt/cargo; \
    cp -a "$RUSTUP_HOME/." /opt/rustup/; \
    cp -a "$CARGO_HOME/bin" /opt/cargo/; \
    chown -R paseo:paseo /opt/rustup /opt/cargo

# Keep /workspace on the same mount as home: pnpm picks one store per mount and
# hardlinks can't cross mounts, so a separate /workspace mount duplicates stores.
RUN rm -rf /workspace && ln -s /home/paseo/workspace /workspace

COPY config/ /etc/devbox/
COPY image/rootfs/ /
RUN chmod +x /usr/local/bin/devbox-* /usr/local/lib/devbox/bin/*

ENTRYPOINT ["/usr/bin/tini", "--", "/usr/local/bin/devbox-entrypoint"]
