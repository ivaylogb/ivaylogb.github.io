---
author: Ivaylo Bahtchevanov
pubDatetime: 2026-04-15T10:00:00Z
title: Adapters for the diagnostic stack - Braintrust, LangSmith, OpenTelemetry, PostHog
slug: plugging-diagnostic-stack-into-braintrust-and-posthog
featured: false
draft: false
tags: []
description: Connecting the diagnostic stack to Braintrust, LangSmith, OpenTelemetry, and PostHog. Combine inputs across platforms, point the diagnostic agent at the result.
---


## Why this exists

When an LLM-mediated system fails, eval and observability platforms tell you that it failed — a score, an error, a thumbs-down — but understanding *why* still means manually walking traces and event streams.

Eval datasets catch the failures you already know about. The emerging patterns — the ones that matter most — only surface in production traces and user signal, scattered across whichever platforms recorded them.

The [diagnostic stack](https://ivaylogb.github.io/posts/tools-for-diagnosing-llm-system-failures/) — [pluma](https://github.com/ivaylogb/pluma), [agent-researcher](https://github.com/ivaylogb/agent-researcher), and [integration-watcher](https://github.com/ivaylogb/integration-watcher) — reads from those scattered systems and produces structured findings. Each finding carries a claim, a `file:line` citation, an applyable edit with a pre-image guard, and an explicit `applyable: false` exit when no clean single-edit fix exists. Findings are tagged into one of four categories — measurement instrument, interface, context available at decision time, or call sequence — so they group cleanly across runs and tools.

[pluma](https://github.com/ivaylogb/pluma) normalizes those findings against the [shared spec](https://github.com/ivaylogb/agent-diagnosis-spec) and triages them by cross-checking each claim against external signal, which is how the methodology catches its own bugs before a developer does.

The result is faster root-cause analysis, cross-platform synthesis the dashboards can't do on their own, and findings you can ship a fix against.


Four adapters connect this stack to the platforms teams already use: **Braintrust, LangSmith, OpenTelemetry, PostHog**. Each one reads from its platform and produces the same `FailingEvalContainer` shape (v0.2 spec). A single `pluma diagnose-agent` run can pull failing rows from a Braintrust experiment, a LangSmith project, and an OTel trace bundle into one diagnostic agent run. The adapters live at [`src/pluma/integrations/`](https://github.com/ivaylogb/pluma/tree/main/src/pluma/integrations).

---

## Adapter Shape 

- **Pass/fail is decided after fetch, client-side.** Server-side has failing feedback filtering is not durable across the four platforms; the CLI lets you narrow server-side for performance, but the failure decision happens locally against a configurable threshold.
- **The primary scorer is specified explicitly via `--primary-feedback-key` (or `--scorer` for Braintrust).** No auto-detection from a list of likely names. If omitted, the fallback rule is "any feedback key below threshold means the row is failing."
- **Output validates against [`failing-eval-container.schema.json`](https://github.com/ivaylogb/agent-diagnosis-spec/blob/main/spec/v0.2/failing-eval-container.schema.json) before the diagnostic agent sees it.** Every adapter ships a golden fixture validated against the schema as part of the test suite.
- **Source flags are mutually exclusive within a family.** You can combine sources across platforms in one run, but only one experiment per platform.

## Braintrust

Reads a Braintrust experiment by ID or name. Pulls runs, feedback, scorers, metadata. Emits `FailingEvalContainer` with `expected` populated from the dataset row and `agent_revision` auto-resolved from experiment metadata if a git SHA is present.

```bash
pluma diagnose-agent \
  --target-agent ./my-agent \
  --braintrust-experiment-id <exp-id> \
  --scorer correctness \
  --threshold 1.0 \
  --output-file hypotheses.md
```

Adapter source: [`src/pluma/integrations/braintrust/`](https://github.com/ivaylogb/pluma/tree/main/src/pluma/integrations/braintrust)

| Behavior | What it does |
|---|---|
| `--scorer` | Names the primary scorer. If omitted, falls back to "any sub-threshold key fails the row." |
| `--threshold` | Failure threshold for the primary scorer. Default 1.0 (anything less than perfect counts as failing). |
| `score_band` | LOW/MEDIUM/HIGH band computed from the raw score using configurable thresholds. Both raw `score` and `score_band` are emitted. |
| `scorer_signature` | Short hash of scorer metadata (name + version + evaluator model). Distinguishes "same name, different implementation" when grouping across experiments. |
| `agent_revision` | Auto-resolved from `experiment.metadata.git_sha` or `run.extra.git_sha`. Overridable via `--agent-revision`. |

## LangSmith

Two workflows supported through two CLI entry points sharing internals.

**Workflow A — Dataset-Experiment.** Reads from `client.evaluate(...)` experiments. Each run aligns to an Example with reference outputs; `expected` populates from `read_example(reference_example_id).outputs`.

```bash
pluma diagnose-agent \
  --target-agent ./my-agent \
  --langsmith-experiment-id <tracing-session-id> \
  --primary-feedback-key correctness \
  --threshold 1.0 \
  --output-file hypotheses.md
```

**Workflow B — Project-traced production.** Reads from a Project with no experiment boundary. Filters via LangSmith's `and(...)`/`or(...)`-wrapped filter DSL. Reference outputs not present unless surfaced via a feedback key.

```bash
pluma diagnose-agent \
  --target-agent ./my-agent \
  --langsmith-project <project-name> \
  --filter 'and(gt(start_time, "2026-04-01T00:00:00Z"), eq(feedback_key, "correctness"))' \
  --primary-feedback-key correctness \
  --reference-feedback-key reference_answer \
  --output-file hypotheses.md
```

Adapter source: [`src/pluma/integrations/langsmith/`](https://github.com/ivaylogb/pluma/tree/main/src/pluma/integrations/langsmith)

| Behavior | What it does |
|---|---|
| `--primary-feedback-key` | Names the primary scorer. LangSmith does not standardize feedback key names — evaluator implementations choose their own. If omitted, fallback rule applies. |
| `--reference-feedback-key` | Workflow B only. Treats a named feedback key's value as the expected output for the row. |
| `--filter` | LangSmith filter DSL, passed through verbatim. Optional narrowing; never the failure decision. |
| `--max-tree-depth` | Maximum depth the run-tree walker descends per failing run. Default 4. |
| `--max-total-nodes` | Global node budget across the subtree. When exceeded, error-bearing paths are prioritized; sibling leaves drop before ancestor paths. Default 50. |
| `--agent-revision` | Required to populate the field. No auto-resolution (LangSmith has no equivalent convention). |

The run-tree walker is the load-bearing piece for LangGraph agents producing deep call graphs. Without the global budget, ten failing runs with twenty child calls per turn produces ~1000 nodes in the diagnostic prompt.

## OpenTelemetry

Reads OTLP-encoded trace bundles. OTel has no native pass/fail or score concept; the adapter applies a configurable convention.

```bash
pluma diagnose-agent \
  --target-agent ./my-agent \
  --otel-trace-bundle ./traces.otlp \
  --service-filter my-agent-service \
  --output-file hypotheses.md
```

Adapter source: [`src/pluma/integrations/otel/`](https://github.com/ivaylogb/pluma/tree/main/src/pluma/integrations/otel)

| Behavior | What it does |
|---|---|
| Default failure rule | `span.status.code = ERROR` counts as failing. Span attribute `gen_ai.evaluation.score` below threshold counts as failing. Both rules configurable per-deployment. |
| `--service-filter` | Scopes which service's spans count as the agent. Prevents multi-service traces from producing noisy diagnostics. |
| proto3-strict patch | The official Python OTel SDK omits zero-valued integer fields on serialization, which proto3-strict parsers reject. The adapter handles this transparently. Documented at the top of the parser. |

## PostHog

Reads PostHog's event export. Groups events by conversation/run ID. Applies a YAML mapping that translates events to `FailingEval` rows.

```bash
pluma diagnose-agent \
  --target-agent ./my-agent \
  --posthog-event-export ./events.jsonl \
  --posthog-mapping ./posthog-mapping.yaml \
  --output-file hypotheses.md
```

Adapter source: [`src/pluma/integrations/posthog/`](https://github.com/ivaylogb/pluma/tree/main/src/pluma/integrations/posthog)

| Behavior | What it does |
|---|---|
| Event mapping | YAML file. Names which events count as feedback (positive/negative/correction), how they translate to `passed: bool`, what the score is. Different teams instrument PostHog differently — the mapping is the integration surface. |
| Reference mapping shipped | `posthog-mapping.example.yaml` in the repo documents the conventions. Copy, edit, point at it. |
| Conversation grouping | Events with the same `conversation_id` collapse to one `FailingEval` row. The final feedback event in the conversation determines `passed`. |

## Combining sources

A single diagnostic run can pull from multiple platforms. The diagnostic agent receives the union and reasons across them.

```bash
pluma diagnose-agent \
  --target-agent ./my-agent \
  --braintrust-experiment-id <exp-id> \
  --langsmith-project <project-name> \
  --otel-trace-bundle ./traces.otlp \
  --output-file hypotheses.md
```

This is the case where the adapter pattern earns its keep: a team has Braintrust experiments for offline eval, LangSmith for online traces, and OTel from their own instrumentation. The diagnostic agent reads all three, groups by `agent_revision` and `scenario_id` where possible, and produces a single set of hypotheses.

## Related repos

- [pluma](https://github.com/ivaylogb/pluma) — the orchestrator and CLI. All four adapters live here.
- [agent-diagnosis-spec](https://github.com/ivaylogb/agent-diagnosis-spec) — the v0.2 schema every adapter targets.
- [agent-researcher](https://github.com/ivaylogb/agent-researcher) — the diagnostic agent that consumes the unified output.
- [integration-watcher](https://github.com/ivaylogb/integration-watcher) — separate agent for cohort-level integration patterns, reads the same `FailingEvalContainer`.

If you're connecting your own platform, the [adapter directory](https://github.com/ivaylogb/pluma/tree/main/src/pluma/integrations) has four worked examples and the schema spec is short.