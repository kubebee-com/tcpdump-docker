# syntax=docker/dockerfile:1

# Stage 1: Builder
ARG DEBIAN_VERSION=12-slim
FROM debian:${DEBIAN_VERSION} AS builder

ARG TARGETPLATFORM
ARG BUILDPLATFORM

RUN apt-get update && \
    apt-get install -y --no-install-recommends \
    iputils-ping \
    && rm -rf /var/lib/apt/lists/*

RUN mkdir -p /staging_root/bin && \
    cp /bin/ping /staging_root/bin/ping && \
    chmod +x /staging_root/bin/ping && \
    # Identify and copy library dependencies
    # Regex extracts the library path (3rd field, or 2nd if it's a direct link like linux-vdso.so.1)
    # It handles lines like:
    # libcap.so.2 => /lib/x86_64-linux-gnu/libcap.so.2 (0x00007f5c3e5b9000)
    # /lib64/ld-linux-x86-64.so.2 (0x00007f5c3e5f6000)
    # linux-vdso.so.1 (0x00007ffc7f5fe000) - This one is virtual and not copied
    ldd /bin/ping | grep "=>" | awk '{print $3}' | grep -v "^$" | while read lib_path; do \
        if [ -f "$lib_path" ]; then \
            mkdir -p "/staging_root$(dirname "$lib_path")"; \
            cp -L "$lib_path" "/staging_root$lib_path"; \
            echo "Copied $lib_path"; \
        fi \
    done && \
    # Also copy the dynamic linker/loader itself if present in ldd output (e.g. /lib64/ld-linux-x86-64.so.2)
    ldd /bin/ping | awk '/\/lib64\/ld-linux-x86-64\.so\.2/{print $1}' | while read ld_path; do \
        if [ -f "$ld_path" ]; then \
            mkdir -p "/staging_root$(dirname "$ld_path")"; \
            cp -L "$ld_path" "/staging_root$ld_path"; \
            echo "Copied loader $ld_path"; \
        fi \
    done


# Stage 2: Final Image
FROM gcr.io/distroless/base-debian12 AS final

ARG TARGETPLATFORM
ARG DEBIAN_VERSION # For labeling consistency, though base image is fixed

COPY --from=builder /staging_root/ /

LABEL org.opencontainers.image.authors="kevin@kubebee.com"
LABEL org.opencontainers.image.architecture=$TARGETPLATFORM
LABEL name="ping"
LABEL version="${DEBIAN_VERSION}-distroless"

WORKDIR /
ENTRYPOINT ["/bin/ping"]
CMD ["--help"]
