---
name: write-to-device-and-track-transaction
description: Issue a write to a physical or virtual device through Synse Server and follow the resulting transaction to completion. Irreversible — read the state first.
api: Synse Server API (v3)
generated: '2026-09-02'
method: generated
source: https://synse.readthedocs.io/en/latest/server/api.v3/
operations:
  - GET /v3/info/{device}
  - GET /v3/read/{device}
  - POST /v3/write/{device}
  - POST /v3/write/wait/{device}
  - GET /v3/transaction/{transaction}
  - GET /v3/transaction
---

# Write to a device through Synse

**These operations act on real hardware, and nothing in this API takes them back.** Synse
documents no cancel, no undo, no rollback and no restore. A queued transaction cannot be
cancelled; `GET /v3/transaction/{transaction}` reports status only. The only route back is a
compensating write, which the API neither models nor guarantees is possible for a given
device. There is also no authentication, so no permission check will stop you.

## 1. Before you write, capture what you are overwriting

```
GET /v3/info/{device}
```

Read `capabilities` — it tells you the mode and, under `write`, the exact `actions` the
device accepts. Do not attempt an action that is not listed: an unsupported action returns
405, and the set of actions is plugin-defined, not universal.

```
GET /v3/read/{device}
```

This reading is the only record of the state you are about to replace. There is no
prior-state endpoint and no history. Capture it.

## 2. Write

Asynchronous — returns immediately with transaction information:

```
POST /v3/write/{device}
{"action": "<action>", "data": "<data>"}
```

Synchronous — blocks until the write resolves, no polling needed:

```
POST /v3/write/wait/{device}
{"action": "<action>", "data": "<data>"}
```

The body may be a single object or an array of objects. An array means multiple writes to
the same device, **processed in the order given** — `[{"action":"color",...},{"action":"state",...}]`
runs `color` first.

`data` is required only for actions that take it; that too is plugin-defined.

## 3. About the optional transaction id

You may supply your own `transaction` in the payload. If it conflicts with an existing
transaction id, the request errors.

Do not mistake this for an idempotency key. It rejects a duplicate rather than replaying
the original result, so a retry gets an error instead of the first response — and once the
id ages out of the cache, the same id will execute a **second physical write**. Build your
retry logic around checking transaction status, not around resubmitting.

## 4. Follow the transaction

```
GET /v3/transaction/{transaction}    # status of one write
GET /v3/transaction                  # all currently tracked ids
```

Transaction ids are not kept indefinitely. They expire after the deployment's configured
cache TTL, after which this returns 404 — a 404 here means "expired or never existed", not
"failed". Poll before the TTL, or use the synchronous write.

## Over WebSocket

The same three steps are `request/write_async` → `response/transaction_info`,
`request/write_sync` → `response/transaction_status`, and `request/transaction` →
`response/transaction_status`, at `ws://{synse-server-host}:5000/v3/connect`. POSTed bodies
go under the `payload` key of `data`. See `asyncapi/vapor-io-synse-websocket-events.yml`.
