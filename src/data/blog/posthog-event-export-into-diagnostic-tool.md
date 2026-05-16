---
author: Ivaylo Bahtchevanov
pubDatetime: 2026-03-125T18:00:00Z
title: A PostHog event export, fed into a diagnostic tool
slug: posthog-event-export-into-diagnostic-tool
featured: false
draft: false
tags: []
description: PostHog produces an event export in one JSON shape. integration-watcher consumes an event stream in a different JSONL shape. I wrote a converter between them.
---

integration-watcher is one of the diagnostic tools I [wrote earlier this week](/posts/tools-for-diagnosing-llm-system-failures/). It reads a stream of API call traces from a cohort of developer integrations and outputs structured findings about where integrations are getting stuck. It expects a specific input shape — JSONL, one call per line, with these fields:

```
timestamp        ISO 8601
developer_id     string
endpoint         "METHOD /path"
request_summary  compact string repr
response_status  HTTP status code
error_code       string or null
latency_ms       integer
```

PostHog produces an event export in a different shape. The relevant endpoint (`GET /api/projects/:id/events`) returns events as a JSON array, each with `id`, `event`, `timestamp`, `distinct_id`, and a `properties` object that holds everything else.

I wrote a converter between the two.

## The mapping

```
integration-watcher field   PostHog source
─────────────────────────   ─────────────────────────────────────
timestamp                   timestamp
developer_id                distinct_id
endpoint                    "{properties.method} {properties.path}"
request_summary             compact repr of properties.request_body
response_status             properties.status_code
error_code                  properties.error_code (null when absent)
latency_ms                  properties.latency_ms (0 when absent)
```

A PostHog event for an API call typically captures `method`, `path`, request body, response status, error code, and latency in `properties`. Those are the fields you'd set if you were instrumenting an API gateway or middleware to send events to PostHog. The converter walks the events, pulls those properties out, and writes integration-watcher's expected JSONL.

## Example: one event in, one trace out

A PostHog event from the fixture:

```json
{
  "id": "0190e0c2-7f3a-91d2-0003-000000000099",
  "event": "API call",
  "timestamp": "2026-05-10T09:12:03Z",
  "distinct_id": "dev_7f3a91",
  "properties": {
    "method": "post",
    "path": "/v1/auth/token",
    "request_body": {
      "client_id": "cli_7f3a91",
      "grant_type": "client_credentials"
    },
    "status_code": 200,
    "latency_ms": 240,
    "$lib": "posthog-python",
    "$lib_version": "3.6.0",
    "$ip": "203.0.113.41",
    "service": "api-gateway"
  }
}
```

The corresponding line in the converted JSONL:

```json
{"timestamp": "2026-05-10T09:12:03Z", "developer_id": "dev_7f3a91", "endpoint": "POST /v1/auth/token", "request_summary": "client_id=\"cli_7f3a91\" grant_type=\"client_credentials\"", "response_status": 200, "error_code": null, "latency_ms": 240}
```

50 events in, 50 traces out. Deterministic — running the converter twice produces byte-identical output, which makes the golden output a usable diff test for changes to the converter.

## Non-API events

A real PostHog export usually mixes API-call events with `$pageview`, `$identify`, and other autocaptured events that have no method/path. The converter doesn't filter — those non-API events convert to a trace with an empty endpoint and status 0. The decision about which events to keep is downstream, in integration-watcher's cohort scope. Filtering at the converter would silently drop rows; the converter is supposed to be a shape transformation, not a content filter.

## What this gets you

Once the conversion is run, integration-watcher consumes the output the same way it consumes any other trace stream. The diagnostic tool doesn't know or care that the traces originally came from PostHog. Point it at the converted JSONL, a cohort definition, and a directory of product artifacts:

```bash
pluma watch \
  --traces  posthog_converted.jsonl \
  --cohort  your_cohort.yaml \
  --product your_product_dir \
  --output-file findings.md
```

And you get back structured findings about where the integrations in the cohort got stuck — with file:line citations into the product artifacts and applyable edits where edits are justified. The same output shape covered in [the previous post](/posts/tools-for-diagnosing-llm-system-failures/).

PostHog handles the data layer: capturing the events, defining the cohort, producing the export. The diagnostic tool reads what PostHog produces and tells you where in the product surface to look. The shapes line up because both are operating on the same primitive, a stream of API calls with the same fields anyone would want to know about a call.

The converter lives at `src/pluma/integrations/posthog/` in [pluma](https://github.com/ivaylogb/pluma).