---
author: Ivaylo Bahtchevanov
pubDatetime: 2026-03-16T10:00:00Z
title: "Spec design to automate shipping: from OpenAPI to SDK to MCP"
slug: spec-stainless
featured: true
draft: false
tags: []
description: "Used pluma's OpenAPI spec to generate a Python SDK, TypeScript SDK, Go SDK, MCP server, and docs site via Stainless. The description fields in the spec become docstrings, JSDoc, Go comments, and the tool descriptions agents read at runtime."
---
I built [pluma](https://github.com/ivaylogb/pluma) and used its OpenAPI spec to generate five things with Stainless: a Python SDK, a TypeScript SDK, a Go SDK, an MCP server, and a docs site. 

The `description` fields in the spec were the key drivers in Stainless's generation pipeline and became the Python docstrings, TypeScript JSDoc, Go comments, and the tool descriptions agents read through the MCP server. Each text translated into workflows in four places, which changed the discipline of what I needed to write. Stainless made it possible for a spec with clean SDKs and text that directly translates to runtime.

## What the spec describes

pluma is a CLI orchestrator over three tools that diagnose failures in developer-API products. The HTTP API behind the SDKs exposes a single workflow: create a diagnostic job from one or more eval/observability sources (Braintrust, LangSmith, PostHog), poll until it completes, fetch structured findings with `file:line` citations and applyable edit specs. I also built a CLI version, documented [here](https://github.com/ivaylogb/pluma)

The three sister tools each diagnose a different part of the workflow, and contribute findings in a shared schema: [funnel-researcher](https://github.com/ivaylogb/funnel-researcher) (dropoff data → hypotheses about quickstart/SDK/error-catalog issues), [agent-researcher](https://github.com/ivaylogb/agent-researcher) (failing eval → hypotheses about where in the agent's source the failure originated), [integration-watcher](https://github.com/ivaylogb/integration-watcher) (cohort traces → findings about integration patterns). Each emits its own report; pluma's `cross` subcommand correlates findings across tools when they cite the same product surface.

The first credibility test of the methodology is in `examples/stripe/`: Stripe Connect onboarding diagnosed against Stripe's docs, the `stripe-python` source, cohort data drawn from GitHub issues, Stack Overflow, and changelog signals. Two diagnostic lenses converged on the silent-200 gate (where the API returns success but onboarding hasn't actually completed) and surface the docs and error-handling friction that produced actual dropoff.

## The Schema

Every tool, regardless of where it's used, creates a "Finding" which maps to a common schema defined in [agent-diagnosis-spec](https://github.com/ivaylogb/agent-diagnosis-spec) v0.1:

- A **claim** stating the proposed cause
- A **Layer** (1–4) identifying where the cause lives
- A **citation** (`file:line`) into product source
- An **applyable** flag
- An **edit** (if applyable) — a structured spec `apply` can execute

The four layers vary by tool because each focuses on a different surface:

| Layer | funnel-researcher | agent-researcher | integration-watcher |
|---|---|---|---|
| 1 | Funnel definition | Evaluation | Trace definition |
| 2 | API/SDK surface | Tools | API/SDK surface |
| 3 | Docs/Context | Context | Docs/Context |
| 4 | Workflow sequence | Workflow | Integration sequence |


The schema does two things that matter for what comes next. First, it makes all findings mechanically correlatable — same vocabulary, same Layer space, same citation shape. Second, and more importantly: every applyable finding ships an `edit` object with a target `path`, a verbatim `before` string acting as a pre-image guard, and an `after` replacement. The `apply` command runs that edit against the live files without re-calling the model; `iterate` applies every applyable finding from a report against the same baseline, snapshots between each, re-runs the eval, and ranks by measured delta. No winner is declared — the operator picks. The schema is what makes mechanical application possible. Without the verbatim `before` guard, an LLM-suggested edit is just a suggestion; with it, the edit is a transaction.

## From spec to SDK to MCP

The diagnosis tools above are read by humans in terminals. The HTTP API is read by agents writing code against an SDK. The LLM composing API calls is now the primary consumer.

The FastAPI server lives at `src/pluma/api/`; the OpenAPI spec is at `src/pluma/api/openapi.yaml`. From that single file, [Stainless](https://www.stainless.com/) generated SDKs in Python, TypeScript, and Go, plus an MCP server. Every config push creates each artifact directly from the spec.

## Spec choices that ship into the SDKs

The first naive spec had everything flat: `client.jobs.retrieve_findings(job_id)` as a method on the jobs resource, `client.healthz.check()` as its own resource, and auto-generated type names like `SourceBraintrustSource` and `JobRetrieveFindingsResponse`. 
Most of the SDK ergonomics were a function of the spec shape itself.

Three changes here:

**1. Sub-resource pattern.** Findings are a sub-resource of jobs rather than a method on the job resource. 

Stainless config:

```yaml
resources:
  jobs:
    methods:
      create: post /v1/jobs
      retrieve: get /v1/jobs/{job_id}
      cancel: delete /v1/jobs/{job_id}
    subresources:
      findings:
        methods:
          retrieve: get /v1/jobs/{job_id}/findings
```

The generated SDK exposes `client.jobs.findings.retrieve(job_id)` instead of `client.jobs.retrieve_findings(job_id)`. The return type becomes `JobFindings`, rather than `JobRetrieveFindingsResponse`.

**2. Explicit model names.** Map each schema in the spec to a clean name in the Stainless config and you get `BraintrustSource`, `LangSmithSource`, `PostHogSource` in the SDK. Skip that step and Stainless generates `SourceBraintrustSource` and friends from the discriminated-union path. Two lines of Stainless config per type.

**3. Top-level health.** The first spec had healthz as its own resource, producing `client.healthz.check()`. Renaming the resource to `health` produces `client.health.check()`, a small change in YAML but meaningful change in the SDK.

After those three edits, the diagnostics counter dropped to zero and the same SDK shape generated across Python, TypeScript, and Go:

| Operation | Generated SDK call |
|---|---|
| Create a job | `client.jobs.create(sources=[...])` |
| Retrieve a job | `client.jobs.retrieve(job_id)` |
| Cancel a job | `client.jobs.cancel(job_id)` |
| Get findings | `client.jobs.findings.retrieve(job_id)` |
| Health check | `client.health.check()` |

## The MCP server: Code Mode

What Stainless generates for an MCP target isn't one tool per endpoint. It's a TypeScript MCP server that exposes **two** tools: `search_docs` (look up methods, parameters, and types from the SDK's embedded docs) and `execute` (write TypeScript against an initialized SDK client; the server runs it in a sandbox and returns whatever the code returns or logs). The agent doesn't pick from a list of N pre-registered API tools. It just writes code.

The `execute` tool ships this prompt verbatim:

> Runs JavaScript code to interact with the Pluma API. You are a skilled TypeScript programmer writing code to interface with the service. Define an async function named "run" that takes a single parameter of an initialized SDK client and it will be run. [...] Code will run in a container, and cannot interact with the network outside of the given SDK client. Variables will not persist between calls, so make sure to return or log any data you might need later.

Why this design choice matters for pluma specifically:

**1. Composition.** "Create a job, poll until it completes, then fetch the findings" is one `execute` call with a loop:

```typescript
async function run(client) {
  const job = await client.jobs.create({ sources: [...] });
  let status = job.status;
  while (!["completed", "failed", "cancelled"].includes(status)) {
    await new Promise(r => setTimeout(r, 2000));
    status = (await client.jobs.retrieve(job.job_id)).status;
  }
  return await client.jobs.findings.retrieve(job.job_id);
}
```

Tool-per-endpoint MCPs require the agent to chain that as three or more separate tool calls with state carried between them. The orchestration becomes the agent's problem rather than something the agent can express directly. For pluma's create-poll-fetch workflow, Code Mode is the strict win regardless of API size.

**2. Context efficiency.** One MCP tool description plus one short SDK schema costs orders of magnitude less context than N tool descriptions covering every endpoint variant. For five endpoints this is a marginal win; for the APIs Stainless typically generates against (Stripe/Anthropic/OpenAI surfaces with hundreds of methods), it's the only way the agent can actually hold the whole surface in working memory.

The tradeoff here is operational: Code Mode requires sandboxed code execution. By default the sandbox is hosted at a Stainless endpoint: 
`--code-execution-mode=local` runs the code in a local Deno sandbox instead. 

Tool-per-endpoint MCPs run on stdio with no execution environment at all. For prototypes that just need to expose three endpoints, tool-per-endpoint is simpler. For anything with chained operations, Code Mode buys back more than it costs.

Both transports work: `node packages/mcp-server/dist/index.js` starts a stdio server; `--transport=http` launches a streamable HTTP server with the same auth (X-Pluma-Key header) as the API.

## The chain: what survives the generation pipeline

Pluma's `FailingEval` schema has six fields (`eval_id`, `claim`, `category`, `citation`, `applyable`, `edit`). In v0.1 only three are populated meaningfully: `category` is always `measurement_instrument`, `applyable` is always `false`, `edit` is always `null`. The temptation: hide the gap in a changelog. The honest move: put the gap in the schema itself.

```yaml
FailingEval:
  type: object
  x-version-capability:
    v0.1:
      eval_id: populated
      claim: short source-data summary
      category: always measurement_instrument
      citation: populated if source metadata has file:line, else null
      applyable: always false
      edit: always null
    v0.2:
      eval_id: populated
      claim: full diagnostic claim
      category: inferred from structural diagnosis
      citation: populated by structural diagnosis when applicable
      applyable: true when single-edit fix is produced
      edit: populated when applyable is true
```

A custom `x-version-capability` extension plus a v0.1/v0.2 split in every field's `description` string. The capability table also lives at the top of the spec's `info.description`. Here's what that costs and what it buys, traced through the pipeline.

In Python: the field-level descriptions become docstrings on the generated Pydantic model in `src/pluma/types/jobs/failing_eval.py`. An SDK user hovering `FailingEval.category` in their IDE sees "v0.1: always measurement_instrument" without leaving the editor.

In TypeScript: same descriptions, JSDoc instead of docstrings. The MCP server's `search_docs` tool searches against this generated documentation. When an agent calls `search_docs("FailingEval.category")`, the v0.1/v0.2 split is in the response.

In Go: same descriptions, Go doc comments.

And then in the MCP tool descriptions themselves: agents writing code through `execute` get the v0.1/v0.2 split surfaced when they ask "what does this field do?" The honest versioning ships all the way from the OpenAPI file to the agent's working context, without anyone manually copying it into four places.

This is a remarkably simple flow. With a few lines per field, all the downstream consumers (Python user, TypeScript user, Go user, MCP agent) read and update the source with no changelog.

[openapi.yaml in pluma](https://github.com/ivaylogb/pluma/blob/main/src/pluma/api/openapi.yaml)

## What's next

When the tools ship v0.2, every consumer (IDEs, agents, docs) gets the upgrade automatically. agent-diagnosis-spec v0.2 fills in the schema. Structured Finding emit from the diagnosis tools, populated `category` / `citation` / `applyable` / `edit` fields, an `iterate` flow that ranks proposed edits by measured eval delta. 

## Where else is this typically useful? 

- Versioned APIs with partial roll-outs: any team shipping v1 -> v2 where some endpoints/fields land before others
- Feature flags exposed at API layer: when a field's behavior depends on the user's plan tier or flag
- Deprecation paths: "deprecated in v3, removed in v4, use new_field instead"
- Multi-team APIs where one team owns spec and another consumes: platform teams ship the spec, while mobile/web/user-facing/internal agents generate from it
- Agent-facing tool docs: as more APIs get MCP servers, the spec's description field becomes the agent context. Inaccurate or partial descriptions impact agent performance.
- Compliance-sensitive APIs: GDPR-impacted fields, SOC2-attested, live in schema 
