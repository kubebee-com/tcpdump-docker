# syntax=docker/dockerfile:1

# Stage 1: Builder
ARG DEBIAN_VERSION=12-slim
FROM debian:${DEBIAN_VERSION} AS builder

ARG TARGETPLATFORM
ARG BUILDPLATFORM

RUN apt-get update && \
    apt-get install -y --no-install-recommends \
    nmap \
    && rm -rf /var/lib/apt/lists/*

RUN mkdir -p /staging_root/usr/bin /staging_root/usr/share && \
    NMAP_PATH=$(which nmap) && \
    echo "Nmap found at: $NMAP_PATH" && \
    cp "$NMAP_PATH" "/staging_root$NMAP_PATH" && \
    chmod +x "/staging_root$NMAP_PATH" && \
    # Identify and copy library dependencies
    ldd "$NMAP_PATH" | grep "=>" | awk '{print $3}' | grep -v "^$" | while read lib_path; do \
        if [ -f "$lib_path" ]; then \
            mkdir -p "/staging_root$(dirname "$lib_path")"; \
            cp -L "$lib_path" "/staging_root$lib_path"; \
            echo "Copied library $lib_path"; \
        fi \
    done && \
    # Also copy the dynamic linker/loader itself if present in ldd output
    ldd "$NMAP_PATH" | awk '/\/lib(64|)\/ld-linux(.*)\.so\.([0-9]+)/{print $1}' | while read ld_path; do \
        if [ -f "$ld_path" ]; then \
            mkdir -p "/staging_root$(dirname "$ld_path")"; \
            cp -L "$ld_path" "/staging_root$ld_path"; \
            echo "Copied loader $ld_path"; \
        fi \
    done && \
    # Copy Nmap's auxiliary files
    if [ -d /usr/share/nmap ]; then \
        echo "Copying /usr/share/nmap..."; \
        cp -a /usr/share/nmap /staging_root/usr/share/nmap; \
    else \
        echo "Warning: /usr/share/nmap not found in builder stage."; \
    fi

# Stage 2: Final Image
FROM gcr.io/distroless/base-debian12 AS final

ARG TARGETPLATFORM
ARG DEBIAN_VERSION # For labeling consistency

COPY --from=builder /staging_root/ /

LABEL org.opencontainers.image.authors="kevin@kubebee.com"
LABEL org.opencontainers.image.architecture=$TARGETPLATFORM
LABEL name="nmap"
LABEL version="${DEBIAN_VERSION}-distroless"

WORKDIR /
ENTRYPOINT ["/usr/bin/nmap"]
CMD ["--help"]
