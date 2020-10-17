# SB-3 (API Refactor)

**Status**: Under Review

This proposal aims to simplify the API and make it possible to build more
scalable implementations.

## Context

The initial design of seabird works well for an MVP, but there are a number of
design decisions which should be re-examined.

Bidirectional streaming complicates implementations and is error prone.
Additionally, there are a number of endpoints which are complex to maintain
but are seldom (if at all) used.

## Goals

- Change bidirectional streaming to server only
- Add callback endpoints rather than streamed messages
- Remove currently unused endpoints
- Allow for implementation of simple chat backends without a command stream

## Details

### SeabirdService

The current API:

```protobuf
service Seabird {
  rpc StreamEvents(StreamEventsRequest) returns (stream Event);

  // Chat actions
  rpc PerformAction(PerformActionRequest) returns (PerformActionResponse);
  rpc PerformPrivateAction(PerformPrivateActionRequest) returns (PerformPrivateActionResponse);
  rpc SendMessage(SendMessageRequest) returns (SendMessageResponse);
  rpc SendPrivateMessage(SendPrivateMessageRequest) returns (SendPrivateMessageResponse);
  rpc JoinChannel(JoinChannelRequest) returns (JoinChannelResponse);
  rpc LeaveChannel(LeaveChannelRequest) returns (LeaveChannelResponse);
  rpc UpdateChannelInfo(UpdateChannelInfoRequest) returns (UpdateChannelInfoResponse);

  // Chat backend introspection
  rpc ListBackends(ListBackendsRequest) returns (ListBackendsResponse);
  rpc GetBackendInfo(BackendInfoRequest) returns (BackendInfoResponse);

  // Chat connection introspection
  rpc ListChannels(ListChannelsRequest) returns (ListChannelsResponse);
  rpc GetChannelInfo(ChannelInfoRequest) returns (ChannelInfoResponse);

  // Seabird introspection
  rpc GetCoreInfo(CoreInfoRequest) returns (CoreInfoResponse);
}
```

#### Remove Unused Endpoints

While they are useful, the backend and channel introspection methods are not
currently used by any known plugins and are complicating the implementation of
SB-1 without an additional data store.

```protobuf
  // Chat backend introspection
  rpc ListBackends(ListBackendsRequest) returns (ListBackendsResponse);
  rpc GetBackendInfo(BackendInfoRequest) returns (BackendInfoResponse);

  // Chat connection introspection
  rpc ListChannels(ListChannelsRequest) returns (ListChannelsResponse);
  rpc GetChannelInfo(ChannelInfoRequest) returns (ChannelInfoResponse);
```

Note that even though this proposal recommends dropping these methods, ideally
they would be added back in the future when a permanent data store is added to
seabird-core.

### ChatBackendService

The current API:

```protobuf
message ChatEvent {
  string id = 1;

  oneof inner {
    // Seabird-internal event types
    HelloChatEvent hello = 2;
    SuccessChatEvent success = 3;
    FailedChatEvent failed = 4;

    // Messages from the service
    common.MessageEvent message = 5;
    common.PrivateMessageEvent private_message = 6;
    common.MentionEvent mention = 7;
    common.CommandEvent command = 8;
    common.ActionEvent action = 12;
    common.PrivateActionEvent private_action = 13;

    // Channel changes
    JoinChannelChatEvent join_channel = 9;
    LeaveChannelChatEvent leave_channel = 10;
    ChangeChannelChatEvent change_channel = 11;
  }
}

message ChatRequest {
  string id = 1;

  oneof inner {
    SendMessageChatRequest send_message = 2;
    SendPrivateMessageChatRequest send_private_message = 3;
    JoinChannelChatRequest join_channel = 4;
    LeaveChannelChatRequest leave_channel = 5;
    UpdateChannelInfoChatRequest update_channel_info = 6;
    PerformActionChatRequest perform_action = 7;
    PerformPrivateActionChatRequest perform_private_action = 8;
  }
}

service ChatIngest {
  rpc IngestEvents(stream ChatEvent) returns (stream ChatRequest);
}
```

### Add Callback Endpoints for ChatIngest

Currently, all backend chat events are shared in a bidirectional stream.
Backend chat events are sent to the server and the server emits command
events.

Moving callbacks to separate endpoints rather than in an event stream allows
us to scale out handling of events if necessary. Multiple seabird-core hosts
could be handling event ingestion rather than it being limited to one.

#### Separate Endpoints

This portion of the proposal would add additional endpoints to ChatIngest as
follows, to act as a replacement for incoming `ChatEvents`:

```protobuf
rpc SuccessCallback(SuccessEvent)
rpc FailureCallback(FailureEvent)

rpc MessageCallback(common.MessageEvent)
rpc PrivateMessageCallback(common.PrivateMessageEvent)
rpc MentionCallback(common.MentionEvent)
rpc CommandCallback(common.CommandEvent)
rpc ActionCallback(common.ActionEvent)
rpc PrivateActionCallback(common.PrivateActionEvent)

rpc JoinChannelCallback(JoinChannelEvent)
rpc LeaveChannelCallback(LeaveChannelEvent)
rpc ChangeChannelCallback(ChangeChannelEvent)
```

At this point the `ChatEvent` type would no longer be necessary, other than
`HelloChatEvent`, which is replaced by modifying the stream endpoint in the
next section.


#### Single Endpoint

Alternatively, we could keep `ChatEvent`, remove `HelloChatEvent` from the
`oneof`, and add the following endpoint:

```
rpc EventCallback(repeated ChatEvent)
```

This would additionally allow for submitting multiple events at once.

### Simplify Streaming to Only Server-Side

The `IngestEvents` endpoint would be replaced with `CommandStream`, as below.
`StartCommandStreamRequest` would replace `HelloChatEvent`.

```protobuf
rpc CommandStream(StartCommandStreamRequest) returns (stream ChatRequest);
```
