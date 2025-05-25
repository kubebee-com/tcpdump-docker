# syntax=docker/dockerfile:1

# Stage 1: Builder
FROM debian:12-slim AS builder-base
RUN apt-get update && \
    apt-get install -y --no-install-recommends \
    g++ \
    make \
    && rm -rf /var/lib/apt/lists/*

FROM builder-base AS builder
WORKDIR /app
COPY ./app/main.cpp .
RUN g++ main.cpp -o netdiag -static

# Stage 2: Final Image
FROM debian:12-slim AS final
ARG DEBIAN_VERSION=12-slim
ARG TARGETPLATFORM

RUN apt-get update && \
    apt-get install -y --no-install-recommends \
    tcpdump \
    iputils-ping \
    traceroute \
    nmap \
    dnsutils \
    iproute2 \
    ca-certificates \
    && rm -rf /var/lib/apt/lists/*

COPY --from=builder /app/netdiag /usr/local/bin/netdiag
RUN chmod +x /usr/local/bin/netdiag

LABEL org.opencontainers.image.authors="kevin@kubebee.com"
LABEL org.opencontainers.image.architecture=${TARGETPLATFORM}
LABEL name="nettool"
LABEL version="debian12"

WORKDIR /
ENTRYPOINT ["/usr/local/bin/netdiag"]
