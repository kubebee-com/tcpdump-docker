# syntax=docker/dockerfile:1

# Stage 1: Builder
ARG DEBIAN_VERSION=12-slim
FROM debian:${DEBIAN_VERSION} AS builder

RUN apt-get update && \
    apt-get install -y --no-install-recommends nmap && \
    rm -rf /var/lib/apt/lists/* && \
    # Determine actual installed path of nping
    ACTUAL_NPING_PATH=$(which nping) && \
    if [ -z "$ACTUAL_NPING_PATH" ]; then \
        echo "Error: nping command not found in PATH after installation." >&2; \
        exit 1; \
    fi && \
    echo "Nping found at: $ACTUAL_NPING_PATH" && \
    # Define a consistent target path for the binary in staging & final image
    TARGET_NPING_PATH_IN_IMAGE="/usr/bin/nping" && \
    # Create the directory for the binary in staging
    mkdir -p "/staging_root_nping$(dirname "$TARGET_NPING_PATH_IN_IMAGE")" && \
    # Copy the nping binary to the target path in staging
    cp "$ACTUAL_NPING_PATH" "/staging_root_nping$TARGET_NPING_PATH_IN_IMAGE" && \
    chmod +x "/staging_root_nping$TARGET_NPING_PATH_IN_IMAGE" && \
    echo "Staged nping to /staging_root_nping$TARGET_NPING_PATH_IN_IMAGE" && \
    # Copy library dependencies based on the actual binary path
    ldd "$ACTUAL_NPING_PATH" | grep "=>" | awk '{print $3}' | grep -v "^$" | while read lib_path; do \
        if [ -f "$lib_path" ]; then \
            mkdir -p "/staging_root_nping$(dirname "$lib_path")"; \
            cp -L "$lib_path" "/staging_root_nping$lib_path"; \
            echo "Copied library: $lib_path"; \
        fi \
    done && \
    # Also copy the dynamic linker/loader itself
    ldd "$ACTUAL_NPING_PATH" | awk '/\/lib(64|)\/ld-linux(.*)\.so\.([0-9]+)/{print $1}' | while read ld_path; do \
        if [ -f "$ld_path" ]; then \
            mkdir -p "/staging_root_nping$(dirname "$ld_path")"; \
            cp -L "$ld_path" "/staging_root_nping$ld_path"; \
            echo "Copied dynamic loader: $ld_path"; \
        fi \
    done
    # Note: nping might sometimes need data files from /usr/share/nmap for certain operations.
    # For now, we are only packaging the binary and its direct libs for basic functionality.
    # If extended features are needed, /usr/share/nmap (or parts of it) might need to be copied.

# Stage 2: Final Image
FROM gcr.io/distroless/base-debian12 AS final

ARG TARGETPLATFORM
ARG DEBIAN_VERSION=12-slim # For labeling consistency, though base is fixed

COPY --from=builder /staging_root_nping/ /

LABEL org.opencontainers.image.authors="kevin@kubebee.com"
LABEL org.opencontainers.image.architecture=$TARGETPLATFORM
LABEL name="nping"
LABEL version="debian${DEBIAN_VERSION}-distroless"

WORKDIR /
ENTRYPOINT ["/usr/bin/nping"]
CMD ["--version"]
