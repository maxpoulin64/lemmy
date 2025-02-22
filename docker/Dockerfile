#
# Docker multiarch image:
#   We build the Lemmy binary for amd64 and arm64 in individual stages using the blackdex/rust-musl image (github.com/blackdex/rust-musl). 
#   This image uses musl-cross-make (github.com/richfelker/musl-cross-make) to build a musl cross compilation toolchain for the target 
#   architecture. It also includes pre-built static libraries such as libpq. These libraries can improve the compile time and eliminate 
#   the requirement for extra dependencies in the final image.
#
#   During each build stage, we use the blackdex/rust-musl openssl 3 images and configure PQ_LIB_DIR=/usr/local/musl/pq15/lib to use 
#   libpq v15. We also ensure the installation of the Rust toolchain corresponding to the target architecture using:
#   `rustup target add $TARGET-unknown-linux-musl`.
#

ARG RUST_VERSION=1.71.0
ARG ALPINE_VERSION=3.18
ARG CARGO_BUILD_FEATURES=default
ARG RUST_RELEASE_MODE=debug
ARG UID=911
ARG GID=911

# AMD64 builder base
FROM --platform=${BUILDPLATFORM} blackdex/rust-musl:x86_64-musl-stable-${RUST_VERSION}-openssl3 AS base-amd64

ENV DEBIAN_FRONTEND=noninteractive
ENV CARGO_HOME=/root/.cargo
ENV PQ_LIB_DIR=/usr/local/musl/pq15/lib

RUN apt update && apt install -y \
    --no-install-recommends \
    git

RUN mkdir -pv "${CARGO_HOME}" && \
    rustup set profile minimal && \
    rustup target add x86_64-unknown-linux-musl

# ARM64 builder base
FROM --platform=${BUILDPLATFORM} blackdex/rust-musl:aarch64-musl-stable-${RUST_VERSION}-openssl3 AS base-arm64

ENV DEBIAN_FRONTEND=noninteractive
ENV CARGO_HOME=/root/.cargo
ENV PQ_LIB_DIR=/usr/local/musl/pq15/lib

RUN apt update && apt install -y \
    --no-install-recommends \
    git

RUN mkdir -pv "${CARGO_HOME}" && \
    rustup set profile minimal && \
    rustup target add aarch64-unknown-linux-musl

# AMD64 builder
FROM base-amd64 AS build-amd64

ARG CARGO_BUILD_FEATURES
ARG RUST_RELEASE_MODE

WORKDIR /lemmy

COPY . ./

# Debug build
RUN --mount=type=cache,target=/lemmy/target set -ex; \
    if [ "${RUST_RELEASE_MODE}" = "debug" ]; then \
        echo "pub const VERSION: &str = \"$(git describe --tag)\";" > crates/utils/src/version.rs; \
        cargo build --target=x86_64-unknown-linux-musl --features "${CARGO_BUILD_FEATURES}"; \
        mv target/x86_64-unknown-linux-musl/debug/lemmy_server ./lemmy; \
    fi

# Release build
RUN set -ex; \
    if [ "${RUST_RELEASE_MODE}" = "release" ]; then \
        echo "pub const VERSION: &str = \"$(git describe --tag)\";" > crates/utils/src/version.rs; \
        cargo build --target=x86_64-unknown-linux-musl --features "${CARGO_BUILD_FEATURES}" --release; \
        mv target/x86_64-unknown-linux-musl/release/lemmy_server ./lemmy; \
    fi

# ARM64 builder
FROM base-arm64 AS build-arm64

ARG CARGO_BUILD_FEATURES
ARG RUST_RELEASE_MODE

WORKDIR /lemmy

COPY . ./

# Debug build
RUN --mount=type=cache,target=/lemmy/target set -ex; \
    if [ "${RUST_RELEASE_MODE}" = "debug" ]; then \
        echo "pub const VERSION: &str = \"$(git describe --tag)\";" > crates/utils/src/version.rs; \
        cargo build --target=aarch64-unknown-linux-musl --features "${CARGO_BUILD_FEATURES}"; \
        mv target/aarch64-unknown-linux-musl/debug/lemmy_server ./lemmy; \
    fi

# Release build
RUN set -ex; \
    if [ "${RUST_RELEASE_MODE}" = "release" ]; then \
        echo "pub const VERSION: &str = \"$(git describe --tag)\";" > crates/utils/src/version.rs; \
        cargo build --target=aarch64-unknown-linux-musl --features "${CARGO_BUILD_FEATURES}" --release; \
        mv target/aarch64-unknown-linux-musl/release/lemmy_server ./lemmy; \
    fi

# Get target binary
FROM build-${TARGETARCH} AS build

## Final image
FROM alpine:${ALPINE_VERSION} 

ARG UID
ARG GID

RUN apk add --no-cache \
    ca-certificates

COPY --from=build --chmod=0755 /lemmy/lemmy /usr/local/bin

RUN addgroup -S -g ${GID} lemmy && \
    adduser -S -H -D -G lemmy -u ${UID} -g "" -s /sbin/nologin lemmy

USER lemmy

CMD ["lemmy"]

EXPOSE 8536

STOPSIGNAL SIGTERM