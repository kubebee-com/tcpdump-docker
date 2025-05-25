# syntax=docker/dockerfile:1

# Stage 1: Builder
ARG DEBIAN_VERSION=12-slim
FROM debian:${DEBIAN_VERSION} AS builder

ARG TARGETPLATFORM
ARG BUILDPLATFORM

RUN apt-get update && \
    apt-get install -y --no-install-recommends \
    dnsutils \
    && rm -rf /var/lib/apt/lists/*

RUN mkdir -p /staging_root/usr/bin && \
    DIG_PATH=$(which dig) && \
    echo "Dig found at: $DIG_PATH" && \
    # Ensure the path is /usr/bin/dig as expected for entrypoint, or adjust entrypoint
    if [ "$DIG_PATH" != "/usr/bin/dig" ]; then \
        echo "Warning: dig is not at /usr/bin/dig, found at $DIG_PATH. Adjust ENTRYPOINT if necessary."; \
    fi && \
    cp "$DIG_PATH" "/staging_root/usr/bin/dig" && \
    chmod +x "/staging_root/usr/bin/dig" && \
    # Identify and copy library dependencies
    ldd "$DIG_PATH" | grep "=>" | awk '{print $3}' | grep -v "^$" | while read lib_path; do \
        if [ -f "$lib_path" ]; then \
            mkdir -p "/staging_root$(dirname "$lib_path")"; \
            cp -L "$lib_path" "/staging_root$lib_path"; \
            echo "Copied library $lib_path"; \
        fi \
    done && \
    # Also copy the dynamic linker/loader itself if present in ldd output
    ldd "$DIG_PATH" | awk '/\/lib(64|)\/ld-linux(.*)\.so\.([0-9]+)/{print $1}' | while read ld_path; do \
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
LABEL name="dig"
LABEL version="${DEBIAN_VERSION}-distroless"

WORKDIR /
ENTRYPOINT ["/usr/bin/dig"]
CMD ["google.com"]
