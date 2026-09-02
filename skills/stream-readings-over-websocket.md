---
name: stream-readings-over-websocket
description: Open the Synse WebSocket API and subscribe to device readings as the plugins produce them — the only Synse surface that pushes, and the only place streamed readings exist.
api: Synse Server API (v3)
generated: '2026-09-02'
method: generated
source: https://synse.readthedocs.io/en/latest/server/api.v3/
operations:
  - WS /v3/connect
  - request/status
  - request/read_stream
  - request/scan
  - request/tags
  - response/reading
  - response/error
---

# Stream readings from Synse over WebSocket

The HTTP and WebSocket APIs expose the same information with one exception that matters:
**streamed readings are WebSocket-only.** The docs state streaming will come to HTTP when
HTTP/2 support allows, but today `request/read_stream` has no HTTP equivalent.

## Connect

```
ws://{synse-server-host}:5000/v3/connect
```

Use `wss://` if the operator configured TLS (`SYNSE_SSL_CERT` / `SYNSE_SSL_KEY`) or put the
instance behind a TLS-terminating proxy, which is what the docs recommend for production.
No credentials are sent — this API has no authentication.

## Message envelope

Every message in both directions has the same three fields:

```json
{
  "id": 0,
  "event": "request/status",
  "data": {}
}
```

- `id` — you assign it and increment it per session. The server reflects it on the response,
  and that is how you match a response to its request. **Track ids yourself; the server does
  not.**
- `event` — `request/*` from you, `response/*` from the server.
- `data` — request parameters by name (what would be URI and query parameters over HTTP;
  POST bodies go under a `payload` key). Omit it entirely when there are none.

## Subscribe

```json
{"id": 1, "event": "request/read_stream", "data": {}}
```

The server then emits `response/reading` messages as the plugins read, each reflecting
`id: 1`. Narrow the stream the same way you narrow a read — by tag. Discover the tag
namespace first:

```json
{"id": 2, "event": "request/tags"}
{"id": 3, "event": "request/scan"}
```

## Errors

Errors arrive as `response/error` carrying the standard envelope
(`http_code`, `description`, `timestamp`, `context`).

**Watch for `id: -1`.** That means the failure happened before your request could be parsed,
so the error cannot be attributed to any message you sent — it is a connection- or
framing-level problem, not a rejection of request 3. Do not retry a specific request in
response to `-1`; fix the message you are sending.

## The full event catalog

Seventeen request events and their responses are documented, covering everything the HTTP
API does: status, version, config, plugin, plugins, plugin_health, scan, tags, info, read,
read_device, read_cache, read_stream, write_async, write_sync, transaction, transactions.
The catalog with HTTP equivalents is in
`asyncapi/vapor-io-synse-websocket-events.yml`.

Note that `request/write_async` and `request/write_sync` are on this connection too — the
WebSocket is not read-only, and writes over it act on hardware exactly as the HTTP ones do.
See `skills/write-to-device-and-track-transaction.md` before sending one.
