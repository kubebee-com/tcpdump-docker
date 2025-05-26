# syntax=docker/dockerfile:1

# Stage 1: Builder
ARG DEBIAN_VERSION=12-slim
FROM debian:${DEBIAN_VERSION} AS builder

ARG TARGETPLATFORM
ARG BUILDPLATFORM

RUN apt-get update && \
    apt-get install -y --no-install-recommends traceroute && \
    rm -rf /var/lib/apt/lists/* && \
    # Determine actual installed path of traceroute
    ACTUAL_TRACEROUTE_PATH=$(which traceroute) && \
    if [ -z "$ACTUAL_TRACEROUTE_PATH" ]; then \
        echo "Error: traceroute command not found in PATH after installation." >&2; \
        exit 1; \
    fi && \
    echo "Traceroute found at: $ACTUAL_TRACEROUTE_PATH" && \
    # Define a consistent target path for the binary in staging & final image
    # This path should match the ENTRYPOINT declaration
    TARGET_TRACEROUTE_PATH_IN_IMAGE="/usr/bin/traceroute" && \
    # Create the directory for the binary in staging
    mkdir -p "/staging_root$(dirname "$TARGET_TRACEROUTE_PATH_IN_IMAGE")" && \
    # Copy the traceroute binary to the target path in staging
    cp "$ACTUAL_TRACEROUTE_PATH" "/staging_root$TARGET_TRACEROUTE_PATH_IN_IMAGE" && \
    chmod +x "/staging_root$TARGET_TRACEROUTE_PATH_IN_IMAGE" && \
    echo "Staged traceroute to /staging_root$TARGET_TRACEROUTE_PATH_IN_IMAGE" && \
    # Copy library dependencies based on the actual binary path
    ldd "$ACTUAL_TRACEROUTE_PATH" | grep "=>" | awk '{print $3}' | grep -v "^$" | while read lib_path; do \
        if [ -f "$lib_path" ]; then \
            mkdir -p "/staging_root$(dirname "$lib_path")"; \
            cp -L "$lib_path" "/staging_root$lib_path"; \
            echo "Copied library: $lib_path"; \
        fi \
    done && \
    # Also copy the dynamic linker/loader itself
    ldd "$ACTUAL_TRACEROUTE_PATH" | awk '/\/lib(64|)\/ld-linux(.*)\.so\.([0-9]+)/{print $1}' | while read ld_path; do \
        if [ -f "$ld_path" ]; then \
            mkdir -p "/staging_root$(dirname "$ld_path")"; \
            cp -L "$ld_path" "/staging_root$ld_path"; \
            echo "Copied dynamic loader: $ld_path"; \
        fi \
    done

# Stage 2: Final Image
FROM gcr.io/distroless/base-debian12 AS final

ARG TARGETPLATFORM
ARG DEBIAN_VERSION # For labeling consistency

COPY --from=builder /staging_root/ /

LABEL org.opencontainers.image.authors="kevin@kubebee.com"
LABEL org.opencontainers.image.architecture=$TARGETPLATFORM
LABEL name="traceroute"
LABEL version="${DEBIAN_VERSION}-distroless"

# Determine the traceroute path from the builder stage for the entrypoint
# This is a bit of a simplification; ideally, we'd pass the exact path.
# However, 'which traceroute' in the builder should give us the correct path.
# For debian, it's typically /usr/bin/traceroute or /usr/sbin/traceroute.
# We assume it's /usr/bin/traceroute for the ENTRYPOINT. If `which` found it elsewhere,
# this would need adjustment or a more dynamic way to set the entrypoint.
# For typical 'traceroute' package on Debian, it's /usr/bin/traceroute.
WORKDIR /
ENTRYPOINT ["/usr/bin/traceroute"]
CMD ["--help"]
