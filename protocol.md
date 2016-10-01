# Logux Protocol

Logux protocol is used to synchronize events between logs.

This protocol is based on simple JS types: boolean, number, string, array
and key-value object.

You can use any encoding and any low-level protocol: binary [MessagePack]
encoding over WebSockets, JSON over AJAX with HTTP persistent connection,
XML over TCP. Main way to send Logux messages is MessagePack over WebSockets.

But low-level protocol should guarantee messages order and content.

[MessagePack]: http://msgpack.org/

## Versions

Protocol uses two major and minor number as version.

```
[number major, number minor]
```

If other client uses bigger `major`, you should send `protocol` error
and disconnect.

## Messages

Communication is based on messages. Every message is a array with string
in the beginning and any types next:

```
[
  string type,
  …
]
```

First string in message array is a message type. Possible types:

* `error`
* `connect`
* `connected`
* `ping`
* `pong`
* `sync`
* `synced`

If client received unknown type, it should send `protocol` error
and continue communication.

## `error`

Error message contain error reason and error type.

```
[
  "error",
  string message,
  string errorType
]
```

Right now there are 2 possible error types: `protocol` and `auth`.

## `connect`

After connection was started one client should send `connect` message to other.

```
[
  "connect",
  number[] protocol,
  string host,
  number synced,
  (any credentials)?
]
```

Receiver should check protocol version in second position in message array.
If major version is different from receiver protocol, it should send protocol
error and close connection.

Third position contains unique host name. Same host is used in default
log timer. This is why sender should use unique host name.
Client should UUID if it can’t guarantee host name uniqueness with other way.

Fourth position contain receiver last `added` time used in previous connection.
If it is first connection, use `0`. Right after `connected` answer receiver will
use `synced` to send all new events since last connection. If it is first
connection, receiver should send all events from log.

Fifth position is optional and contains credentials data. It could has any type.
Receiver may check credentials data. On wrong credentials data receiver may
send `auth` error and close connection.

## `connected`

This message is answer to received `connect` message.

```
[
  "connected",
  number[] protocol,
  string host,
  [number start, number end],
  (any credentials)?
]
```

`protocol` and `host` positions are same with `connect` message.

Fourth position contains `connect` receiving time and `connected` sending time.
Time should be a milliseconds elapsed since 1 January 1970 00:00:00 UTC.
Receiver may use this information to calculate difference between sender
and receiver time. It could prevent problems if somebody have wrong time
or wrong time zone. Fix calculated by this information may be used to correct
events `created` time in `sync` messages.

Fifth position is optional and contains credentials data. It could has any type.
Receiver may check credentials data. On wrong credentials data receiver may
send `auth` error and close connection.

Right after this message receiver should send `sync` message with all new events
since last connection. If it is first connection, sender should send all events
from its log.

## `ping`

Client could send `ping` message to check connection.

```
[
  "ping",
  number synced
]
```

Message array contains also sender last `added`. So receiver could update it
to use in next `connect` message.

Receiver should send `pong` message as soon as possible.

## `pong`

`pong` message is a answer to `ping` message.

```
[
  "pong",
  number synced
]
```

Message array contains sender last `added` too.

## `sync`

This message contains new events for synchronization.

```
[
  "sync",
  number synced
  (object event, array created)+
]
```

Second position contain biggest sender `added` time from events in message.
Receiver should send it back in `synced` message.

This event array length is dynamic. For each event sender should use 2 position:
for event object and event’s `created` time.

Event object could contains any key and values, but it must contain at least
`type` key with string value.

[`created` time] is a array with number or strings. It actual format depends
on used timer. For example, standard timer generated:

```
[number milliseconds, string host, number orderInMs]
```

Sender and receiver should use same timer type to have same time format.

Every events should have unique `created` time. If receiver’s log already
contains event with same `created` time, receiver must silently ignore
new event from `sync`.

Received event’s `created` time may be different with sender’s log,
because sender could correct event’s time based on data from `connected`
message. This correction could fix problems when some client have wrong
time or time zone.

[`created` time]: https://github.com/logux/logux-core#created-time

## `synced`

`synced` message is a answer to `sync` message.

```
[
  "synced",
  number synced
]
```

Receiver should mark all events with lower `added` time as synchronized.