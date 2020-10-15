# SB-1 (AMQP Backed Implementation)

**Status**: Under Review

This proposal introduces a method of using a message queue (Anything implementing
AMQP 0.9.1, focusing on RabbitMQ) rather than the internal, custom queue.

## Context

Much of Seabird core has been built around a custom message queue, limited to a
single node. This has made more complex operations harder to implement and led
to less robust code.

## Goals

- Remove custom internal message queue
- Replace as much logic as possible with queue routing

## Unsolved Problems

- This loses information about the current streams, such as channels,
  and backend info.

## Details

There are two types of "requests" in Seabird: request-response and streaming.
Both of these will need to be backed by a message queue. All of this will be
handled by seabird-core, so the actual implementation of clients should not
change.

These request types refer to the internal implementation of seabird-core.

Queues will come in three forms:

- `seabird.reply.*`, a separate queue per RPC request, an exclusive, direct
  queue. This could potentially have a name provided by rabbit.
- `seabird.command.*`, a separate queue per backend connection, an exclusive,
  direct queue. The usage will be described under request-response.
- `seabird.events`, a single "firehose", header exchange. All backend events will
  be submitted to this queue. On startup, seabird-core will attempt to create
  this queue.

### Request-Response

On an incoming RPC request (as an example "SendMessageRequest") the following
will happen in order:

- seabird-core will create an exclusive, direct queue to be used for
  the given RPC reply
- seabird-core will send the request to the proper `seabird.command.*` queue,
  including the response queue name.
- The proper backend will handle the request, submitting the event to the
  associated stream.
- On ingest, seabird-core will attempt to send the appropriate response message
  type to the reply queue. If the queue does not exist, the client has closed
  that request, so a warning should be logged.
- The client will receive the final response via the RPC reply queue

Note: this slightly changes the semantics of sent messages - it means proxied
messages will be sent

### Streaming Input (Chat Backend)

- On connection, a new exclusive, direct queue will be opened. Any events sent
  via this connection will have this queue ID attached automatically.
- If an incoming event has an associated response queue, the reply will be sent
  there.
- All events will be submitted to the global event exchange.

### Streaming Output (Plugin)

- Simply create a new queue and route relevant messages to it from the
  `seabird.events` exchange.

Note: the first version will still subscribe all plugins to the firehose. A
separate RFC will be submitted relating to filtering.
