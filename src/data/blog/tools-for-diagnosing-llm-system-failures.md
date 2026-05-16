---
author: Ivaylo Bahtchevanov
pubDatetime: 2026-04-11T10:00:00Z
title: Diagnostic Tools for LLM Systems
slug: tools-for-diagnosing-llm-system-failures
featured: false
draft: false
tags: []
description: A few months of work on tools for figuring out why an LLM-mediated system failed. They read from an existing stack (Braintrust, LangSmith, PostHog, OpenTelemetry), iteratively identify defects, and propose fixes.
---


I've spent the last few years working on LLM-mediated systems, agents, developer-facing APIs, products built around language models. A lot of that time has been spent diagnosing what went wrong and why.

Agent tooling has matured fast. There's good literature on crafting rigorous evals, mature platforms for capturing traces and running experiments, and a healthy practice around manual review and product analytics. None of that is going away anytime soon. But when something fails, an eval regresses, a funnel stalls, an integration cohort can't get past step three, what you get back from your tools is still mostly a verdict and a paragraph of prose. At the end of the day, you still have a human
reading dashboards and writing memos.

The tools here are built to help make the process a bit more systematic. 

This is a description of what's in each.

## agent-researcher

The starting point was an eval failing on a routing classifier. A scenario tagged `bug` was getting predicted as `unknown`. I wanted something that could read the failing scenario, read the agent's source, and tell me what to change.

[agent-researcher](https://github.com/ivaylogb/agent-researcher) takes a failing eval and outputs three structured hypotheses. Each hypothesis has:

- Claim/hypothesis about why the agent failed
- Citation into a file with specific line range `path:start-end`
- Proposed code edit, formatted as a structured artifact with the pre-edit content the editor expects to find. If the file at that path doesn't match the expected pre-edit content, the edit is refused rather than applied silently
- Verification step: apply the edit, re-run the eval, report the delta on a named metric

When the tool can't propose a clean single-file edit, it returns `applyable: false` with a reason.

The eval doesn't have to be in a specific framework. agent-researcher reads a single JSON file describing the failing scenario, the expected output, and what the agent actually produced. That format is generic enough to ingest and map Braintrust experiments, LangSmith runs, and custom eval harnesses.

On the `bug`/`unknown` scenario it emitted three hypotheses (H1, H2, H3). I applied H1 and H2 with the verifier. Pass rate went from 6/7 to 7/7 under H1 and stayed at 6/7 under H2. H2's prose read more confident than H1's. The H1/H2/H3 run is the worked example in `examples/issue_107/` in the repo.

## funnel-researcher

A different problem: a developer-facing API where activation was stalling at a particular step in the onboarding funnel. I had funnel definitions and dropoff numbers. I needed something that could read those alongside the product artifacts (docs, SDK, error catalog) and tell me where in the surface the friction was living.

[funnel-researcher](https://github.com/ivaylogb/funnel-researcher) takes a funnel definition, dropoff data, and a directory of product artifacts. It outputs structured hypotheses about why developers fall off at a named target step. Findings have the same shape as agent-researcher's: claim, `path:start-end` citation, applyable edit (often into a doc file rather than source), explicit `applyable: false` when there's no clean single-edit fix.

Most teams running developer APIs already produce this data in PostHog, Amplitude, Mixpanel, or a similar analytics platform. The funnel definitions and dropoff cohorts they emit map onto the shape funnel-researcher reads.

I used the same finding shape as agent-researcher because the failure I was describing didn't depend on whether the system was an agent or a developer API. The structure — defect, evidence, edit, verification — was the same.

## integration-watcher

A third problem: a cohort of developer integrations against an API. The question was where in the call sequence integrations were getting stuck. I had traces — JSONL streams of API calls, one line per call, with timestamps and request/response shapes. I had a watch question (what we want the tool to look for). I needed something that could read the traces alongside the product artifacts and tell me what patterns developers were falling into.

[integration-watcher](https://github.com/ivaylogb/integration-watcher) does that. Inputs: trace stream, cohort definition, watch question, product artifact directory. Outputs: structured findings about patterns in the cohort, with prevalence numbers ("3 of 7 integrations"), trace citations, and the same applyable / `applyable: false` exit.

The trace stream can come from anywhere — PostHog event exports, OpenTelemetry spans, an API gateway's logs. They all describe the same underlying primitive: a stream of API calls with metadata about who made them and what came back.


## Interoperability

The shared format allows the findings to be correlated against each other. Each finding contains the following: a claim, a 
`file:line` citation, an applyable structured edit with a pre-image guard, and an explicit `applyable: false` exit when a clean fix isn't available. Each finding is tagged with one of four categories: the measurement instrument, the interface, the context available at decision time, or the call sequence. 

A single applier can mechanically apply edits from any tool. A finding from one tool can be ablated, falsified, or extended 
by another. None of that works if each tool defines its own shape.

The format is documented at [agent-diagnosis-spec](https://github.com/ivaylogb/agent-diagnosis-spec): 
JSON Schema, prose definitions, and a conformance suite that runs 
against each tool's output.


## Pluma

The fourth tool is an orchestrator: [pluma](https://github.com/ivaylogb/pluma). Pluma is your friendly "Plumber" that brings a CLI wrapping the underlying tools and then uses them as instruments, compares outputs on the same product surface and reports findings.

funnel-researcher sees aggregate dropoff numbers, integration-watcher sees the call sequences, and Pluma will normalize their findings against the spec and identify specific parts of the product surface to focus on. 

## Running the stack against Stripe Connect

Running the stack against Stripe Connect onboarding was an informative real-world test. Product surface included: Stripe's relevant docs from docs.stripe.com, `stripe-python` SDK source, Stripe's distilled error catalog, the Connect-relevant OpenAPI slice. Cohort data was synthesized into the funnel, dropoff numbers, trace stream, integration profile distribution, with additional grounding signal: `stripe-python` GitHub issues, Stack Overflow questions tagged stripe-connect, Stripe Support KB articles, Stripe's changelog.

funnel-researcher found that the largest dropoff was between `requirements_collected` and `requirements_satisfied` (63% pass rate, whereas the next-worst step was 85%). The drop was concentrated in two failure buckets: (1) developers who submit fields and abandon when `currently_due` doesn't change, and (2) developers whose POST updates return 200 but leave `currently_due` populated. The tool's conclusion: a 200 from `POST /v1/accounts` doesn't necessarily mean the account is progressing, the caller has to re-read `requirements.currently_due` on the response.

integration-watcher polled the call traces and found that three of seven integrations created an account, GET-read requirements once, and then GET-polled forever without ever POSTing the requested fields back. 57 of 150 cohort calls followed that pattern.

Both tools zeroed in on the `docs/persons.md` (Stripe's "Handle verification with the API" guide).

Another interesting finding from this run was how the triage was managed.

integration-watcher reported that developers were re-POSTing rejected verification documents without enough information to fix them. It cited specific trace evidence: a developer hitting `verification_document_failed_greyscale`, then submitting the same 
document shape again, and stalling. It then proposed authoring a new `errors.md` to document what each rejection code means. 

The identified pattern was correct, and was mapped to `stripe-python` issues and Stripe's own support knowledge base where the friction was described.

However, in the first run, Pluma's loader didn't ingested top-level files in the input directory. We intentionally hid them so the watcher would miss the catalog, and wouldn't see that there's an existing `errors.md` with 201 lines and 91 entries, which included the rejection code in question. It recommended an unnecessary fix in this step.

But when Pluma ran the triage step, it identified the limitation. The diagnostic tool cross-references each finding against external signal to validate the claims on real-world sources. It passed the pattern claim but flagged the proposed fix claim, marking the finding as BORDERLINE (pattern correct, fix moot). It flagged the issue in the loader, and proposed an update there.


One borderline finding came out of the original run worth mentioning. integration-watcher reported a pattern around developers re-POSTing rejected verification documents without the information needed to fix them. The friction was real and grounded in the public signal. But the finding's claim that "there is no error catalog in the artifacts" was false. `errors.md` is part of the product surface — 201 lines, 91 entries, including the exact row in question. The product loader hadn't ingested the top-level `errors.md`.

The triage step is what surfaced this. Each finding gets cross-checked against external signal — does the claim match what we know from real-world sources? The pattern claim matched. The proposed fix didn't, because `errors.md` clearly existed. Investigating why the tool would say something demonstrably false traced the problem to the loader, not the diagnostic methodology itself.

When re-running with the fixed loader code, the F3 reformulated correctly citing `errors.md:173`. Pluma identified the genuine residual gap (the row tells developers to "re-collect a color document" but doesn't link them to the test-mode fix token), and proposes a surgical one-line catalog edit. BORDERLINE → REAL. 

Full worked example, including the synthesized inputs, the run outputs, and the triage notes, is at [pluma/examples/stripe](https://github.com/ivaylogb/pluma/tree/main/examples/stripe).


## Plugging it into your existing stack

The tools don't make assumptions about where their inputs come from. Adapters bridge platform-specific data formats into the shapes the tools consume. There's a [separate post about the first two adapters](/posts/plugging-diagnostic-stack-into-braintrust-and-posthog/) — one for Braintrust experiments, one for PostHog event exports. Both adapters live in [pluma](https://github.com/ivaylogb/pluma) under `src/pluma/integrations/`. More are coming as the data shapes of common platforms (LangSmith, OpenTelemetry, Amplitude) get mapped in.
