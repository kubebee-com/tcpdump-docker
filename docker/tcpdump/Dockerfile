ARG DEBIAN_VERSION=12
FROM gcr.io/distroless/base-debian${DEBIAN_VERSION} AS base

FROM debian:${DEBIAN_VERSION}-slim AS builder

# TARGETARCH and TARGETVARIANT are automatically provided by Docker Buildx / BuildKit
ARG TARGETARCH
ARG TARGETVARIANT

# Define and create the tcpdump user and group for privilege dropping
# Choose UID/GID that don't conflict. distroless/base uses nonroot (65532)
ARG TCPDUMP_UID=65530
ARG TCPDUMP_GID=65530

# Determine the architecture-specific library path suffix
# and create a file to hold it for subsequent RUN commands.
# Also, prepare the staging root directory for all final artifacts.
RUN set -e; \
  echo "Preparing builder for TARGETARCH: ${TARGETARCH}, TARGETVARIANT: ${TARGETVARIANT}"; \
  case "${TARGETARCH}" in \
  amd64) LIB_PATH_SUFFIX="x86_64-linux-gnu" ;; \
  arm64) LIB_PATH_SUFFIX="aarch64-linux-gnu" ;; \
  arm) \
  case "${TARGETVARIANT}" in \
  v7) LIB_PATH_SUFFIX="arm-linux-gnueabihf" ;; \
  v6) LIB_PATH_SUFFIX="arm-linux-gnueabi" ;; \
  *) echo "Unsupported ARM variant: '${TARGETVARIANT}' for TARGETARCH arm. Please build for a specific variant like linux/arm/v7."; exit 1 ;; \
  esac ;; \
  s390x) LIB_PATH_SUFFIX="s390x-linux-gnu" ;; \
  ppc64le) LIB_PATH_SUFFIX="powerpc64le-linux-gnu" ;; \
  *) echo "Unsupported architecture: ${TARGETARCH}"; exit 1 ;; \
  esac; \
  echo "/usr/lib/${LIB_PATH_SUFFIX}" > /lib_path.txt; \
  LIB_FULL_PATH=$(cat /lib_path.txt); \
  echo "Library path for ${TARGETARCH} is ${LIB_FULL_PATH}"; \
  mkdir -p /staging_root/usr/bin; \
  mkdir -p /staging_root/etc; \
  mkdir -p /staging_root${LIB_FULL_PATH}

# As there is no echo executable in distroless image, it's hard to modify files there,
# so we copy the files from distroless to debian, modify them here, then copy them
# back.
COPY --from=base /etc/group /etc/passwd /tmp/

RUN set -e; \
  apt-get update && \
  apt-get install -y --no-install-recommends tcpdump && \
  # Append tcpdump group to /tmp/group (which contains original group info from distroless)
  (cat /tmp/group; echo "tcpdump:x:${TCPDUMP_GID}:") > /staging_root/etc/group && \
  # Append tcpdump user to /tmp/passwd (which contains original passwd info from distroless)
  (cat /tmp/passwd; echo "tcpdump:x:${TCPDUMP_UID}:${TCPDUMP_GID}:tcpdump unprivileged user:/nonexistent:/sbin/nologin") > /staging_root/etc/passwd

# Copy tcpdump binary to staging_root and make it executable
RUN cp /usr/bin/tcpdump /staging_root/usr/bin/tcpdump && \
  chmod +x /staging_root/usr/bin/tcpdump

# Copy necessary lib files.
RUN set -e; \
  LIB_FULL_PATH=$(cat /lib_path.txt); \
  DEST_LIB_DIR="/staging_root${LIB_FULL_PATH}"; \
  echo "Copying libraries from ${LIB_FULL_PATH} to ${DEST_LIB_DIR}"; \
  \
  # Define library patterns
  # Note: libcap.so.* (libcap2) might not be a direct strict dependency of tcpdump itself,
  # which usually depends on libcap-ng.so.*. Included as per your original list.
  REQUIRED_LIBS_PATTERNS="libpcap.so.* libcap-ng.so.* libcap.so.* libdbus-1.so.* libsystemd.so.* libgcrypt.so.* liblzma.so.* libzstd.so.* liblz4.so.* libgpg-error.so.*"; \
  \
  echo "--- Processing REQUIRED libraries ---"; \
  for pattern in ${REQUIRED_LIBS_PATTERNS}; do \
  echo "Searching for required library pattern: '${pattern}' in '${LIB_FULL_PATH}'"; \
  # Check if at least one file matches the pattern.
  # -print -quit: find first match, print it, then quit (efficient for just checking existence)
  # grep -q .: checks if find produced any output (i.e., found a file)
  if find "${LIB_FULL_PATH}" -maxdepth 1 -name "${pattern}" -print -quit 2>/dev/null | grep -q .; then \
  echo "Found. Copying files matching '${pattern}' to '${DEST_LIB_DIR}'..."; \
  find "${LIB_FULL_PATH}" -maxdepth 1 -name "${pattern}" -exec cp -L -v {} "${DEST_LIB_DIR}/" \;; \
  else \
  echo "Error: Required library matching pattern '${pattern}' not found in '${LIB_FULL_PATH}'. Build failed."; \
  exit 1; \
  fi; \
  done; \
  echo "Library copying process finished."

# Stage 2: Final distroless stage
# Use Google's distroless base image for Debian 11.
# This image contains essential runtime libraries like glibc and openssl.
FROM gcr.io/distroless/base-debian${DEBIAN_VERSION}

# Make ARGs available in this stage if needed for metadata or other logic
ARG DEBIAN_VERSION
ARG TARGETARCH

LABEL org.opencontainers.image.authors="kevin@kubebee.com"
LABEL org.opencontainers.image.architecture=${TARGETARCH}
LABEL name="tcpdump"
LABEL version="debian${DEBIAN_VERSION}-${TARGETARCH}"

WORKDIR /

# Copy all staged files from the builder (maintains directory structure)
COPY --from=builder /staging_root/ /

# Set the entrypoint to tcpdump
ENTRYPOINT ["/usr/bin/tcpdump"]

# Default command to show help if no arguments are provided to the entrypoint.
CMD ["--help"]
