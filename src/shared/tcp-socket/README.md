# tcp-socket

A small TCP socket layer for the game. It is designed to replace the parts of
Socket.IO that this project uses, without pulling in Engine.IO, HTTP polling or
the Socket.IO dependency tree.

The module uses Node's built-in `net` module, so it runs in the server process
and in the Electron main process. A plain browser renderer cannot open raw TCP
connections; the Electron main process (or a preload script with Node access)
should own the connection and bridge events to the renderer.

## Files

| File | Purpose |
| --- | --- |
| `protocol.js` | Length-prefixed framing, opcodes, JSON event encoding/decoding |
| `tcp-socket.js` | One connection: events, heartbeat, backpressure, disconnect |
| `tcp-server.js` | Listener, connection tracking, broadcast |
| `tcp-client.js` | Connector, event buffering, automatic reconnect |
| `index.js` | Public exports |

## Wire format

```
+-----------------------+----------+-------------------+
| payload length (u32)  | opcode   | payload (N bytes) |
+-----------------------+----------+-------------------+
```

- Length and opcode header is 5 bytes.
- The length field covers only the payload.
- All integers are big endian.
- Default maximum payload is 1 MiB.

Opcodes:

| Opcode | Name | Purpose |
| --- | --- | --- |
| `0x01` | `EVENT` | JSON `{ event, args }` application message |
| `0x02` | `PING` | Heartbeat request |
| `0x03` | `PONG` | Heartbeat reply |
| `0x04` | `HANDSHAKE` | JSON metadata sent by the client after connecting |

Event payloads look like:

```json
{ "event": "serverTellPlayerMove", "args": [playerData, users, foods, mass, viruses] }
```

Multiple arguments are preserved, which matches Socket.IO's
`socket.emit(event, ...args)` API.

## Server usage

```js
const { createServer } = require('../shared/tcp-socket');

const server = createServer({
    requireHandshake: true,
    handshakeTimeout: 5000
});

server.on('connection', (socket) => {
    const type = socket.handshake.query.type;

    socket.on('gotit', (player) => {
        // ...
        socket.emit('welcome', playerData, gameSizes);
    });

    socket.on('disconnect', (reason) => {
        console.log('client left:', reason);
    });
});

server.listen(3000, '0.0.0.0', () => {
    console.log('listening', server.address());
});
```

`server.emit(event, ...args)` broadcasts to every connection, matching
`io.emit(...)`. `socket.broadcast.emit(...)` sends to everyone except that
socket.

## Client usage

```js
const { createClient } = require('../shared/tcp-socket');

const client = createClient({
    host: '127.0.0.1',
    port: 3000,
    query: { type: 'player' },
    reconnect: true
});

client.on('connect', () => {
    client.emit('gotit', player);
});

client.on('serverTellPlayerMove', (playerData, users, foods, mass, viruses) => {
    // ...
});

client.on('disconnect', (reason) => {
    // ...
});

client.connect();
```

The client buffers application events while connecting or reconnecting.
`client.close()` is an alias for `client.disconnect()`, matching the existing
client code.

## Handshake

Raw TCP has no URL query string, so the client sends a `HANDSHAKE` frame with
`query` and `auth` before any application event. When the server is created with
`requireHandshake: true`, the `connection` event is not emitted until that frame
arrives. The server exposes:

```js
socket.handshake.query
socket.handshake.auth
socket.handshake.address
socket.handshake.port
```

`address` and `port` always come from the transport and cannot be spoofed by
the peer.

## Heartbeat and reconnect

- Both sides send `PING` at `heartbeatInterval` (default 15s) and reply with
  `PONG`.
- A peer that stops responding is disconnected with reason
  `heartbeat timeout` after `heartbeatTimeout` (default 10s).
- Set `heartbeatInterval: 0` to disable the heartbeat.
- The client reconnects with exponential backoff by default. Set
  `reconnect: false` to disable it.

## Events

Reserved local events (not valid application event names): `connect`,
`connection`, `disconnect`, `disconnecting`, `error`, `close`, `drain`,
`handshake`, `ping`, `pong`, `protocolError`, `newListener`, `removeListener`.
The client also has `connecting`, `connect_error`, `reconnect`, `reconnecting`
and `reconnect_failed`.

Remote peers cannot spoof these names; sending one is treated as a protocol
error.

## Notes

- Raw TCP needs its own port. It cannot share Express's HTTP port without a
  protocol-negotiation layer. The future Electron client can connect directly
  to the TCP port, or a WebSocket bridge can be added if a browser build is
  still needed.
- The module has no TLS support yet. If encryption is needed later, wrap the
  listener/connection with Node's `tls` module or add a `tls` option.
- It is not wired into `src/server/server.js` or the client yet. The existing
  Socket.IO code is untouched.
- When the server is built by gulp, remember to add this shared folder to the
  build output before importing it from the server.
