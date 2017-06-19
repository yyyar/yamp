## Yamp - Application Messaging Protocol

**Yamp** is simple and flexible messaging protocol for
realtime client-server applications. It provides custom messages
transfering alongside with request-response pattern. **Yamp** is transport
and format agnostic and may be adopted to any specific needs.

This document describes **Yamp v1.0 Draft**.


## Background

There are lots of messaging protocols out of there like AMQP, Thrift,
RMI, MQTT, Socket.io, XMPP, ZMQ, and others. Many of them
are overcomplicated and heavyweight, others suffer from lack of
features. Some of them (for example, WAMP) designed explecitely for IoT and 
require Router component as a mediator to be presented in a solution, even where
it's redundant. On other hand, protocols like Websockets can't handle 
request-response pattern natively forcing developers to implement their own solutions.

**Yamp** designed to provide application developers with the semantics they need
to implement client-server bi-directional communication with the following features:

- Events: publish/subsribe pattern. It's just that websocket can do.
- Requests: RPC-like functionality (req/res), one party emits message on what other party responds.
- Progressive responses, request canceling and timeouts.
- Pluggable Transports: can be implemented on top of any reliable protocol: TCP, TLS, websocket, or other.
- Pluggable Formats: message body may serialized/parsed with any setialization format.
- Cross-Language: can be easily implemented in any language


## Structure

Basically, **Yamp** operates with frames that are tranfered over specific transport.

```
  frame
  +-----------------------------------------------+
  |                                               |
  |  frame-specific data                          |
  |                                               |
  |  body (user frames only)                      |
  |  +-----------------------------------------+  |
  |  |               binary data               |  |
  |  |      (protobuf, msgpack, json, etc)   <-|--|------ format
  |  |                                         |  |
  |  +-----------------------------------------+  |
  |                                               |
  +-----------------------------------------------+
+---------------------------------------------------+
|                                                   |
|            (tcp/tls, ws/wss, etc...)              | <-- transport
|                                                   |
+---------------------------------------------------+

```

### Transport adapters
From a transport adapter, **Yamp** expects ability to send/receive buffer (bytes array) of data.

```
transport.write(buffer)
buffer = transport.read()
```

## Assumptions

**Yamp** uses network byte order (big-endian).
We use C-alike pseudo language alongside pictures to describe messages structure.

## Frame

Frames is what **Yamp** tranfes through transport.
First byte is frame is frame type, following bytes interpretation are specific to concrete frame type.

```
//
//  0       7                  N
//  +-------++~~~~~~~~~~~~~~ ~ ~ ~
//  | 1     ||
//  +-------++ depends on type
//  | type  ||
//  +-------++~~~~~~~~~~~~~~ ~ ~ ~
//

Frame {

  byte type

  select(type) {
    0x00: system.handshake,
    0x01: system.close,
    0x02: system.ping,

    0x10: event,
    0x11: request,
    0x12: cancel,
    0x13: response
  }
}
```


## System Frames

These frames control internal communication between parties and are not exposed
to user as user messages. While defining user procotol one should not rely on these
frames and use User Messages instead.


### 0x00 system.handshake

Should be first frame after connection establishment client party sends
to server party to identify itself. Server party should respond with the same
handshake (echo) in case it support client, or with `system.close` frame.

```
//
//        0       15
//  ~ ~ ~ +--------+
//        | 2      |
//        +--------+
//        |version |
//  ~ ~ ~ +--------+
//

system.handshake {

  byte[2]      version     // (required)
  
}

```


### 0x01 system.close
Should be sent by any party before gracefully closing connection.
Connection or session drop should follow this message. Reason of connection
drop and recommendation on what action to do depends on code.

Is server party wants client to use
another server with different url it sends code = 'redirect'. 
After this message server should close connection / session.
In this case `message` field should contain string in the following format: 
`<protocol>://<host>:<port>[/uri]`, for example:
`tcp://192.168.0.42:5000` or `ws://hostname:8888/hello/world?msg=goodmorning`

```
//
//        0        7                 23                     N
//  ~ ~ ~ +--------++----------------++------------~ ~ ~ ~--+ 
//        | 1      || 2              || 1 x size            | 
//        +--------++----------------++------------~ ~ ~ ~--+ 
//        | code   || size           || message             | 
//  ~ ~ ~ +--------++----------------++------------~ ~ ~ ~--+ 
//

system.close {

  byte           code = {    // (required)
    0x00: unknown,
    0x01: version_not_supported,
    0x02: timeout,
    0x03: redirect,
  }

  byte[2]        size       // (required)
  string[size]   message    // (optional)

}

```



### 0x02 system.ping
My be periodically sent by any party to another to check if it's alive or
keep underlying connection or session alive.
Each `system.ping` frame with false 'ack' from one party should be followed with corresponding
`system.ping` message from another party with 'ack' field set to 'true' and the 
same payload within defined (reasonable) pediod of time.

```
//
//        0        7         15                     N
//  ~ ~ ~ +--------++--------++------------~ ~ ~ ~--+ 
//        | 1      || 1      || 1 x size            | 
//        +--------++--------++------------~ ~ ~ ~--+ 
//        | ack    || size   || payload             | 
//  ~ ~ ~ +--------++--------++------------~ ~ ~ ~--+ 
//

system.ping {
 
 ack           byte(bool)  // (required)
 byte          size        // (required)
 string[size]  payload     // (optional)

}

```


## User Message Frames

These messages are building blocks for user application protocol.


### Basic Structures

These basic structures are reusable in several user messages so
they are defined separately to avoid duplication.


#### UserMessageHeader

Common header for all user messages.
Contains unique message identifier and uri (name) of user message.

```
//
//        0              127               127+1                    N
//  ~ ~ ~ +-----~ ~ ~-----+ +----------------++------------~ ~ ~ ~--+ ~ ~ ~
//        | 16            | | 1              || 1 x size            |
//        +-----~ ~ ~-----+ +----------------++------------~ ~ ~ ~--+
//        | uid           | | size           || uri                 |
//  ~ ~ ~ +---------------+ +------------------------------~ ~ ~ ~--+ ~ ~ ~
//

UserMessageHeader {

  byte[16]       uid    // (required)

  byte           size   // (required)
  string[size]   uri    // (required)
}

```

#### UserMessageBody

Common body for most of user messages.

```
//
//        0                31                     N
//  ~ ~ ~ +--~~~~~~~~~~~~--++------------~ ~ ~ ~--+ ~ ~ ~
//        | 4              || 1 x size            |
//        +----------------++------------~ ~ ~ ~--+
//        | size           || body                |
//  ~ ~ ~ +-~~~~~~~~~~~~~~-++------------~ ~ ~ ~--+ ~ ~ ~
//

UserMessageBody {

  byte[4]      size   // (required)
  byte[size]   body   // (optional) maximum 4Gb

}
```


### 0x10 event

User message (pub/sub event) with binary body in any serialization format

```
//
//        0                     N                    N+M
//  ~ ~ ~ +-~ ~ ~ ~ ~ ~ ~ ~ ~ ~-++-~ ~ ~ ~ ~ ~ ~ ~ ~ ~-+
//        |                     ||                     |
//        | UserMessageHeader   ||   UserMessageBody   |
//        |                     ||                     |
//  ~ ~ ~ +-~ ~ ~ ~ ~ ~ ~ ~ ~ ~-++-~ ~ ~ ~ ~ ~ ~ ~ ~ ~-+
//

event {

  UserMessageHeader  header  // (required)
  UserMessageBody    body    // (required)

}

```


### 0x11 request

Request from one party to another. Requester should wait until
corresponding `response` message comes to fulfull request. Requester
party should also define some reasonable timeout value after
which requester should provide user with an timeout error.

```
//
//        0                     N                      M
//  ~ ~ ~ +-~ ~ ~ ~ ~ ~ ~ ~ ~ ~-++-~ ~ ~ ~ ~ ~ ~ ~ ~ ~-+
//        |                     ||                     |
//        | UserMessageHeader   ||   UserMessageBody   |
//        |                     ||                     |
//  ~ ~ ~ +-~ ~ ~ ~ ~ ~ ~ ~ ~ ~-++-~ ~ ~ ~ ~ ~ ~ ~ ~ ~-+
//

request {

  UserMessageHeader  header       // (required)
  UserMessageBody    body         // (required)

}
```


### 0x12 cancel

If requester wants to ask requested party to cancel (or stop) processing
previously sent `request` it should use this message. Meaning of cancel
is implementation dependent and may be graceful rollback or just halt.

```
//
//        0                     N               N+128
//  ~ ~ ~ +-~ ~ ~ ~ ~ ~ ~ ~ ~ ~-++---~~~~~~~~-----+
//        |                     ||       16       |
//        | UserMessageHeader   |+---~~~~~~~~-----+
//        |                     ||   request_uid  |
//  ~ ~ ~ +-~ ~ ~ ~ ~ ~ ~ ~ ~ ~-++---~~~~~~~~~----+
//

cancel {

  UserMessageHeader  header       // (required)
  byte[16]           request_uid  // (required) original request id to cancel

}

```


### 0x13 response

Response is message sent by party that previously received `request` and:
- successfully completed request (`done`)
- faced error while processing request (`error`)
- got next progress step, if responder supports progress (`progress`)
- successfully cancelled request if got from requester `cancel` message.

```
//
//        0                     N               N+128    N+128+8             N+128+8+M
//  ~ ~ ~ +-~ ~ ~ ~ ~ ~ ~ ~ ~ ~-++---~~~~~~~~-----++--------++-~ ~ ~ ~ ~ ~ ~ ~ ~ ~-+
//        |                     ||       16       ||   1    ||                     |
//        | UserMessageHeader   |+---~~~~~~~~-----++--------+|   UserMessageBody   |
//        |                     ||   request_uid  ||  type  ||                     |
//  ~ ~ ~ +-~ ~ ~ ~ ~ ~ ~ ~ ~ ~-++---~~~~~~~~~----++--------++-~ ~ ~ ~ ~ ~ ~ ~ ~ ~-+
//

response {

  UserMessageHeader header        // (required)
  byte[16]          request_uid   // (required)

  byte              type = {      // (required)
    0x00: done,
    0x01: error,
    0x02: progress,
    0x03: cancelled
  }

  UserMessageBody   body          // (required)
}

```

## Authors
- Yaroslav Pogrebnyak <yyyaroslav@gmail.com>
- Illarion Kovalchuk <illarion.kovalchuk@gmail.com>

