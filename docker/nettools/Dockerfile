# syntax=docker/dockerfile:1
ARG DEBIAN_VERSION=12

#==============================================================================
# Stage 1: netdiag-builder (C++ application)
#==============================================================================
FROM debian:12-slim AS netdiag-builder-base
RUN apt-get update && \
    apt-get install -y --no-install-recommends \
    g++ \
    make \
    && rm -rf /var/lib/apt/lists/*

FROM netdiag-builder-base AS netdiag-builder
WORKDIR /app
COPY ./app/main.cpp .
RUN g++ main.cpp -o netdiag -static

#==============================================================================
# Stage 2: tcpdump-prep
# Adapting logic from the original docker/tcpdump/Dockerfile
#==============================================================================
# ARG DEBIAN_VERSION=12 # As per original tcpdump Dockerfile for this stage
FROM debian:12-slim AS tcpdump-prep-builder-base

# Copy base /etc/group and /etc/passwd from a distroless image to modify
# ARG DEBIAN_VERSION=12
FROM gcr.io/distroless/base-debian${DEBIAN_VERSION} AS tcpdump-distroless-files
COPY /etc/group /etc/passwd /tmp/distroless-etc/

FROM tcpdump-prep-builder-base AS tcpdump-prep
ARG DEBIAN_VERSION # Make it available
ARG TARGETARCH # Automatically provided by Docker Buildx / BuildKit
ARG TARGETVARIANT # Automatically provided by Docker Buildx / BuildKit

ARG TCPDUMP_UID=65530
ARG TCPDUMP_GID=65530

# Prepare staging root and determine lib path
RUN set -e; \
  LIB_PATH_SUFFIX=""; \
  case "${TARGETARCH}" in \
    amd64) LIB_PATH_SUFFIX="x86_64-linux-gnu" ;; \
    arm64) LIB_PATH_SUFFIX="aarch64-linux-gnu" ;; \
    arm) \
      case "${TARGETVARIANT}" in \
        v7) LIB_PATH_SUFFIX="arm-linux-gnueabihf" ;; \
        v6) LIB_PATH_SUFFIX="arm-linux-gnueabi" ;; \
        *) echo "Unsupported ARM variant: '${TARGETVARIANT}' for TARGETARCH arm." >&2; exit 1 ;; \
      esac ;; \
    s390x) LIB_PATH_SUFFIX="s390x-linux-gnu" ;; \
    ppc64le) LIB_PATH_SUFFIX="powerpc64le-linux-gnu" ;; \
    *) echo "Unsupported architecture: ${TARGETARCH}" >&2; exit 1 ;; \
  esac; \
  echo "/usr/lib/${LIB_PATH_SUFFIX}" > /lib_path.txt; \
  mkdir -p /staging_root_tcpdump/usr/bin; \
  mkdir -p /staging_root_tcpdump/etc; \
  mkdir -p /staging_root_tcpdump/usr/lib/"${LIB_PATH_SUFFIX}"

COPY --from=tcpdump-distroless-files /tmp/distroless-etc/group /tmp/distroless-etc/passwd /tmp/
RUN apt-get update && \
    apt-get install -y --no-install-recommends tcpdump && \
    rm -rf /var/lib/apt/lists/* && \
    # Append tcpdump group and user to the copies from distroless
    (cat /tmp/group; echo "tcpdump:x:${TCPDUMP_GID}:") > /staging_root_tcpdump/etc/group && \
    (cat /tmp/passwd; echo "tcpdump:x:${TCPDUMP_UID}:${TCPDUMP_GID}:tcpdump unprivileged user:/nonexistent:/sbin/nologin") > /staging_root_tcpdump/etc/passwd

RUN cp /usr/bin/tcpdump /staging_root_tcpdump/usr/bin/tcpdump && \
    chmod +x /staging_root_tcpdump/usr/bin/tcpdump

# Copy necessary libraries for tcpdump
RUN set -e; \
  LIB_FULL_PATH=$(cat /lib_path.txt); \
  DEST_LIB_DIR="/staging_root_tcpdump${LIB_FULL_PATH}"; \
  TCPDUMP_PATH="/usr/bin/tcpdump"; \
  # Use ldd to find direct dependencies
  ldd "$TCPDUMP_PATH" | grep "=>" | awk '{print $3}' | grep -v "^$" | while read lib_path; do \
    if [ -f "$lib_path" ]; then \
      mkdir -p "$(dirname "${DEST_LIB_DIR}$(echo $lib_path | cut -d'/' -f2-)")"; \
      cp -L "$lib_path" "${DEST_LIB_DIR}/$(basename $lib_path)"; \
      echo "Copied by ldd: $lib_path to ${DEST_LIB_DIR}/$(basename $lib_path)"; \
    fi \
  done && \
  # Also copy the dynamic linker/loader itself
  ldd "$TCPDUMP_PATH" | awk '/\/lib(64|)\/ld-linux(.*)\.so\.([0-9]+)/{print $1}' | while read ld_path; do \
    if [ -f "$ld_path" ]; then \
      mkdir -p "$(dirname "${DEST_LIB_DIR}$(echo $ld_path | cut -d'/' -f2-)")"; \
      cp -L "$ld_path" "${DEST_LIB_DIR}/$(basename $ld_path)"; \
      echo "Copied loader by ldd: $ld_path to ${DEST_LIB_DIR}/$(basename $ld_path)"; \
    fi \
  done; \
  # Explicitly copy libraries as per original Dockerfile's patterns for robustness
  REQUIRED_LIBS_PATTERNS="libpcap.so.* libcap-ng.so.*"; \
  # Optional: libcap.so.* libdbus-1.so.* libsystemd.so.* libgcrypt.so.* liblzma.so.* libzstd.so.* liblz4.so.* libgpg-error.so.*
  # For nettool, focusing on essential ones for tcpdump identified by ldd and core functionality.
  # The original tcpdump Dockerfile had a more extensive list, some of which might be indirect or for specific features.
  # We are relying more on ldd here for direct deps, and adding libpcap & libcap-ng explicitly.
  echo "--- Processing REQUIRED libraries explicitly ---"; \
  for pattern in ${REQUIRED_LIBS_PATTERNS}; do \
    find "${LIB_FULL_PATH}" -maxdepth 1 -name "${pattern}" -exec cp -L -v {} "${DEST_LIB_DIR}/" \;; \
  done; \
  echo "Library copying process finished."

#==============================================================================
# Stage 3: ping-prep
#==============================================================================
FROM debian:12-slim AS ping-prep
RUN apt-get update && \
    apt-get install -y --no-install-recommends iputils-ping && \
    rm -rf /var/lib/apt/lists/*
RUN mkdir -p /staging_root_ping/bin && \
    cp /bin/ping /staging_root_ping/bin/ping && \
    chmod +x /staging_root_ping/bin/ping && \
    ldd /bin/ping | grep "=>" | awk '{print $3}' | grep -v "^$" | while read lib_path; do \
        if [ -f "$lib_path" ]; then mkdir -p "/staging_root_ping$(dirname "$lib_path")"; cp -L "$lib_path" "/staging_root_ping$lib_path"; fi \
    done && \
    ldd /bin/ping | awk '/\/lib(64|)\/ld-linux(.*)\.so\.([0-9]+)/{print $1}' | while read ld_path; do \
        if [ -f "$ld_path" ]; then mkdir -p "/staging_root_ping$(dirname "$ld_path")"; cp -L "$ld_path" "/staging_root_ping$ld_path"; fi \
    done

#==============================================================================
# Stage 4: traceroute-prep
#==============================================================================
FROM debian:12-slim AS traceroute-prep
RUN apt-get update && \
    apt-get install -y --no-install-recommends traceroute && \
    rm -rf /var/lib/apt/lists/*
RUN TRACEROUTE_PATH=$(which traceroute) && \
    mkdir -p "/staging_root_traceroute$(dirname "$TRACEROUTE_PATH")" && \
    cp "$TRACEROUTE_PATH" "/staging_root_traceroute$TRACEROUTE_PATH" && \
    chmod +x "/staging_root_traceroute$TRACEROUTE_PATH" && \
    ldd "$TRACEROUTE_PATH" | grep "=>" | awk '{print $3}' | grep -v "^$" | while read lib_path; do \
        if [ -f "$lib_path" ]; then mkdir -p "/staging_root_traceroute$(dirname "$lib_path")"; cp -L "$lib_path" "/staging_root_traceroute$lib_path"; fi \
    done && \
    ldd "$TRACEROUTE_PATH" | awk '/\/lib(64|)\/ld-linux(.*)\.so\.([0-9]+)/{print $1}' | while read ld_path; do \
        if [ -f "$ld_path" ]; then mkdir -p "/staging_root_traceroute$(dirname "$ld_path")"; cp -L "$ld_path" "/staging_root_traceroute$ld_path"; fi \
    done

#==============================================================================
# Stage 5: nmap-prep
#==============================================================================
FROM debian:12-slim AS nmap-prep
RUN apt-get update && \
    apt-get install -y --no-install-recommends nmap && \
    rm -rf /var/lib/apt/lists/*
RUN NMAP_PATH=$(which nmap) && \
    mkdir -p "/staging_root_nmap$(dirname "$NMAP_PATH")" && \
    cp "$NMAP_PATH" "/staging_root_nmap$NMAP_PATH" && \
    chmod +x "/staging_root_nmap$NMAP_PATH" && \
    ldd "$NMAP_PATH" | grep "=>" | awk '{print $3}' | grep -v "^$" | while read lib_path; do \
        if [ -f "$lib_path" ]; then mkdir -p "/staging_root_nmap$(dirname "$lib_path")"; cp -L "$lib_path" "/staging_root_nmap$lib_path"; fi \
    done && \
    ldd "$NMAP_PATH" | awk '/\/lib(64|)\/ld-linux(.*)\.so\.([0-9]+)/{print $1}' | while read ld_path; do \
        if [ -f "$ld_path" ]; then mkdir -p "/staging_root_nmap$(dirname "$ld_path")"; cp -L "$ld_path" "/staging_root_nmap$ld_path"; fi \
    done && \
    if [ -d /usr/share/nmap ]; then cp -a /usr/share/nmap /staging_root_nmap/usr/share/nmap; fi

#==============================================================================
# Stage 6: dig-prep
#==============================================================================
FROM debian:12-slim AS dig-prep
RUN apt-get update && \
    apt-get install -y --no-install-recommends dnsutils && \
    rm -rf /var/lib/apt/lists/*
RUN DIG_PATH=$(which dig) && \
    mkdir -p "/staging_root_dig$(dirname "$DIG_PATH")" && \
    cp "$DIG_PATH" "/staging_root_dig$DIG_PATH" && \
    chmod +x "/staging_root_dig$DIG_PATH" && \
    ldd "$DIG_PATH" | grep "=>" | awk '{print $3}' | grep -v "^$" | while read lib_path; do \
        if [ -f "$lib_path" ]; then mkdir -p "/staging_root_dig$(dirname "$lib_path")"; cp -L "$lib_path" "/staging_root_dig$lib_path"; fi \
    done && \
    ldd "$DIG_PATH" | awk '/\/lib(64|)\/ld-linux(.*)\.so\.([0-9]+)/{print $1}' | while read ld_path; do \
        if [ -f "$ld_path" ]; then mkdir -p "/staging_root_dig$(dirname "$ld_path")"; cp -L "$ld_path" "/staging_root_dig$ld_path"; fi \
    done

#==============================================================================
# Stage 7: ss-prep
#==============================================================================
FROM debian:12-slim AS ss-prep
RUN apt-get update && \
    apt-get install -y --no-install-recommends iproute2 && \
    rm -rf /var/lib/apt/lists/*
RUN SS_PATH=$(which ss) && \
    mkdir -p "/staging_root_ss$(dirname "$SS_PATH")" && \
    cp "$SS_PATH" "/staging_root_ss$SS_PATH" && \
    chmod +x "/staging_root_ss$SS_PATH" && \
    ldd "$SS_PATH" | grep "=>" | awk '{print $3}' | grep -v "^$" | while read lib_path; do \
        if [ -f "$lib_path" ]; then mkdir -p "/staging_root_ss$(dirname "$lib_path")"; cp -L "$lib_path" "/staging_root_ss$lib_path"; fi \
    done && \
    ldd "$SS_PATH" | awk '/\/lib(64|)\/ld-linux(.*)\.so\.([0-9]+)/{print $1}' | while read ld_path; do \
        if [ -f "$ld_path" ]; then mkdir -p "/staging_root_ss$(dirname "$ld_path")"; cp -L "$ld_path" "/staging_root_ss$ld_path"; fi \
    done

#==============================================================================
# Final Stage: nettool-final
#==============================================================================
FROM gcr.io/distroless/base-debian12 AS nettool-final
ARG TARGETPLATFORM
# ARG DEBIAN_VERSION=12 # Match version used for distroless base

LABEL org.opencontainers.image.authors="kevin@kubebee.com"
LABEL org.opencontainers.image.architecture=${TARGETPLATFORM}
LABEL name="nettools"
LABEL version="debian${DEBIAN_VERSION}-distroless"

WORKDIR /

# Copy the C++ application
COPY --from=netdiag-builder /app/netdiag /usr/local/bin/netdiag
RUN chmod +x /usr/local/bin/netdiag

# Copy artifacts from tcpdump-prep (includes binaries, libs, and modified /etc files)
COPY --from=tcpdump-prep /staging_root_tcpdump/ /

# Copy artifacts from other prep stages
COPY --from=ping-prep /staging_root_ping/ /
COPY --from=traceroute-prep /staging_root_traceroute/ /
COPY --from=nmap-prep /staging_root_nmap/ /
COPY --from=dig-prep /staging_root_dig/ /
COPY --from=ss-prep /staging_root_ss/ /

# Ensure all copied binaries are executable (belt and braces)
RUN find /usr/bin /usr/sbin /bin /usr/local/bin -type f -exec chmod +x {} \; || true

ENTRYPOINT ["/usr/local/bin/netdiag"]
# CMD ["--help"] # Assuming netdiag might not have --help yet
CMD []
