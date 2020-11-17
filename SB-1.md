# SB-1 (AMQP Backed Implementation)

**Status**: On Hold

This proposal introduces a method of using a message queue (Anything implementing
AMQP 0.9.1, focusing on RabbitMQ) rather than the internal, custom queue.

## Context

Much of Seabird core has been built around a custom message queue, limited to a
single node. This has made more complex operations harder to implement and led
to less robust (and less scalable) code.

AMQP 0.9.1 is a widely supported protocol that does what we need. Many brokers
have not implemented AMQP 1.0, so it is not a serious consideration at the
moment.

## Goals

- Remove custom internal message queue implementation without changing the
  consumer API
- Replace internal routing with queue-based routing

## Potential Problems

- This loses information about the current streams, such as channels,
  and backend info. This may be acceptable for now and is related to the API refactor in [SB-3](./SB-3.md)

## Essentials of AMQP 0.9.1

Throughout the rest of this proposal many AMQP specific terms will be used. This
section can be used as a short reference for those.

- Exchange - The message ingest. There can be multiple exchanges of different
  types, to be clarified below. Not to be confused with queues.
- Channel - channels can be thought of as a logical connection to RabbitMQ, but
  they are multiplexed onto the actual connection.
- Queue - when a client connects to RabbitMQ, generally they set up queues to
  listen for messages on exchanges. Each exchange type has different ways of
  subscribing.
- Binding - a binding is a rule which routes messages to a queue.
- Exclusive - when the channel is closed, any relevant resources in RabbitMQ
  will also be closed.
- Mandatory - a flag when sending a message to specify that the producer would
  like to be notified if it cannot be delivered to any queues.

### AMQP Exchange Types

- Default - messages must be published to this exchange with a routing key
  matching the queue the message should be routed to. This is a special
  exchange.
- Direct - any message published to a direct exchange will be delivered to one
  subscribed consumer queue. There is no other routing.
- Fanout - any message published to a fanout exchange will be delivered to all
  subscribed consumer queues. There is no other routing.
- Topic - any messaged published to a topic exchange will be delivered to all
  matching consumer queues. Routing is based on "topics" which are dot-separated
  paths like `seabird.message` or "seabird.command.ping". Subscriptions can
  either be exact or use a wildcard (`*` is used for 1 level of topic, `#` can
  be used for multiple levels. `seabird.*` would get `seabird.message` but would
  not get `seabird.command.ping` while `seabird.#` would get both.
- Header - any message published to a header exchange will be delivered to all
  matching consumer queues. Routing is based on key-value pairs. Matching can
  either be "all" or "any" of the headers depending on what is needed.

## Details

There are two types of "requests" in Seabird: request-response and streaming.
Both of these will need to be backed by a message exchange. All of this will be
handled by seabird-core, so the actual implementation of clients should not
change.

These request types refer to the internal implementation of seabird-core.

There are two exchanges, though there is additional communication through the
default exchange:

- `seabird.events` - a "firehose" header exchange. All backend events
  will be submitted to this queue. On startup, seabird-core will attempt to
  create this exchange. All events will have a "type" header which can be
  filtered on. Specific event types may have additional filters, but they should
  be prefixed based on the event name to avoid routing conflicts.
- `seabird.requests` - a "firehose" header exchange. All plugin requests will be
  submitted to this queue. TODO: maybe this should be a topic exchange, with a
  routing key based on the backend ID.

Additionally, seabird-core should set up an alternate exchange for all mandatory
messages so it can determine if a message was properly delivered.

### Request-Response

On an incoming RPC request (as an example "SendMessageRequest") the following
will happen in order:

- seabird-core will create an exclusive queue to be used for the given RPC reply
- seabird-core will send the request to the `seabird.request` exchange with the
  headers "backend" and "type" set to the target backend ID and the message
  type respectively. The ID of the reply queue will be included as a field in
  the request.
- The proper backend will handle the request and submit the response via the
  associated stream.
- On ingest, seabird-core will attempt to send the appropriate response message
  type to the reply queue through the default exchange. If the message fails to
  be delivered, the client has either closed that request or an invalid reply ID
  was used so a warning should be logged.
- The client will receive the final response via the RPC reply queue

Note: this slightly changes the semantics of sent messages - it means proxied
messages will be sent

### Streaming Input (Chat Backend)

- On connection, a new exclusive queue will be opened. Any events sent via this
  connection will have this queue ID attached automatically.
- If an incoming event has an associated response queue, the reply will be sent
  there via the default exchange.
- All events will be submitted to the `seabird.events` exchange.

### Streaming Output (Plugin)

- Simply create a new queue and route relevant messages to it from the
  `seabird.events` exchange.

Note: the first version will still subscribe all plugins to the firehose. A
separate RFC will be submitted relating to filtering.
