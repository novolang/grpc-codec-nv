# Changelog

Every published version, newest first. This file is on the publish
allow-list, so it travels with the package: it is the only thing a
consumer deciding whether to upgrade can read.

## 0.0.2 — 2026-09-15

README rewritten to the package README style guide (docs/writing-a-readme.md); no change to the interface.

## 0.0.1 — 2026-09-11

The **interface**, before anyone implements it.  Every signature, every
type and every effect row is published; every body is `todo()`, and the
release is stamped `NOT IMPLEMENTED — interface only`.  Adding this
package works and calling it panics.

- Five modules.  `grpcframe` is the five-byte length prefix and the
  deframer that reads it back; `grpcstatus` is the seventeen codes and
  the two trailers that carry one; `grpcmeta` is metadata, the content
  type and `grpc-timeout`; `grpcsvc` is `/package.Service/Method` over
  protobuf-nv's descriptors; `grpccall` is one call as a state machine.
- **`GrpcAction` is the load-bearing interface.**  Every call function
  answers a step holding the call, the events the caller learned and
  the actions the host must perform.  Nothing here writes a byte
  anywhere, and a host that walks the action list in order has
  implemented gRPC.
- **What the host provides is written down**: an HTTP/2 stream —
  HEADERS, DATA, TRAILERS, per-stream flow control, RST_STREAM — plus
  the clock and the compression.  Header pairs and byte chunks are the
  whole of the interface between the two, so a host can drive this
  package over any HTTP/2 implementation it has.
- **HTTP/2 is a named gap.**  `http-codec-nv` is HTTP/1.1 and cannot
  carry gRPC: no request trailers, no two independent directions, no
  per-stream flow control.  The missing package is `http2-nv` — frames,
  HPACK, the stream machine, flow control — and it is not a gRPC
  package, which is why it does not belong here.  `grpc-nv` is the
  `host` layer above both.
- **Five things a wrong implementation gets wrong quietly** are each a
  test: the length is big-endian while the protobuf inside it is not;
  `M` is minutes and `m` is milliseconds; `grpc-message` is
  percent-encoded; a trailers-only response is a HEADERS frame and a
  client that assumed otherwise waits forever; a proxy must decrement
  the deadline, because `grpc-timeout` is a duration and not an
  instant.
- **A status is not a transport fault**, and the two are different
  types.  `is_retryable` is true for `UNAVAILABLE` and nothing else —
  `DEADLINE_EXCEEDED` is false because the deadline may have passed
  after the work succeeded.
- **`GrpcFrameHead` is not a `@value` struct**, and the comment on it
  says why: a header read is fallible, so it is a `Result` payload, and
  a `@value` struct cannot be one (E2015).
- Depends on protobuf-nv for `PbService`, `PbMethod` and `PbStreaming`
  — a caller that fetched a descriptor from a reflection service hands
  the same values here.  No compression dependency: the flag says that
  a message is compressed and `grpc-encoding` says with what.
- No device claim: gRPC needs HTTP/2, and a device that could afford
  HPACK's dynamic table and per-stream flow control would not be using
  gRPC.
- Every value is from `PROTOCOL-HTTP2.md` or `grpc/status.proto`.

### Design notes

- The implementation this interface was checked against is `tonic`
  (Rust, MIT), whose codec half covers the same ground: the framing,
  the status codes, the metadata rules and the call state machine.
  gRPC's own `PROTOCOL-HTTP2.md` and `grpc/status.proto` are the
  specification.
