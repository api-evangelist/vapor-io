---
name: check-synse-deployment-health
description: Establish whether a Synse Server deployment is reachable, which plugins are registered, and whether they are healthy — the diagnosis to run before trusting any reading.
api: Synse Server API (v3)
generated: '2026-09-02'
method: generated
source: https://synse.readthedocs.io/en/latest/server/api.v3/
operations:
  - GET /test
  - GET /version
  - GET /v3/config
  - GET /v3/plugin
  - GET /v3/plugin/{plugin}
  - GET /v3/plugin/health
  - GET /v3/scan
---

# Diagnose a Synse deployment

Synse Server routes every read and write to a plugin. If a plugin is not registered or not
reachable, the API stays up and simply reports fewer devices — so an empty result is
ambiguous until you have run these checks.

## 1. Is the server there?

```
GET /test
```

200 with `{"status": "ok", "timestamp": "..."}` means up and ready. This endpoint sits
outside the `/v3/` prefix on purpose, along with `/version`, so it can be called before you
know the API version.

```
GET /version
```

Gives `version` and `api_version`. Use `api_version` to build subsequent paths.

## 2. What is it configured to do?

```
GET /v3/config
```

Returns the **unified** configuration — built-in defaults, then the YAML config file, then
environment variables, each overriding the last. This is the fastest way to see which
plugins the instance was told to register and what the cache TTLs are, without reading the
operator's files.

## 3. Which plugins are registered, and are they alive?

```
GET /v3/plugin           # summary of all registered plugins
GET /v3/plugin/{plugin}  # detail for one
GET /v3/plugin/health    # health summary
```

A plugin that registered and is communicating is **active**; one that failed registration or
communication is **inactive**. Synse marks a plugin inactive after a single failed request
so later requests do not block on it, then retries the connection in the background with
exponential backoff, flipping it back to active once it responds.

Plugin *refresh* — the search for and registration of new plugins — is a different
mechanism from active/inactive, which is only about reachability of an already-registered
plugin. A plugin can be registered and inactive.

## 4. Confirm the device cache reflects reality

```
GET /v3/scan?force=true
```

Forces a cache rebuild rather than serving the last one. Run this after a plugin has just
come up; otherwise a healthy plugin can still look deviceless.

## Reading the result

- Server 200 on `/test`, zero plugins in `/v3/plugin` → the deployment was never configured
  with a plugin. Nothing is broken; nothing is connected.
- Plugins registered but `inactive` in `/v3/plugin/health` → network or plugin failure. The
  server will keep retrying; readings you get meanwhile are stale or absent.
- Plugins active, scan empty → force the cache rebuild before escalating.
- 500 from any operation → check `/v3/plugin/health` first; an inactive plugin is a common
  root cause, and the error envelope's `context` field usually names it.
