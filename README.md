# grpc-codec-nv

**Status: NOT IMPLEMENTED — interface only.**

Every public function below is published with its signature and its
effect row, and every body is `todo()`.  Installing this package works;
calling it panics with `not implemented`.

## What this is

gRPC without a socket.  One call is a state machine: it takes what
arrived on an HTTP/2 stream the host owns and answers what the host
must send back, and nothing in this package touches the network.

The protocol itself is smaller than its reputation.  A call is an
HTTP/2 stream; the `:path` is `/package.Service/Method`; the body is
zero or more messages, each behind a five-byte prefix of one compressed
flag and a four-byte big-endian length; and the answer is in the
TRAILERS, as `grpc-status` and `grpc-message`.  Everything else —
metadata, deadlines, the content type — is headers.

Five surfaces, and a reader should know which one they are on.

| surface | module | reach for it when |
| --- | --- | --- |
| the **framing** | `grpcframe` | you have DATA bytes and want messages, or the reverse |
| the **status** | `grpcstatus` | a call ended and you want to know how |
| the **headers** | `grpcmeta` | metadata, the content type, the deadline |
| the **service** | `grpcsvc` | you are routing, or building a path |
| the **call** | `grpccall` | you are writing a client or a server |

## Adding it, and checking it

```bash
novo pkg add grpc-codec-nv    # into your novo.toml
novo pkg build                # type- and effect-check the package
novo test --isolate tests/grpc_codec_tests.nv
```

`novo test` is red today and that is the point of the release: all
sixty-seven assertions fail with `not implemented: <module>.<fn>`.  They
turn green one at a time as bodies land.

## The one example that will work

```novo
use std.bytes
use grpcframe

fn main() [io]
    // A protobuf message with gRPC's five bytes in front of it: one
    // flag, four big-endian length bytes, then the body.
    let body = bytes.from_hex("089601") ?? bytes.zeros(0)
    match grpcframe.frame(body, false, grpcframe.default_limits())
        Ok(out) => println(bytes.to_hex(out))   // 0000000003089601
        Err(e)  => println(e.message())
```

## The load-bearing interface

`GrpcAction` — the enum a `GrpcCallStep` carries.

Every function in `grpccall` answers a step holding the call after it,
the EVENTS the caller has learned, and the ACTIONS the host must
perform.  Nothing in this package writes a byte anywhere; the action
list is the whole of what reaches the network, and a host that walks it
in order has implemented gRPC.

```novo norun:pseudo
enum GrpcAction
    GrpcSendHeaders(md: GrpcMetadata)
    GrpcSendData(data: Bytes, end_stream: Bool)
    GrpcSendTrailers(md: GrpcMetadata)
    GrpcSendEndStream
    GrpcSendReset(http2_code: Int)
```

That is `docs/publishing.md` § How a `core` package takes bytes from
its host applied to a protocol rather than to a format: the core is a
state machine, and the host is the only party that performs anything.
It is also what makes the protocol testable — every test in
`tests/grpc_codec_tests.nv` is values in and values out, with no socket
and no clock.

## What the host must provide

An HTTP/2 stream, and specifically six things:

1. **Opening a stream**, and closing it.
2. **HEADERS in and out**, as name-value pairs.  This package hands over
   `GrpcMetadata` — pseudo-headers included, `:`-prefixed names first,
   which is the order HTTP/2 requires — and takes the same back.  HPACK
   never appears in this API.
3. **DATA in and out**, as bytes, with HTTP/2's `end_stream` flag.
4. **TRAILERS**, which is a HEADERS frame with `end_stream` set.
5. **Per-stream flow control** — window updates, and not sending more
   than the peer's window allows.
6. **RST_STREAM** in both directions, with the error code as a number.

And two things the host must also know: **the clock**, because a `core`
package has none (`grpcmeta.remaining` takes the elapsed nanoseconds as
a parameter), and **compression**, because the frame's flag says THAT a
message is compressed and `grpc-encoding` says with what — the bytes
are the host's to inflate, with flate-nv or whatever the header named.

## Where HTTP/2 belongs

Not here, and not in http-codec-nv either.  **HTTP/1.1 cannot carry
gRPC**, and the reason is three-fold rather than a matter of taste:

- **Trailers.**  Every gRPC response ends with `grpc-status` AFTER the
  body, because a server that has streamed nine of ten responses and
  then fails has already sent `:status: 200`.  HTTP/1.1 has trailers
  only in chunked transfer encoding, they are optional, proxies strip
  them routinely, and a REQUEST cannot carry them at all.
- **Two independent directions.**  A bidirectional call sends and
  receives at the same time for minutes.  HTTP/1.1 is request then
  response.
- **Per-stream flow control.**  A client-streaming call that outran its
  server would have nothing to push back with.

So `http-codec-nv` is the right package for HTTP/1.1 and is deliberately
not a dependency of this one.  What is missing from the grid is
**`http2-nv`** — `core`, category `networking`: the frame layer
(HEADERS, DATA, SETTINGS, WINDOW_UPDATE, RST_STREAM, GOAWAY, PING),
HPACK with its dynamic table, the stream state machine, connection and
stream flow control, and the `h2c` and ALPN preludes.  It is a bigger
package than this one and it is not a gRPC package: an HTTP/2 server
that never speaks gRPC needs exactly the same thing.

Above both sits **`grpc-nv`** (`host`, already a row on the grid):
sockets, TLS, a connection pool, retries, deadlines against a real
clock, and the generated stubs.  That is where `std.http` and a clock
appear, and it is the layer that turns the action list this package
produces into bytes on a wire.

Until `http2-nv` exists, a host can drive this package over any HTTP/2
implementation it has — including one written against a C library on
the bindings shelf — because the interface between them is header pairs
and byte chunks and nothing else.

## Which call does what

The four shapes are protobuf's two booleans with names on, and the state
machine enforces the message counts:

| shape | client sends | server sends |
| --- | --- | --- |
| `PbUnary` | one | one |
| `PbServerStream` | one | any number |
| `PbClientStream` | any number | one |
| `PbBidiStream` | any number | any number |

`grpcsvc.request_count` and `.response_count` answer 1 or -1, and -1
rather than a large number so that a state machine comparing against it
cannot be made to pass by a peer sending enough.

## Five things a wrong implementation gets wrong quietly

Each is a test in `tests/grpc_codec_tests.nv`.

**The length is big-endian.**  The protobuf message inside the frame is
little-endian throughout, and a reader who has been staring at one of
those will reach for the wrong helper.

**`M` is minutes and `m` is milliseconds.**  Getting the two the wrong
way round is a call that times out sixty thousand times too late.

**`grpc-message` is percent-encoded.**  It travels in an HTTP/2 header,
so it may hold only printable ASCII; a client that printed the raw
header value would show `caf%C3%A9` for a message a server wrote as
`café`.  `decode_message` is deliberately lenient about a stray `%`,
because refusing to read one would turn a server's bad error string into
a client's transport failure.

**A trailers-only response is a HEADERS frame.**  A server answering
`UNIMPLEMENTED` for an unroutable path sends ONE header set, with
`grpc-status` in it and `end_stream` set.  A client that assumed a
HEADERS frame is the response head will wait forever for a body that is
never coming.  `grpccall.is_trailers_only` is the check.

**A proxy must decrement the deadline.**  `grpc-timeout` is a duration
from receipt, not an instant — which is what lets a call cross a machine
whose clock is wrong — so forwarding one unchanged gives the next hop
the whole budget again, and a chain of five proxies multiplies the
deadline by five.

## A status is not a transport fault

`NOT_FOUND` is a call that worked: the frames parsed, the trailers
arrived, the server answered.  A malformed frame is something else
entirely, and the two are different types here on purpose —
`GrpcTrailers` against `grpcerr.GrpcDecodeError`.  Confusing them is the
most common mistake in gRPC client code, and a package that returned one
type for both would be inviting it.

`grpcstatus.is_retryable` answers true for `UNAVAILABLE` and for nothing
else.  `DEADLINE_EXCEEDED` is false because the deadline may have passed
after the work succeeded, so a retry can do it twice; `ABORTED` is false
because the retry belongs at the level that knows what the transaction
was.

## The layer, and why

`core`.  Everything here is arithmetic over bytes and values the caller
already holds, and no function declares an effect.  There is no socket,
no clock and no compression in this package.

**No device claim.**  gRPC needs HTTP/2, which needs HPACK's dynamic
table and per-stream flow control, and a device that could afford those
would not be using gRPC.  An embedded producer that wants a
request-response protocol over a constrained link should look at
mqtt-codec-nv, or at CBOR over its own framing — cbor-nv carries the
device probe that says so.

## The reference implementation

gRPC's own `PROTOCOL-HTTP2.md` and `grpc/status.proto`, with `tonic`
(Rust, MIT) as the implementation to check against — its codec half is
what the grid's row for this package names.  Every value in
`tests/grpc_codec_tests.nv` is from the protocol document: the request
and response header sets, the `Timeout` grammar with its six units, the
`Status-Code` list, the percent-encoded message rule, and the
HTTP-status mapping table.  The framed message is protobuf's own
`08 96 01` with gRPC's five bytes in front of it.

## Status

| function | implemented |
| --- | --- |
| `grpcerr.decode_offset`, the two `message` impls | no |
| `grpcframe.header_len`, `.max_encodable_len`, `.framed_len` | no |
| `grpcframe.default_limits`, `.small_limits` | no |
| `grpcframe.write_header_into`, `.read_header_at`, `.frame` | no |
| `grpcframe.deframer`, `.feed`, `.finish`, `.deframe_all` | no |
| `grpcframe.offset`, `.pending_len`, `.at_boundary` | no |
| `grpcstatus.status_code`, `.status_of_code`, `.status_name`, `.status_of_name` | no |
| `grpcstatus.is_retryable`, `.status_of_http`, `.status_of_stream_reset`, `.status_of_decode_error` | no |
| `grpcstatus.trailers`, `.ok_trailers`, `.is_ok` | no |
| `grpcstatus.encode_message`, `.decode_message` | no |
| `grpcstatus.status_header`, `.message_header`, `.details_header` | no |
| `grpcmeta.empty`, `.metadata`, `.get`, `.get_all`, `.has` | no |
| `grpcmeta.set`, `.append`, `.remove`, `.entries_of`, `.count` | no |
| `grpcmeta.is_valid_name`, `.is_reserved_name`, `.is_binary_name`, `.is_valid_text_value` | no |
| `grpcmeta.content_type`, `.content_type_for`, `.is_grpc_content_type`, `.message_subtype` | no |
| `grpcmeta.encoding_header`, `.accept_encoding_header`, `.timeout_header` | no |
| `grpcmeta.unit_char`, `.unit_of_char`, `.unit_nanos`, `.max_timeout_digits` | no |
| `grpcmeta.deadline_text`, `.deadline_of_text`, `.deadline_nanos`, `.deadline_of_nanos`, `.remaining` | no |
| `grpcsvc.path_of`, `.parse_path`, `.full_name_of_path_service` | no |
| `grpcsvc.services_of`, `.service_of`, `.method_of`, `.method_of_path`, `.method_desc` | no |
| `grpcsvc.client_streams`, `.server_streams`, `.request_count`, `.response_count` | no |
| `grpcsvc.reflection_service_name`, `.reflection_method_path`, `.health_service_name` | no |
| `grpccall.call`, `.state_of`, `.state_name`, `.half_closed_local`, `.half_closed_remote` | no |
| `grpccall.sent_count`, `.received_count`, `.trailers_of` | no |
| `grpccall.start_request`, `.start_response`, `.trailers_only`, `.finish_response` | no |
| `grpccall.send_message`, `.half_close`, `.cancel` | no |
| `grpccall.on_headers`, `.on_data`, `.on_trailers`, `.on_reset`, `.on_connection_lost` | no |
| `grpccall.request_headers`, `.response_headers`, `.trailer_headers` | no |
| `grpccall.trailers_of_headers`, `.is_trailers_only` | no |
