# str0m threat model

str0m is a Sans I/O WebRTC implementation in Rust. Applications provide network
packets and timeouts to an Rtc instance and drain its outputs. It implements
session negotiation, ICE, DTLS integration, SRTP/SRTCP, RTP/RTCP, media packet
assembly, bandwidth estimation, and SCTP data channels.

## Trust boundaries and scope

The target is https://github.com/algesten/str0m on main, including crates/proto,
crates/is, crates/netem, and the crypto providers in crypto/. Treat remote SDP,
STUN, DTLS, RTP, RTCP, SCTP/data-channel messages, and media payloads as
attacker-controlled. Consider both unauthenticated network traffic and malicious
authenticated WebRTC peers.

Prioritize authentication boundaries, DTLS certificate fingerprint enforcement,
ICE integrity and candidate selection, SRTP/SRTCP authentication and replay
protection, key/sequence-number handling, protocol demultiplexing, and isolation
between streams and sessions. Also examine parsers, reassembly, retransmission
queues, depacketizers, and resource use under loss, duplication, reordering,
truncation, and adversarial packet sizes or timestamps.

Application signaling transport, authorization of remote sessions, sockets, and
the event loop are supplied by the caller. Reproducers should follow the
documented mutation contract: drain poll_output to Output::Timeout after each
mutation before performing the next, and provide monotonic time values. Explain
any finding that depends on violating those requirements.

The Windows and Apple crypto providers remain in scope for source analysis;
the Linux image exercises the Linux-supported providers. Dependencies hosted in
other repositories, including dimpl and sctp-proto, are separate projects. Focus
reports here on str0m defects and its integration with those dependencies, and
identify the component containing the root cause.

## Build, offline tests, and fuzzing

The checkout, packet fixtures, and build artifacts are in /src. Dependencies for
the workspace and separate crates/fuzz package, nightly Rust, cargo-fuzz, and
the fuzz binaries are installed before networking is removed. Cargo is then
configured with CARGO_NET_OFFLINE=true.
The image retains libraries, examples, and fuzz binaries; regression-test
executables are rebuilt offline as needed to keep the image size manageable.

Run the regular Linux provider tests independently:

```sh
cd /src
for backend in aws-lc-rs rust-crypto openssl openssl-dimpl; do
  cargo test --locked --no-default-features --features "$backend" || exit "$?"
done
cargo test --locked -p str0m-proto -p is -p str0m-netem \
  -p str0m-aws-lc-rs -p str0m-rust-crypto -p str0m-openssl
cargo test --locked --features pii,drv --test pii --test drv_input
```

tests/common.rs provides in-memory peers and deterministic network simulation;
tests/data contains packet captures. Existing tests cover malformed packets,
DTLS fingerprints, SRTP replay, ICE, stream lifecycles, and network conditions.
Use these helpers for reproducers without relying on external services.

The existing fuzz targets cover SDP, STUN, RTP, RTCP, receiving, retransmission,
and media depacketization. List and run them offline with:

```sh
cargo +nightly fuzz list --fuzz-dir crates/fuzz
cargo +nightly fuzz run rtp_packet --fuzz-dir crates/fuzz -- -max_total_time=60
```

## Reports

Include the revision, crypto provider and feature flags, attacker capabilities,
the sequence of packets/API calls, expected and observed behavior, and a
deterministic test or minimized fuzz input. Prefer a small patch and regression
test addressing the root cause, and group variants of that cause together.

Explain the demonstrated security impact and realistic prerequisites. Distinguish
authentication bypass, disclosure, and acceptance of forged media or data from
harmless protocol deviations. For panics, hangs, or resource exhaustion, show
reachability through the public API and quantify the input and resources needed.
