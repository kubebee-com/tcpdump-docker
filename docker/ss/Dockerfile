# syntax=docker/dockerfile:1

# Stage 1: Builder
ARG DEBIAN_VERSION=12-slim
FROM debian:${DEBIAN_VERSION} AS builder

ARG TARGETPLATFORM
ARG BUILDPLATFORM

RUN apt-get update && \
    apt-get install -y --no-install-recommends \
    iproute2 \
    && rm -rf /var/lib/apt/lists/*

RUN mkdir -p /staging_root/usr/bin && \
    SS_PATH=$(which ss) && \
    echo "ss found at: $SS_PATH" && \
    # Ensure the path is /usr/bin/ss as expected for entrypoint, or adjust entrypoint
    if [ "$SS_PATH" != "/usr/bin/ss" ]; then \
        # Some systems might have it in /usr/sbin/ss, common for iproute2 tools
        # We will copy it to /usr/bin/ss in staging to match the common ENTRYPOINT
        echo "Warning: ss is not at /usr/bin/ss, found at $SS_PATH. Will copy to /staging_root/usr/bin/ss."; \
    fi && \
    cp "$SS_PATH" "/staging_root/usr/bin/ss" && \
    chmod +x "/staging_root/usr/bin/ss" && \
    # Identify and copy library dependencies
    ldd "$SS_PATH" | grep "=>" | awk '{print $3}' | grep -v "^$" | while read lib_path; do \
        if [ -f "$lib_path" ]; then \
            mkdir -p "/staging_root$(dirname "$lib_path")"; \
            cp -L "$lib_path" "/staging_root$lib_path"; \
            echo "Copied library $lib_path"; \
        fi \
    done && \
    # Also copy the dynamic linker/loader itself if present in ldd output
    ldd "$SS_PATH" | awk '/\/lib(64|)\/ld-linux(.*)\.so\.([0-9]+)/{print $1}' | while read ld_path; do \
        if [ -f "$ld_path" ]; then \
            mkdir -p "/staging_root$(dirname "$ld_path")"; \
            cp -L "$ld_path" "/staging_root$ld_path"; \
            echo "Copied loader $ld_path"; \
        fi \
    done

# Stage 2: Final Image
FROM gcr.io/distroless/base-debian12 AS final

ARG TARGETPLATFORM
ARG DEBIAN_VERSION # For labeling consistency

COPY --from=builder /staging_root/ /

LABEL org.opencontainers.image.authors="kevin@kubebee.com"
LABEL org.opencontainers.image.architecture=$TARGETPLATFORM
LABEL name="ss"
LABEL version="${DEBIAN_VERSION}-distroless"

WORKDIR /
ENTRYPOINT ["/usr/bin/ss"]
CMD ["-tulpn"]
