---
author: Ivaylo Bahtchevanov
pubDatetime: 2026-03-15T10:00:00Z
title: Tools I built for diagnosing LLM-mediated system failures
slug: tools-for-diagnosing-llm-system-failures
featured: true
draft: false
tags: []
description: A few months of work on tools for figuring out why an LLM-mediated system failed, what each does, how they ended up sharing a structure, and what happened when I ran them against a real product surface.
---

Over the last few months I've been writing tools for diagnosing failures in LLM-mediated systems. They started as one-offs for different problems I was running into. They ended up convering on a single orchestrator service, here's a brief description of what's in each.

## The first tool: agent-researcher
agent-researcher consumes failing eval scenarios, ie the kind of output LangSmith or Braintrust produce.
The starting point is an eval failing on a routing classifier. For example, take a scenario tagged `bug` was getting predicted as `unknown`.

[agent-researcher](https://github.com/ivaylogb/agent-researcher) would take the failing eval and output three structured hypotheses. Each hypothesis has:

- A claim about why the agent failed.
- A citation into a file at a line range, `path:start-end`.
- A proposed code edit, formatted as a structured artifact with the pre-edit content the editor expects to find (used as a guard: if the file at that path doesn't match, the edit is refused rather than applied silently).
- A verification step: apply the edit, re-run the eval, report the delta on a named metric.

Sometimes it can't propose a clean single-file edit. In that case it returns `applyable: false` with specific reason. The honest "I don't have a fix for this" output is very useful here. On the `bug`/`unknown` scenario it emitted H1, H2, H3. I applied H1 and H2 with the verifier; pass rate went from 6/7 to 7/7 under H1 and stayed at 6/7 under H2. H2's prose read more confident than H1's, but the verifier didn't care about that. The H1/H2/H3 run is the worked example in `examples/issue_107/` in the repo.

## funnel-researcher

A different problem here: a developer-facing API where activation rates were stalling at a particular step in the onboarding funnel. I had funnel definitions and dropoff numbers, but I needed something that could read those alongside the product artifacts (docs, SDK, error catalog) and tell me where in the surface the friction was actually living.

[funnel-researcher](https://github.com/ivaylogb/funnel-researcher) takes a funnel definition, dropoff data, and a directory of product artifacts. It outputs structured hypotheses about why developers fall off at a named target step. Same shape as agent-researcher's findings: a claim, a `path:start-end` citation, an applyable edit (often into a doc file rather than a source file), and an explicit `applyable: false` when the fix isn't a clean single-edit. The failure didn't depend on whether system being diagnoses was an agent or dev API. The structure of "here's a defect, here's where to find evidence, here's an edit that would address it, here's how you'd verify" was the same.

## integration-watcher

A third problem. A cohort of developer integrations against an API, and a question like "where in the call sequence are these integrations getting stuck?" Input call traces (in this case JSONL streams of API calls, one line per call, with timestamps and request/response shapes) and a watch question/criteria. This provides something able to read the traces alongside product artifacts and identify what patterns developers call into.

[integration-watcher](https://github.com/ivaylogb/integration-watcher) does that. 
Inputs: trace stream, cohort definition, watch question, product artifact directory. 
Outputs: structured findings about patterns in the cohort, with prevalence numbers ("3 of 7 integrations" etc.), trace citations, and the same applyable/`applyable: false` exit.

By the time I'd built the third tool, I noticed something I hadn't planned for.

## agent-diagnosis-spec

All three tools were emitting findings with the same structure: claim, file:line citation, applyable edit with pre-image guard, explicit `applyable: false` exit, and a categorical tag. The tag locates the defect in one of four buckets: the measurement instrument (the eval/funnel/metric definition), the interface (the API surface, SDK, error catalog), the context available at decision time (docs, prompts, signals), or the call sequence (state-machine ordering, retry patterns).

I hadn't planned the four buckets. The first tool's findings landed there because that's where they landed. The second and third tools' findings ended up in the same four because the four are what the underlying problem actually decomposes into.

I captured this as a spec: [agent-diagnosis-spec](https://github.com/ivaylogb/agent-diagnosis-spec). JSON Schema for the Finding object, prose definitions of the four buckets, and a conformance test suite that runs against the three tools' outputs to check they conform.

The first time I ran the conformance suite it caught a real bug: one of the tools was emitting citations as a bare-range string (`docs/x.md:96-103`) instead of the structured `{file, line_start, line_end}` object the other two used. The schema rejected it. I fixed the tool at the source.

## The orchestrator

The last thing I built was an orchestrator that runs the three tools and reports across them. It's called [pluma](https://github.com/ivaylogb/pluma). One CLI, with three subcommands that wrap the underlying tools, plus a fourth `cross` that runs two diagnostic lenses against the same product surface and reports where they independently agree.

The cross report normalizes findings from funnel-researcher and integration-watcher in the same product into the spec, then looks for findings pointing to the same part of the product surface area or effect. It converges on structural signals that the defect is real rather than a model artifact.

## Running the whole stack against Stripe Connect

The first real test was Stripe Connect onboarding: Stripe's product docs from docs.stripe.com (selected pages), `stripe-python` SDK source, Stripe's distilled error catalog, the Connect-relevant OpenAPI slice. The cohort data, ie the funnel, the dropoff numbers, the trace stream, the integration profile distribution, was synthesized, but grounded in public signal (`stripe-python` GitHub issues, Stack Overflow questions tagged stripe-connect, Stripe Support KB articles, Stripe's changelog).

Run results:

funnel-researcher, reading only the funnel and dropoff numbers, concluded that the largest dropoff in the funnel was between `requirements_collected` and `requirements_satisfied` (63% pass rate, next-worst step is 85%), concentrated in two failure buckets that share one mechanism: the API returns 200 from `POST /v1/accounts` but `requirements.currently_due` stays populated, and developers don't realize that's the failure signal.

integration-watcher, reading only the call traces, found that three of seven integrations created an account, GET-read requirements once, and then GET-polled forever without ever POSTing the requested fields back. 57 of 150 cohort calls followed that pattern. The integrations were waiting for a transition the API couldn't make without their submission. 


Same byte-exact citation into `docs/persons.md` (Stripe's "Handle verification with the API" guide). Triage caught one borderline finding worth mentioning. integration-watcher reported a pattern around developers re-POSTing rejected verification documents without the information needed to fix them. The friction was evident, but the finding's load-bearing claim "there is no error catalog in the artifacts" was false; `errors.md` is part of the product surface, 201 lines, 91 entries, including the exact row in question. The diagnostic harness's product loader hadn't ingested the top-level `errors.md`. I shipped that finding documented rather than papered over, because the triage step rejecting the false premise is the discipline working as designed.

Full worked example, including the synthesized inputs, the run outputs, and the triage notes, is at [pluma/examples/stripe](https://github.com/ivaylogb/pluma/tree/main/examples/stripe).
