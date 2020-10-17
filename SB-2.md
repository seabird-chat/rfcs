# SB-2 (Pluggable Transport)

**Status**: Under Review

This proposal aims to introduce a pluggable transport to seabird-core and
introduce an alternative to gRPC.

## Context

All of the seabird APIs have been built around gRPC and protobufs. This is a
format which is backed by Google for RPC servers and includes code for client
and server autogeneration. Protobufs have a number of rough edges because of the
various limitations. However, even though this works fairly well for
server-to-server communication, it still falls short for web applications.

## Dependencies

- [SB-1 (AMQP Backed Implementation)](./SB-1.md) - SB-2 can be completed
  without [SB-1](./SB-1.md), but it would be much easier to do this after
  there's a simpler implementation.
- [SB-3 (API Refactor)](./SB-3.md) - this refactor would allow us to only
  support server streams and would simplify this implementation more.

## Goals

- Introduce a new transport
- Allow multiple transports to be running at once, selectable by the backend or
  plugin.
- Continue to be backwards compatible

## Potential Problems

- How to best handle bidirectional streaming/chat ingest
  - The current plan is to use HTTP/1.1 Chunked Encoding (or another similar
    technology) once the API only requires server streaming and not ingestion
    streaming.

## Details

The main complication with other transports relates to bidirectional streaming.
Currently, the only APIs which use bidirectional streaming are the chat
backends. Because most users only implement plugins, it may be acceptable to add
additional complexity there and drop bidirectional streaming.

### Data Format

In general, either JSON (with `application/json`) or msgpack (with
`application/msgpack`) should be used for the serialization/deserialization.

#### Replacing anyof

Essentially, anyof is a tagged union. The simplest way to decode this in Go is
to have a wrapper struct with `type` and `payload` with the `payload` being a
`json.RawMessage`. Some msgpack libraries have an alternative to this. With
minimal glue, it should be possible to create a `RawJsonOrMsgpackMessage` which
would let us deserialize to either type trivially. Unfortunately, this also
means a two-step encoding process, so additional wrappers would need to be
written.

For rust, serde supports tagged unions out of the box with `#[serde(tag =
"type", payload = "payload")]`.

### Endpoints

#### Request/Response

Most endpoints are fairly straight-forward. Request-response over HTTP will
react fairly similarly no matter if you're using the data format in this
proposal or gRPC/protobufs.

#### Chat Ingest

Because Chat Ingest is bidirectional, it doesn't cleanly map onto HTTP.

Thankfully there are other options:
- websockets (which require server support and
additional wrappers for reading auth from query params but support web cleanly
and matches the current paradigm)
- HTTP Multipart (which also requires some
additional wrappers to properly support streaming the form data)
- HTTP/2 (which
requires server support and has relatively limited library support, especially
on the frontend)
- HTTP/1.1 chunked encoding (though this would require having
additional ingestion endpoints as it would only be usable for events streaming
from the server).

See https://github.com/twitchtv/twirp/issues/3

#### Event Stream

Because the event stream is only reading events (and not bidirectional), it will
also map onto the same pattern as chat ingest, but without the additional work
to support sending events.
