# Alpine base image. Pinned for reproducible builds; Renovate keeps it current.
FROM alpine:3.25.3@sha256:fd791d74b68913cbb027c6546007b3f0d3bc45125f797758156952bc2d6daf40

# Set working directory
WORKDIR /app

# Create a dedicated unprivileged runtime user and writable directories.
RUN addgroup -S bitwarden-sync && adduser -S -D -H -G bitwarden-sync -u 1000 bitwarden-sync && \
    mkdir -p /app/data /app/backups /app/.npm-global /var/spool/cron/crontabs /var/log && \
    chown -R bitwarden-sync:bitwarden-sync /app /var/spool/cron /var/log

# Install required packages. Build tooling is intentionally omitted to reduce
# the image footprint and attack surface.
RUN apk --no-cache add \
    bash \
    binutils \
    ca-certificates \
    coreutils \
    curl \
    findutils \
    grep \
    jq \
    ncurses \
    nodejs \
    npm \
    openssl \
    python3 \
    tar \
    unzip \
    util-linux \
    uuidgen \
    wget

# Suppress Node.js deprecation warnings and keep npm installs in a writable
# location for the non-root runtime user.
ENV NODE_OPTIONS="--no-deprecation" \
    NPM_CONFIG_PREFIX=/app/.npm-global \
    PATH=/app/.npm-global/bin:${PATH}

# Bitwarden CLI versions. Override at build time with --build-arg, or at runtime
# by setting the matching env var (the entrypoint reconciles a concrete version
# that differs from the baked-in one). Keep comments on their own lines: Docker
# does NOT strip an inline "# ..." after an ARG value.
# Both are pinned to a known-good version (Renovate proposes bumps, gated by the
# CLI compatibility test) — 2026.x has broken both Vaultwarden login (#50) and
# Bitwarden Cloud import.
# Source / Vaultwarden compatibility (issue #50):
# renovate: datasource=npm depName=@bitwarden/cli
ARG BW_CLI_OLD_VERSION=2026.6.0
# Destination / Bitwarden cloud:
# renovate: datasource=npm depName=@bitwarden/cli
ARG BW_CLI_NEW_VERSION=2026.6.0
# Bridge the build args to env vars so they can be overridden at runtime too.
ENV BW_CLI_OLD_VERSION=${BW_CLI_OLD_VERSION} \
    BW_CLI_NEW_VERSION=${BW_CLI_NEW_VERSION}

# Install both versions of Bitwarden CLI in separate locations.
# Install the pinned "old" CLI into an isolated path for the source, then the
# "new" CLI globally for the destination.
# NOTE: with NPM_CONFIG_PREFIX=/app/.npm-global, global installs land under
# /app/.npm-global/lib/node_modules (NOT /usr/local/lib/node_modules).
RUN npm install -g semver && \
    npm install -g @bitwarden/cli@${BW_CLI_OLD_VERSION} && \
    mkdir -p /opt/bw-old && \
    cp -r /app/.npm-global/lib/node_modules/@bitwarden/cli /opt/bw-old/ && \
    cp -r /app/.npm-global/lib/node_modules /opt/bw-old/node_modules && \
    npm install -g @bitwarden/cli@${BW_CLI_NEW_VERSION} && \
    chown -R bitwarden-sync:bitwarden-sync /app/.npm-global /opt/bw-old

# Create wrapper scripts for both versions
RUN echo '#!/bin/sh' > /usr/local/bin/bw-old && \
    echo 'NODE_PATH=/opt/bw-old/node_modules exec node /opt/bw-old/cli/build/bw.js "$@"' >> /usr/local/bin/bw-old && \
    chmod 0755 /usr/local/bin/bw-old && \
    ln -sf "$(which bw)" /usr/local/bin/bw-new

# Define a default cron schedule
ENV CRON_SCHEDULE="57 23 * * *"

# Create a cron job file with the defined schedule
RUN printf '%s bitwarden-sync /app/script.sh > /var/log/cron.log 2>&1\n' "$CRON_SCHEDULE" > /var/spool/cron/crontabs/bitwarden-sync && \
    chmod 0600 /var/spool/cron/crontabs/bitwarden-sync && \
    chown bitwarden-sync:bitwarden-sync /var/spool/cron/crontabs/bitwarden-sync

# Copy your script, entrypoint, shared CLI helpers, and encryption files
COPY --chown=bitwarden-sync:bitwarden-sync docker/bitwarden_sync.sh /app/script.sh
COPY --chown=bitwarden-sync:bitwarden-sync docker/entrypoint.sh /app/entrypoint.sh
COPY --chown=bitwarden-sync:bitwarden-sync docker/bw-cli-lib.sh /app/bw-cli-lib.sh

# Make your scripts executable
RUN chmod 0755 /app/script.sh /app/entrypoint.sh

# The entrypoint reconciles CLI versions (if overridden) then execs the CMD.
ENTRYPOINT ["/app/entrypoint.sh"]

# Start cron and log output to console
# CMD ["sh", "-c", "crond -f -L /dev/stdout -c /var/spool/cron/crontabs"]

USER bitwarden-sync