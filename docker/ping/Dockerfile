# syntax=docker/dockerfile:1

# TODO: update this as new versions of debian are released
ARG DEBIAN_VERSION=12-slim

# TODO: update this as new versions of tcpdump are released
ARG TCPDUMP_VERSION=4.99.4-1~deb12u1
ARG TCPDUMP_FULL_VERSION=${TCPDUMP_VERSION}-1~deb12u1

FROM debian:${DEBIAN_VERSION} AS base
ARG TARGETPLATFORM
ARG BUILDPLATFORM
RUN dpkg --add-architecture armhf && \
    dpkg --add-architecture armel && \
    dpkg --add-architecture arm64 && \
    dpkg --add-architecture i386 && \
    dpkg --add-architecture ppc64el && \
    dpkg --add-architecture mips64el && \
    dpkg --add-architecture s390x && \
    apt-get update && \
    apt-get install -y --no-install-recommends \
    iputils-ping \
    ca-certificates \
    && rm -rf /var/lib/apt/lists/*

LABEL org.opencontainers.image.authors="The Tcpdump Group <tcpdump-workers@lists.tcpdump.org>"
LABEL org.opencontainers.image.architecture=$TARGETPLATFORM
LABEL name="ping"
LABEL version="1.0"

ENTRYPOINT [ "/bin/ping" ]
