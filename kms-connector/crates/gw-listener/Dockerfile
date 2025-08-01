# syntax=docker/dockerfile:1

# RUST_IMAGE_VERSION arg can be used to override the default version
ARG RUST_IMAGE_VERSION=1.87.0

# Stage 1: Build gw-listener
FROM ghcr.io/zama-ai/fhevm/gci/rust-glibc:${RUST_IMAGE_VERSION} AS builder

# The profile used to run `cargo build`
ARG LTO_RELEASE=release

# Use root user for build stage
USER root

WORKDIR /app

COPY gateway-contracts/rust_bindings ./gateway-contracts/rust_bindings
COPY kms-connector ./kms-connector

# TODO: Remove SSH configuration once KMS Core repo is public.
#       This is currently needed as the KMS Connector depends on KMS Core to be built.
# Install build dependencies
RUN --mount=type=cache,target=/var/cache/apk,sharing=locked \
    --mount=type=cache,target=/var/lib/apk,sharing=locked \
    apk add --no-cache \
    openssh-keyscan \
    gcc \
    protoc

# Setup SSH and git
RUN mkdir -p -m 0600 /root/.ssh && \
    ssh-keyscan -H github.com >> ~/.ssh/known_hosts && \
    mkdir -p /app/kms-connector/bin

# Configure git with secure token handling
RUN --mount=type=secret,id=BLOCKCHAIN_ACTIONS_TOKEN,env=BLOCKCHAIN_ACTIONS_TOKEN \
    git config --global url."https://$BLOCKCHAIN_ACTIONS_TOKEN@github.com".insteadOf ssh://git@github.com

# Build with improved caching
RUN --mount=type=cache,target=$SCCACHE_DIR,sharing=locked cd kms-connector && \
    cargo build --profile=${LTO_RELEASE} -p gw-listener

# Stage 2: Runtime image
FROM cgr.dev/chainguard/glibc-dynamic:latest AS prod

COPY --from=builder /etc/group /etc/group
COPY --from=builder /etc/passwd /etc/passwd
COPY --from=builder --chown=fhevm:fhevm /app/kms-connector/target/release/gw-listener /app/kms-connector/bin/gw-listener

USER fhevm:fhevm

ENTRYPOINT ["/app/kms-connector/bin/gw-listener", "start"]

FROM prod AS dev
