FROM rust:1-bookworm

ENV DEBIAN_FRONTEND=noninteractive
RUN apt-get update && apt-get install -y --no-install-recommends \
      ca-certificates build-essential clang libclang-dev cmake git libssl-dev \
      pkg-config autoconf automake libtool python3 \
    && rm -rf /var/lib/apt/lists/*

ARG CARGO_BUILD_JOBS=4
# Keep the existing fuzz targets usable after the audit loses network access.
RUN rustup toolchain install nightly --profile minimal --component rust-src \
    && cargo +nightly install cargo-fuzz --locked

# The scanner supplies the str0m checkout as the build context.
COPY . /src
WORKDIR /src
RUN cargo fetch --locked \
    && cargo fetch --manifest-path crates/fuzz/Cargo.toml

ENV CARGO_NET_OFFLINE=true \
    CARGO_INCREMENTAL=0 \
    CARGO_PROFILE_DEV_DEBUG=1 \
    CARGO_PROFILE_TEST_DEBUG=1 \
    RUSTFLAGS="-C force-frame-pointers=yes"

# Build examples and the optional Linux backends and application features.
RUN cargo build --locked --lib --examples --features pii,drv,openssl-dimpl,rust-crypto

# Match the regular CI's Linux crypto-provider configurations.
RUN set -e; \
    for backend in aws-lc-rs rust-crypto openssl openssl-dimpl; do \
      cargo test --locked --no-default-features --features "$backend"; \
      find target/debug/deps -maxdepth 1 -type f -executable ! -name '*.so' -delete; \
    done
RUN cargo test --locked -p str0m-proto -p is -p str0m-netem \
      -p str0m-aws-lc-rs -p str0m-rust-crypto -p str0m-openssl \
    && cargo test --locked --features pii,drv --test pii --test drv_input

# Precompile every harness, including its sanitizer runtime, for offline fuzzing.
RUN cargo +nightly fuzz build --fuzz-dir crates/fuzz
