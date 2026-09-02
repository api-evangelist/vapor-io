---
name: read-device-telemetry
description: Discover the devices a Synse Server deployment manages and read their current values, using tags to narrow the set.
api: Synse Server API (v3)
generated: '2026-09-02'
method: generated
source: https://synse.readthedocs.io/en/latest/server/api.v3/
operations:
  - GET /test
  - GET /v3/scan
  - GET /v3/tags
  - GET /v3/info/{device}
  - GET /v3/read
  - GET /v3/read/{device}
  - GET /v3/readcache
---

# Read device telemetry from Synse

Synse Server is self-hosted. Every call below goes to the operator's own instance, which
defaults to port 5000: `http://{synse-server-host}:5000`. There is no authentication — do
not send a key, and do not read the absence of a 401 as authorization.

## 1. Confirm the instance is up

```
GET /test
```

Returns `{"status": "ok", "timestamp": "..."}`. This is the liveness endpoint; it is also
what the documented Docker and Kubernetes health probes call.

```
GET /version
```

Returns `version` and `api_version`. Use the returned `api_version` for the path prefix of
every subsequent call rather than hard-coding `v3`.

## 2. Find the devices

```
GET /v3/scan
```

Lists every device the registered plugins have made known to the server. If a plugin was
registered after the last cache build, force a refresh with `?force=true`.

An empty result usually means no plugin is registered, not that there is no hardware. Check
`GET /v3/plugin` and `GET /v3/plugin/health` before concluding a deployment has no devices.

## 3. Narrow with tags, not with pages

Tags are the addressing primitive of this API. `GET /v3/tags` lists the tags in the current
namespace; `GET /v3/scan` and `GET /v3/read` both accept tag filters.

There is no pagination anywhere in this API — no page, limit, offset or cursor parameter is
documented. A collection call returns the whole collection, so narrow with tags before you
call, not after.

## 4. Read

```
GET /v3/read                 # every device matching the tag filter
GET /v3/read/{device}        # one device by id or alias
GET /v3/readcache            # cached readings, bounded by start/end
```

Passing the `id` tag to `GET /v3/read` is functionally equivalent to `GET /v3/read/{device}`.

Use `GET /v3/info/{device}` for the device's full metadata, outputs and capabilities — that
is where you learn the unit and precision a reading is expressed in, and whether the device
can be written to at all.

## Errors

The envelope is `{http_code, description, timestamp, context}` — not RFC 9457 problem
details. Documented codes are 400 (invalid input), 404 (not found), 405 (action not
supported for device) and 500 (server-side error). Some failures return no JSON body: that
class means the application is not ready, not available, or not reachable. See
`errors/vapor-io-problem-types.yml`.

There are no rate limits and no `Retry-After` — but there is no vendor absorbing your load
either, because the server is running on the operator's own hardware.
