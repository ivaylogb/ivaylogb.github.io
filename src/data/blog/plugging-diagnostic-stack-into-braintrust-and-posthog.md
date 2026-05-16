---
author: Ivaylo Bahtchevanov
pubDatetime: 2026-04-16T10:00:00Z
title: Adapters for Braintrust and PostHog
slug: plugging-diagnostic-stack-into-braintrust-and-posthog
featured: true
draft: false
tags: []
description: Two small converters that connect Braintrust experiments and PostHog event exports to the diagnostic tools I wrote earlier this week.
---

The diagnostic tools I [wrote about earlier this week](/posts/tools-for-diagnosing-llm-system-failures/) consume three kinds of input: a failing eval scenario, a developer-API funnel with dropoff numbers, or a stream of API call traces from a cohort of integrations. Most teams running production LLM systems already produce that data across Braintrust/LangSmith for evals, in PostHog/Amplitude for funnels, in API gateway logs or OpenTelemetry exporters for traces.

The shapes don't match the diagnostic tools' input formats out of the box, so I wrote a couple of converters. Two so far: one for Braintrust experiments, one for PostHog event exports. Both live in `pluma/src/pluma/integrations/`.

## Braintrust → agent-researcher

[agent-researcher](https://github.com/ivaylogb/agent-researcher) reads a failing eval scenario and outputs hypotheses for why the agent failed. The input it expects is a JSON object listing the scenario, expected output, actual output, score, and scorer name.

Braintrust experiments contain that data. An experiment is a set of scored evaluations where each row has `input`, `expected`, `output`, a `scores` object (scorer name to numeric score), `metadata`, and a timestamp.

The converter takes a Braintrust experiment export, filters out the passing scenarios (default threshold: anything below 1.0 on the primary scorer counts as failing), and writes the remaining rows in agent-researcher's expected shape. The original Braintrust row stays attached to each record, so the diagnostic agent has access to the full scorer breakdown and any metadata when it reasons about the failure.

```bash
python -m pluma.integrations.braintrust.cli \
  --input  experiment.json \
  --output failing_evals.json

pluma diagnose-agent \
  --target-agent your_agent_dir \
  --eval-result  failing_evals.json
```

## PostHog → integration-watcher

[integration-watcher](https://github.com/ivaylogb/integration-watcher) reads a stream of API call traces and finds patterns in how developer integrations get stuck. It expects JSONL with fields for the timestamp, integration identifier, endpoint, request body summary, response status, error code, and latency.

PostHog captures events as JSON objects with `id`, `event`, `timestamp`, `distinct_id`, and a `properties` bag for everything else. A team instrumenting their API gateway to send events to PostHog typically captures method, path, status, error code, and latency in `properties`.

The converter walks PostHog events and writes integration-watcher's JSONL. `distinct_id` becomes `developer_id`, `properties.method` and `properties.path` get joined into `endpoint`, and so on. A real PostHog export mixes API-call events with autocaptured events like `$pageview` and `$identify`; those convert to traces with empty endpoints and status 0 rather than getting filtered out. Cohort scope downstream in integration-watcher decides what to keep.

```bash
python -m pluma.integrations.posthog.cli \
  --input  posthog_events.json \
  --output traces.jsonl

pluma watch \
  --traces  traces.jsonl \
  --cohort  your_cohort.yaml \
  --product your_product_dir
```

## Other platforms with similar shapes

LangSmith runs are shaped similarly to Braintrust experiments with the same input/expected/output/score primitives. OpenTelemetry spans for HTTP calls carry the same fields PostHog captures for API-call events. Amplitude funnel exports look similar to the funnel definitions [funnel-researcher](https://github.com/ivaylogb/funnel-researcher) reads.

Each is a converter of similar size and shape to the two above.

Both converters live at [pluma](https://github.com/ivaylogb/pluma) under `src/pluma/integrations/`.
