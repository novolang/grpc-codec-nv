# grpc-codec-nv

gRPC is a remote procedure call protocol in which one call is one HTTP/2
stream. It is specified in
[gRPC over HTTP/2](https://github.com/grpc/grpc/blob/master/doc/PROTOCOL-HTTP2.md),
and the status codes it ends calls with are gRPC's own `grpc/status.proto`.
This package brings the part of gRPC that is arithmetic over bytes to
novo-lang, with no socket underneath. The stream those bytes travel on is
[http2-nv](https://novo-lang.org/packages/http2-nv)'s, the messages inside
them are [protobuf-nv](https://novo-lang.org/packages/protobuf-nv)'s, and
the client and server that put the three together are
[grpc-nv](https://novo-lang.org/packages/grpc-nv)'s.

**Status: NOT IMPLEMENTED — interface only.** Every function is declared
with its full signature, but every body is a `todo()` that panics when
called. The package is published so its design can be reviewed and
depended on before it is implemented. Version 0.1.0 will be the first
working release.

## What gRPC is

A call is one HTTP/2 stream. The method is named by the stream's `:path`
pseudo-header, which is a slash, the service's fully-qualified name, a
second slash and the method's simple name: `/acme.Orders/Get`. The
request is the headers, then zero or more messages. The response is a
second header set, then zero or more messages, then the **trailers**,
which are a third header set carrying `grpc-status` and `grpc-message`.
The protocol document defines the two under *Requests* and *Responses*.

The status arrives after the body because a server that has streamed nine
of ten responses and then fails has already sent `:status: 200`. HTTP/1.1
carries trailers only in chunked transfer encoding, and not on a request
at all, which is why gRPC is an HTTP/2 protocol.

The unit inside the body is the **length-prefixed message**, defined under
*Length-Prefixed-Message*. It is five bytes of header and then the
message.

| Bytes | Field | Contents |
| --- | --- | --- |
| 1 | compressed flag | 0 for a plain body, 1 for a body compressed with whatever `grpc-encoding` named. The other 254 values are reserved. |
| 4 | length | The body's length in bytes, big-endian. |
| length | body | The message, in the encoding the content type's subtype names. |

The length is big-endian. A protobuf message inside that length is
little-endian throughout, so one buffer holds two numbers read in
opposite directions.

A message may be split across any number of HTTP/2 DATA frames, and
several messages may share one frame. Reading them back is therefore a
state machine over a byte stream rather than a reader of frames. That
state machine is a **deframer**. It is fed the bytes that arrived and
answers the messages that finished.

**Metadata** is gRPC's name for a call's headers, defined under
*Custom-Metadata*. Three rules apply to a name. It is lower case, because
HTTP/2 header names are. A name beginning `grpc-` belongs to the protocol
and an application may not set one. A name ending `-bin` carries arbitrary
bytes, base64-encoded on the wire. Every other name carries printable
ASCII, `0x20` to `0x7e`, and nothing else.

A **deadline** travels in `grpc-timeout` as up to eight digits followed by
one unit character. It is a duration counted from receipt, not an instant,
so the two ends need no agreement about the clock.

| Character | Unit |
| --- | --- |
| `H` | hours |
| `M` | minutes |
| `S` | seconds |
| `m` | milliseconds |
| `u` | microseconds |
| `n` | nanoseconds |

Every call ends with a status, and there are seventeen of them. The
numbers are the wire's: `grpc-status: 5` is `NOT_FOUND`.

| Code | Name | Meaning |
| --- | --- | --- |
| 0 | `OK` | The call succeeded. |
| 1 | `CANCELLED` | The caller gave up. |
| 2 | `UNKNOWN` | A failure the server could not map onto another code. |
| 3 | `INVALID_ARGUMENT` | The arguments are wrong whatever the state is. |
| 4 | `DEADLINE_EXCEEDED` | The deadline passed. |
| 5 | `NOT_FOUND` | The thing asked for is not there. |
| 6 | `ALREADY_EXISTS` | The thing being created is already there. |
| 7 | `PERMISSION_DENIED` | The caller is known and is not allowed. |
| 8 | `RESOURCE_EXHAUSTED` | A quota, a rate limit or a disk ran out. |
| 9 | `FAILED_PRECONDITION` | The system is not in a state where this can be done. |
| 10 | `ABORTED` | A concurrency conflict, such as an aborted transaction. |
| 11 | `OUT_OF_RANGE` | A read past the end. |
| 12 | `UNIMPLEMENTED` | This server does not serve this method. |
| 13 | `INTERNAL` | An invariant the system depends on is broken. |
| 14 | `UNAVAILABLE` | The service is not available right now. |
| 15 | `DATA_LOSS` | Data was lost or corrupted beyond recovery. |
| 16 | `UNAUTHENTICATED` | The caller is not identified. |

This package performs no input or output. It opens no socket, reads no
clock and compresses nothing. A call is a value that takes what arrived
and answers what must be sent, so the same code runs in a client, in a
server, in a proxy and in a capture tool reading a file.

## Install

```
novo pkg add grpc-codec-nv
```

## Example

```novo
use std.bytes
use grpcframe
use grpcstatus
use grpcsvc

fn main() [io]
    // The :path this method is called at, built from the service's
    // fully-qualified name and the method's simple name.
    println(grpcsvc.path_of(".acme.Orders", "Get"))   // /acme.Orders/Get

    // Put gRPC's five bytes in front of one protobuf message: the
    // compressed flag 0, then the length 3 in four big-endian bytes.
    let body = bytes.from_hex("089601") ?? bytes.zeros(0)
    match grpcframe.frame(body, false, grpcframe.default_limits())
        Ok(out) => println(bytes.to_hex(out))   // 0000000003089601
        Err(e)  => println(e.message())

    // Read the same bytes back as they arrive in DATA frames. The
    // deframer answers the messages that finished and how many more
    // bytes the one in progress still needs.
    let wire = bytes.from_hex("0000000003089601") ?? bytes.zeros(0)
    let d = grpcframe.deframer(grpcframe.default_limits())
    match grpcframe.feed(d, bytes.to_byte_list(wire))
        Ok(s)  => println("${s.messages.len()} message, ${s.need} bytes wanted")
        Err(e) => println(e.message())

    // How a call ended. This is the server's answer, not a fault.
    let t = grpcstatus.trailers(GrpcNotFound, "no such order")
    println("${grpcstatus.status_code(t.status)} ${grpcstatus.status_name(t.status)}")
    // 5 NOT_FOUND
```

Build and test with `novo pkg build` and `novo test`. Today `novo test`
fails on purpose: every test reaches a `not implemented: <module>.<fn>`
panic. The tests are the specification the implementation will have to
satisfy.

## What the package contains

| Module | Contents |
| --- | --- |
| `grpcframe` | The five-byte prefix, written and read, the size limits a peer may not exceed, and the deframer that cuts a byte stream into messages. |
| `grpcstatus` | The seventeen status codes, their numbers and their names, the trailers that carry one, the percent-encoding of `grpc-message`, and the tables that map an HTTP status or an HTTP/2 stream reset onto a status. |
| `grpcmeta` | Metadata and the three rules its names follow, the `application/grpc` content type, and the `grpc-timeout` deadline in both directions. |
| `grpcsvc` | The `/package.Service/Method` path, built and parsed, and the service and method descriptors it is built from. |
| `grpccall` | One call as a state machine. What arrived becomes events, and what must be sent becomes actions. |
| `grpcerr` | The two refusal types. `GrpcDecodeError` is what a peer sent that gRPC does not allow, and `GrpcEncodeError` is what this package was asked to write that gRPC does not allow. |

## How to choose an entry point

**`grpccall` is the entry point for a client or a server.** It holds the
whole protocol. Every function on it answers a `GrpcCallStep`: the call
after the step, the events the caller has learned, and the actions the
host must perform on its HTTP/2 stream. A host that walks the action list
in order has implemented gRPC.

**`grpcframe.deframe_all` reads every message in a buffer you already
hold.** Use it for a unary call, where the whole body arrived in one
piece.

**`grpcframe.deframer` handles a body that arrives in pieces.** Feed it
each chunk of DATA with `feed`, then call `finish` when that direction
ends. Use it for any streaming call, and keep one per direction.

**`grpcstatus`, `grpcmeta` and `grpcsvc` are the pieces on their own.** A
program with its own call loop, and a proxy that only routes and
forwards, use them without `grpccall`.

## The rules a user needs

1. **The length is big-endian.** Section *Length-Prefixed-Message*. The
   protobuf message inside that length is little-endian throughout, so
   one buffer holds numbers read in both directions.
2. **The compressed flag says that a message is compressed, not with
   what.** `grpc-encoding` names the algorithm and inflating the body is
   the caller's work. A flag of 1 on a stream that named no encoding is
   `GrpcCompressedWithoutEncoding`.
3. **The size limit is checked before anything is reserved.** A length
   prefix is an allocation instruction from a peer, and four bytes can
   ask for four gigabytes. `GrpcLimits.max_message` is the bound, it
   counts the body alone, and `default_limits` sets it to gRPC's own
   4 MiB, which is 4,194,304 bytes.
4. **Keep one deframer per direction.** A bidirectional call has two
   independent message streams. One deframer over both would interleave
   two halves of two messages.
5. **`M` is minutes and `m` is milliseconds.** Section *Requests*, the
   `Timeout` grammar. The two differ only in case.
6. **A proxy must decrement the deadline.** `grpc-timeout` is a duration
   from receipt, so forwarding one unchanged gives the next hop the whole
   budget again. `grpcmeta.remaining` is that subtraction. It takes the
   elapsed nanoseconds as a parameter, because this package reads no
   clock.
7. **A deadline is spelled in the unit that loses nothing.**
   `deadline_of_nanos(1500000)` answers `1500u` and not `1m`. A value no
   unit spells exactly within eight digits is `GrpcTimeoutTooLong`.
8. **`grpc-message` is percent-encoded.** Section *Responses*. The header
   may carry only printable ASCII, so everything else travels as `%XX`
   over the UTF-8 bytes and `%` itself travels as `%25`. `decode_message`
   keeps a stray `%` as a literal rather than refusing the header.
9. **A trailers-only response is one HEADERS frame.** Section
   *Trailers-Only*. A server answering `UNIMPLEMENTED` for a path it does
   not route sends one header set with `grpc-status` in it and
   `end_stream` set. A client that took that header set for the response
   head waits forever for a body that is never sent.
   `grpccall.is_trailers_only` is the check.
10. **A status is not a transport fault.** `NOT_FOUND` is a call that
    worked: the frames parsed, the trailers arrived, the server answered.
    That is a `GrpcTrailers`. A malformed frame is a
    `grpcerr.GrpcDecodeError`. The two are separate types, and
    `grpcstatus.status_of_decode_error` turns the second into the status
    to close the stream with.
11. **`is_retryable` is true for `UNAVAILABLE` and nothing else.**
    `DEADLINE_EXCEEDED` is false, because the deadline may have passed
    after the work succeeded. `ABORTED` is false, because a retry there
    belongs at the level that knows what the transaction was.
12. **A metadata name is refused, never corrected.** Section
    *Custom-Metadata*. `set` answers `GrpcReservedMetadataName` for a
    `grpc-` or `:` prefix, and `GrpcInvalidMetadataName` for a character
    the protocol does not allow. A name silently lower-cased is a name
    the caller will search for and not find.
13. **The path drops the descriptor's leading dot.** A descriptor spells
    a service `.acme.Orders` and a path spells it `/acme.Orders/Get`.
    `grpcsvc.path_of` drops the dot and
    `grpcsvc.full_name_of_path_service` puts it back.
14. **A streaming direction has a message count of -1.** `request_count`
    and `response_count` answer 1 or -1. A state machine comparing
    against -1 cannot be satisfied by a peer that sends enough messages.
15. **Nothing here writes a byte.** `grpccall` answers a list of
    `GrpcAction` values and the host performs every one of them: send
    these headers, send these bytes, send these trailers, end the stream,
    reset the stream.

## What is not included

- **HTTP/2.** A gRPC call needs frames, HPACK, a stream state machine and
  per-stream flow control, and none of that is gRPC.
  [http2-nv](https://novo-lang.org/packages/http2-nv) is the package for
  it. The interface between the two is header name-value pairs and byte
  chunks, so this package drives over any HTTP/2 implementation a program
  has.
- **Compression.** The flag says that a message is compressed and
  `grpc-encoding` says with what. Choosing an algorithm here would be
  choosing for a protocol whose two ends negotiate one.
- **A clock.** `grpcmeta.remaining` takes the elapsed nanoseconds as a
  parameter, so the deadline arithmetic is a table a test asserts.
- **Timers, retries and a connection.** Nothing here times a call out,
  sends one again or opens anything.
  [grpc-nv](https://novo-lang.org/packages/grpc-nv) is the package for
  those.
- **The reflection and health services.** Three functions name them —
  `grpcsvc.reflection_service_name`, `grpcsvc.reflection_method_path` and
  `grpcsvc.health_service_name` — and neither service is implemented.
  Each is protobuf messages over this framing, which a program holding
  this package can write.
- **A microcontroller claim.** gRPC needs HPACK's dynamic table and
  per-stream flow control. A device with no heap allocator that wants a
  request-response protocol over a constrained link is served by
  [mqtt-codec-nv](https://novo-lang.org/packages/mqtt-codec-nv).

## Related packages

- [grpc-nv](https://novo-lang.org/packages/grpc-nv) is the host half of
  the same protocol, and the other end of this one. It performs the
  actions this package produces: a channel over an endpoint, deadlines
  against a real clock, retries, a server's routing, and the contract a
  generated stub is written against.
- [http2-nv](https://novo-lang.org/packages/http2-nv) is the transport
  that carries a gRPC call. It turns a connection's bytes into HEADERS,
  DATA, TRAILERS and RST_STREAM, and back again. It is not a gRPC
  package: an HTTP/2 server that never speaks gRPC needs exactly the
  same thing.
- [protobuf-nv](https://novo-lang.org/packages/protobuf-nv) is the wire
  format of the messages inside the frames. This package depends on it
  for `PbService`, `PbMethod`, `PbStreaming` and `PbFileSet`, so a caller
  that parsed a descriptor set hands the same values here.
- [http-codec-nv](https://novo-lang.org/packages/http-codec-nv) is
  HTTP/1.1. It cannot carry gRPC: a request may not carry trailers, the
  two directions are not independent, and there is no per-stream flow
  control. It is the package for an HTTP/1.1 program and not for this
  one.
- [mqtt-codec-nv](https://novo-lang.org/packages/mqtt-codec-nv) is a
  request-response protocol for a constrained link, and it builds for a
  microcontroller.

## Tests

```bash
novo test tests/grpc_codec_tests.nv     # 67 tests against the signatures
```

The values the suite asserts against are the protocol document's own: the
request and response header sets, the `Timeout` grammar with its six
units, the `Status-Code` list, the percent-encoded message rule and the
table that maps an HTTP status onto a gRPC status. The framed message is
protobuf's own `08 96 01` with gRPC's five bytes in front of it, so a test
that passes here is bytes any conforming implementation reads the same
way. The status names and numbers are `grpc/status.proto`'s.

The suite also pins the five rules a wrong implementation breaks quietly:
the big-endian length, `M` against `m`, the percent-encoded message, the
trailers-only response, and the deadline a proxy must decrement.

The tests compile today and fail at run, each on the `not implemented:
<module>.<fn>` panic that is its body. That is the expected state of an
interface release. They turn green one at a time as bodies land.

## Implementation status

| Item | Implemented |
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

## Licence

Apache-2.0. See `LICENSE`.

<!-- docs/writing-a-readme.md is the style guide for this page. -->
