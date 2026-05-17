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

The diagnostic tools I [wrote about earlier this week](https://ivaylogb.github.io/posts/tools-for-diagnosing-llm-system-failures/) consume any of the following types of input: a failing eval scenario, a developer-API funnel with dropoff numbers, or a stream of API call traces from a cohort of integrations. Most teams running production LLM systems already produce that data across Braintrust/LangSmith for evals, in PostHog/Amplitude for funnels, in API gateway logs or OpenTelemetry exporters for traces.

This post covers existing converters: one for Braintrust experiments, one for PostHog event exports, one for OpenTelemetry trace exports. All live in `pluma/src/pluma/integrations/`.

## Braintrust → agent-researcher

[agent-researcher](https://github.com/ivaylogb/agent-researcher) reads a failing eval scenario and outputs hypotheses for why the agent failed. The input it expects is a JSON object listing the scenario, expected output, actual output, score, and scorer name.

Braintrust experiments contain that data. An experiment is a set of scored evaluations where each row has `input`, `expected`, `output`, a `scores` object (scorer name to numeric score), `metadata`, and a timestamp.

The converter takes a Braintrust experiment export and writes the failing rows in agent-researcher's expected shape. The original Braintrust row stays attached to each record, so the diagnostic agent has access to the full scorer breakdown and metadata when it reasons about the failure.

Four things the converter preserves beyond the basic field mapping:

**Spans.** Braintrust experiments carry the trace of what the agent did, prompt sent, model response, tool calls, and intermediate reasoning. The converter keeps these on each failing record (capped at 50 spans per row by default, with an explicit truncation marker if exceeded). agent-researcher reads the spans alongside the source code, so its hypotheses can localize the failure to a specific step of the agent's execution rather than just a file:line in source.

**Per-scorer signature.** Real Braintrust experiments have multiple scorers per row — exact_match, factuality, calibrated_confidence, custom LLM-judges. Which scorers a row fails is itself diagnostic signal (a row that fails factuality but passes exact_match is a different bug than one that fails exact_match but passes factuality). The converter emits a `scorer_signature` dict with per-scorer pass/fail booleans, so the diagnostic agent can reason over the pattern.

**agent_revision.** If the Braintrust experiment metadata includes a git SHA tagging which version of the agent was scored, the converter resolves it and attaches it to the failing-eval container. agent-researcher then cites the right revision when proposing file:line edits — no drift between "experiment ran on commit X" and "diagnostic agent reads commit HEAD."

**Optional pre-clustering.** Most Braintrust experiments have either one systemic failure (one mechanism, many rows manifesting it) or scattershot noise (many independent mechanisms, one row each). Running agent-researcher independently on 30 rows that share a root cause wastes 29 model calls. The converter has an optional `--cluster` pre-pass that collapses equivalent failures (by scorer signature and expected/predicted pair) and feeds one row per cluster downstream, with cluster size attached.

The default invocation:

\`\`\`
python -m pluma.integrations.braintrust.cli \\
  --input  experiment.json \\
  --output failing_evals.json

pluma diagnose-agent \\
  --target-agent your_agent_dir \\
  --eval-result  failing_evals.json
\`\`\`

That's two commands. The adapter is also wired directly into Pluma's main CLI, so the same flow collapses to one:

\`\`\`
pluma diagnose-agent \\
  --target-agent your_agent_dir \\
  --braintrust-experiment-id <experiment-id> \\
  --output-file hypotheses.md
\`\`\`

This pulls the experiment live from Braintrust's API, converts it, and runs agent-researcher in one step. Or `--braintrust-project <name> --latest` for the most recent experiment in a project. The two-step file-mode path stays available for CI environments where the experiment JSON is already on disk.

## PostHog → integration-watcher

[integration-watcher](https://github.com/ivaylogb/integration-watcher) reads a stream of API call traces and finds patterns in how developer integrations get stuck. It expects JSONL with fields for the timestamp, integration identifier, endpoint, request body summary, response status, error code, and latency.

PostHog captures events as JSON objects with `id`, `event`, `timestamp`, `distinct_id`, and a `properties` bag for everything else. A team instrumenting their API gateway to send events to PostHog typically captures method, path, status, error code, and latency in `properties`.

The converter walks PostHog events and writes integration-watcher's JSONL. `distinct_id` becomes `developer_id`, `properties.method` and `properties.path` get joined into `endpoint`, and so on. A real PostHog export mixes API-call events with autocaptured events like `$pageview` and `$identify`; those convert to traces with empty endpoints and status 0 rather than getting filtered out. Cohort scope downstream in integration-watcher decides what to keep.

\`\`\`
python -m pluma.integrations.posthog.cli \\
  --input  posthog_events.json \\
  --output traces.jsonl

pluma watch \\
  --traces  traces.jsonl \\
  --cohort  your_cohort.yaml \\
  --product your_product_dir
\`\`\`

## OpenTelemetry → integration-watcher

Anyone exporting OTel-shaped traces from their observability stack (Datadog, Honeycomb, Tempo, Jaeger, Grafana Cloud, AWS X-Ray, Lightstep, Splunk, etc) can feed integration-watcher directly without a designated adapter.

The converter handles three input shapes: OTLP/JSON (the OTLP HTTP/JSON export spec), Jaeger trace exports (the `data` + `spans` structure), and bare OTel span arrays (the minimal "trace data only" export). The format is auto-detected from the file's top-level shape; an explicit format flag is also available.

It also handles both pre-1.21 and post-1.21 HTTP semantic conventions. OTel renamed `http.method` → `http.request.method`, `http.status_code` → `http.response.status_code`, and `http.url` → `http.url.full` in the 1.21 spec. Both variants produce equivalent output records, so an integration-watcher run doesn't have to know what version of OTel produced the source data.

\`\`\`
python -m pluma.integrations.otel.cli \\
  --input  otel_traces.json \\
  --output traces.jsonl

pluma watch \\
  --traces  traces.jsonl \\
  --cohort  your_cohort.yaml \\
  --product your_product_dir
\`\`\`

In the worked verification in the repo, the test fixture deliberately encodes a "developer stuck in a 401 loop" pattern across 8 consecutive HTTP calls. Running the converter on it and then `integration_watcher.analyze_cohort` on the output reconstructs the pattern as `longest_consecutive_same_error == ("401", 8)`. The diagnostic signal survives the conversion end-to-end.

## LangSmith

LangSmith runs are shaped similarly to Braintrust experiments — the same input/expected/output/feedback primitives, plus richer run-tree structure for multi-step agents. That adaptor will be added here soon with the latest updates.

## A public input contract

The diagnostic tools each consume a specific input shape: failing-eval scenarios for agent-researcher, funnel + dropoff data for funnel-researcher, trace cohorts for integration-watcher. Until recently those shapes were implicit in the tools' loader code — anyone writing a new adapter had to read the Python source to figure out what to emit. v0.2 of [agent-diagnosis-spec](https://github.com/ivaylogb/agent-diagnosis-spec) lifts the first of those input contracts (the failing-eval scenario shape) into a public JSON Schema. `FailingEval` and `FailingEvalContainer` are now documented; adapters can target a documented contract rather than reverse-engineer the loader. The other two input contracts (FunnelDropoff for funnel-researcher, TraceCohort for integration-watcher) are named in the spec's roadmap and lifted in a future release when there are multiple adapter implementations to validate the shape against.

## In CI and in Claude Code

A [GitHub Action template](https://github.com/ivaylogb/pluma/tree/main/templates/github-action) for teams running Pluma diagnosis in CI. 

Triggers on Braintrust experiment completion (via webhook), manual workflow dispatch, or a reusable workflow call from a CI job. Posts findings to the PR as a comment or opens an issue.

A [Claude Code skill](https://github.com/ivaylogb/skill-diagnose-agent-failure), `diagnose-agent-failure`. Clone it into a project's `.claude/skills/` directory or your user-scoped `~/.claude/skills/`, and Claude Code auto-invokes it when the conversation mentions a failing eval. The skill resolves the failure shape, runs `pluma diagnose-agent`, and surfaces findings inline without applying edits or re-running the eval silently.

All adapters live at [pluma](https://github.com/ivaylogb/pluma) under `src/pluma/integrations/`