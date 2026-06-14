# SPDX-FileCopyrightText: Copyright (c) 2025-2026 NVIDIA CORPORATION & AFFILIATES. All rights reserved.
# SPDX-License-Identifier: Apache-2.0

# Sandbox image that runs `claude` under the NeMo Relay runtime.
#
#   openshell sandbox create \
#     --from examples/nemo-relay \
#     --policy examples/nemo-relay/sandbox-policy.yaml \
#     -- nemo-relay claude
#
# NeMo Relay wraps `claude`, attaches hooks/guardrails/traces, and forwards LLM
# traffic to OpenShell's in-sandbox inference router (https://inference.local)
# rather than straight to api.anthropic.com. OpenShell therefore keeps owning
# the real provider credentials and the egress policy; the sandbox never sees a
# raw ANTHROPIC_API_KEY.
FROM ghcr.io/nvidia/openshell-community/sandboxes/base:latest

# The base image runs as the unprivileged `sandbox` user. Installing into
# /usr/local/bin and /etc needs root, so switch up and restore afterwards.
USER root

# --- NeMo Relay CLI ------------------------------------------------------------
# Released binaries follow NeMo Relay's `cargo binstall` layout:
#   {repo}/releases/download/{version}/nemo-relay-cli-{target}-{version}
# The tag/path uses the bare version (no leading "v"). The target triple is
# detected from the build arch so this image builds on both amd64 and arm64.
# Pin NEMO_RELAY_VERSION to a version that has a published release asset.
ARG NEMO_RELAY_VERSION=0.3.0-beta.2
RUN set -eux; \
    arch="$(uname -m)"; \
    case "$arch" in \
      x86_64 | amd64)  target="x86_64-unknown-linux-musl" ;; \
      aarch64 | arm64) target="aarch64-unknown-linux-musl" ;; \
      *) echo "unsupported architecture: $arch" >&2; exit 1 ;; \
    esac; \
    curl -fsSL \
      "https://github.com/NVIDIA/NeMo-Relay/releases/download/${NEMO_RELAY_VERSION}/nemo-relay-cli-${target}-${NEMO_RELAY_VERSION}" \
      -o /usr/local/bin/nemo-relay; \
    chmod 0755 /usr/local/bin/nemo-relay; \
    nemo-relay --version

# Point NeMo Relay's Anthropic upstream at OpenShell's inference router. The env
# var overrides the bundled config file regardless of the config schema, so it
# is the schema-safe routing mechanism; the TOML files below are kept so the
# intent is discoverable without reading the Dockerfile.
ENV NEMO_RELAY_ANTHROPIC_BASE_URL=https://inference.local

# NeMo Relay's HTTP client must trust the sandbox CA to speak TLS to
# inference.local. The base image installs the sandbox CA into the system trust
# store; pointing SSL_CERT_FILE at it covers clients that read the env var
# directly. Adjust the path if your base image stores the CA elsewhere.
ENV SSL_CERT_FILE=/etc/ssl/certs/ca-certificates.crt

# Runtime config (upstream routing) and observability exporters. Exporters must
# live in plugins.toml; routing/agents live in config.toml.
COPY nemo-relay-config.toml /etc/nemo-relay/config.toml
COPY nemo-relay-plugins.toml /etc/nemo-relay/plugins.toml

# Restore the base image's default unprivileged user.
USER sandbox
