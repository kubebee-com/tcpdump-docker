# syntax=docker/dockerfile:1

ARG DEBIAN_VERSION=12-slim
ARG TARGETPLATFORM

FROM debian:${DEBIAN_VERSION} AS base
RUN apt-get update && \
    apt-get install -y --no-install-recommends \
    dnsutils \
    ca-certificates \
    && rm -rf /var/lib/apt/lists/*

LABEL org.opencontainers.image.authors="kevin@kubebee.com"
LABEL org.opencontainers.image.architecture=$TARGETPLATFORM
LABEL name="dig"
LABEL version="debian12"

ENTRYPOINT [ "/usr/bin/dig" ]
