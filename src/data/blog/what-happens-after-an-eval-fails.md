---
author: Ivaylo Bahtchevanov
pubDatetime: 2026-03-24T10:00:00Z
title: What Happens After an Eval Fails?
featured: true
draft: false
tags: []
description: How GEPA uses failed examples, traces, and written feedback to propose prompt changes for replay and review.
---

<style>
  #article .gepa-article {
    --gepa-blue: #3567b7;
    --gepa-green: #287a58;
    --gepa-red: #a64242;
  }

  html[data-theme="dark"] #article .gepa-article {
    --gepa-blue: #8bb5ff;
    --gepa-green: #78c9a3;
    --gepa-red: #f19a9a;
  }

  #article .gepa-figure {
    position: relative;
    left: 50%;
    width: min(900px, calc(100vw - 2rem));
    margin: 3rem 0;
    transform: translateX(-50%);
  }

  #article .gepa-visual {
    overflow: hidden;
    border: 1px solid var(--border);
    border-radius: 0.9rem;
    background: var(--surface);
    box-shadow: 0 20px 48px -42px rgb(15 23 42 / 70%);
    font-family: ui-sans-serif, system-ui, sans-serif;
  }

  #article .gepa-flow,
  #article .gepa-candidates {
    display: grid;
    grid-template-columns: repeat(4, minmax(0, 1fr));
    gap: 2.1rem;
    padding: 1.4rem;
  }

  #article .gepa-card {
    position: relative;
    min-width: 0;
    border: 1px solid var(--border);
    border-radius: 0.7rem;
    padding: 1rem;
    background: color-mix(in srgb, var(--surface) 84%, var(--muted));
  }

  #article .gepa-flow .gepa-card:not(:last-child)::after,
  #article .gepa-candidates .gepa-card:not(:last-child)::after {
    position: absolute;
    top: 50%;
    left: calc(100% + 0.75rem);
    width: 0.5rem;
    height: 0.5rem;
    border-top: 2px solid currentColor;
    border-right: 2px solid currentColor;
    color: var(--subtle);
    content: "";
    transform: translateY(-50%) rotate(45deg);
  }

  #article .gepa-card-kicker {
    margin: 0 0 0.45rem;
    color: var(--gepa-blue);
    font-size: 0.75rem;
    font-weight: 700;
    letter-spacing: 0.08em;
    line-height: 1.2;
    text-transform: uppercase;
  }

  #article .gepa-card-title {
    margin: 0;
    color: var(--foreground);
    font-size: 1rem;
    font-weight: 700;
    line-height: 1.35;
  }

  #article .gepa-card-copy {
    margin: 0.45rem 0 0;
    color: var(--subtle);
    font-size: 0.84rem;
    line-height: 1.5;
  }

  #article .gepa-route-note,
  #article .gepa-run-note {
    margin: 0;
    border-top: 1px solid var(--border);
    padding: 0.95rem 1.4rem;
    background: color-mix(in srgb, var(--muted) 50%, var(--surface));
    color: var(--subtle);
    font-size: 0.82rem;
    line-height: 1.55;
    text-align: center;
  }

  #article .gepa-route-note strong {
    color: var(--gepa-red);
  }

  #article .gepa-status {
    display: inline-flex;
    margin-top: 0.7rem;
    border: 1px solid currentColor;
    border-radius: 999px;
    padding: 0.2rem 0.5rem;
    font-size: 0.72rem;
    font-weight: 700;
    line-height: 1.2;
  }

  #article .gepa-status-retain {
    color: var(--gepa-green);
  }

  #article .gepa-status-reject {
    color: var(--gepa-red);
  }

  #article .gepa-case-row {
    display: flex;
    justify-content: space-between;
    gap: 0.6rem;
    margin-top: 0.45rem;
    border-top: 1px solid var(--border);
    padding-top: 0.45rem;
    color: var(--subtle);
    font-size: 0.78rem;
    line-height: 1.35;
  }

  #article .gepa-pass {
    color: var(--gepa-green);
    font-weight: 700;
  }

  #article .gepa-fail {
    color: var(--gepa-red);
    font-weight: 700;
  }

  #article .gepa-feedback {
    margin: 2rem 0;
    border: 1px solid var(--border);
    border-left: 3px solid var(--gepa-blue);
    border-radius: 0.75rem;
    padding: 1rem 1.1rem;
    background: var(--surface);
  }

  #article .gepa-feedback dl {
    display: grid;
    grid-template-columns: 8.5rem minmax(0, 1fr);
    gap: 0.55rem 1rem;
    margin: 0;
  }

  #article .gepa-feedback dt {
    margin: 0;
    color: var(--subtle);
    font-family: ui-sans-serif, system-ui, sans-serif;
    font-size: 0.8rem;
    font-weight: 700;
    line-height: 1.55;
  }

  #article .gepa-feedback dd {
    margin: 0;
    color: var(--foreground);
    font-family: ui-sans-serif, system-ui, sans-serif;
    font-size: 0.9rem;
    line-height: 1.55;
  }

  #article .gepa-figure figcaption {
    max-width: 46rem;
    margin: 0.85rem auto 0;
    color: var(--subtle);
    font-family: ui-sans-serif, system-ui, sans-serif;
    font-size: 0.9rem;
    line-height: 1.55;
    text-align: center;
  }

  @media (max-width: 900px) {
    #article .gepa-flow,
    #article .gepa-candidates {
      grid-template-columns: 1fr;
      gap: 2rem;
      padding: 1rem;
    }

    #article .gepa-flow .gepa-card:not(:last-child)::after,
    #article .gepa-candidates .gepa-card:not(:last-child)::after {
      top: calc(100% + 0.5rem);
      left: 50%;
      transform: translateX(-50%) rotate(135deg);
    }

    #article .gepa-feedback dl {
      grid-template-columns: 1fr;
      gap: 0.15rem;
    }

    #article .gepa-feedback dd + dt {
      margin-top: 0.55rem;
    }
  }
</style>

<div class="gepa-article">

A red row in an eval table tells you a run missed the target. To decide what to change, you still have to find out whether the problem came from the prompt, the evaluator, a tool call, or context lost between services.

The usual investigation starts with the trace: read the failure, edit a prompt, and rerun a few examples. GEPA, short for Genetic-Pareto, makes that process repeatable. It uses failed examples, scores, and written feedback to propose revised instructions. Each revision returns to the evaluation workflow for replay.

Existing eval platforms already store traces, datasets, evaluator results, experiment runs, and prompt versions. Keep that history there. GEPA works alongside it during the improvement process.

<figure class="gepa-figure"><div class="gepa-visual"><div class="gepa-flow"><div class="gepa-card"><p class="gepa-card-kicker">Evidence</p><p class="gepa-card-title">Review the failed trace</p><p class="gepa-card-copy">Join the score to the prompt, model, tools, state, and final outcome.</p></div><div class="gepa-card"><p class="gepa-card-kicker">GEPA search</p><p class="gepa-card-title">Propose prompt candidates</p><p class="gepa-card-copy">Use the expected result and written feedback to produce focused changes.</p></div><div class="gepa-card"><p class="gepa-card-kicker">Evaluation</p><p class="gepa-card-title">Replay and compare</p><p class="gepa-card-copy">Run prior successes, important categories, required checks, and untouched examples.</p></div><div class="gepa-card"><p class="gepa-card-kicker">Release decision</p><p class="gepa-card-title">Review the evidence</p><p class="gepa-card-copy">Inspect the exact prompt changes, results, cost, remaining failures, and rollback plan.</p></div></div><p class="gepa-route-note"><strong>Move to an engineering fix</strong> when the failure comes from a tool, retrieval system, schema, policy, or runtime.</p></div><figcaption>GEPA uses the evidence already stored in the evaluation workflow and returns candidate prompts for the same workflow to test.</figcaption></figure>

## Start with the trace

Consider an evaluator that checks whether a support agent transferred a case too early. In one conversation, the customer supplied the required information and a person later completed the requested action. The evaluator still marked the transfer as justified because an optional detail was missing.

The score identifies a misclassification. The trace shows what the customer provided, which capability was available, when the transfer happened, and how the case ended.

That evidence also points to the right fix. If a field disappeared in an adapter, repair the adapter. If a required tool was unavailable, fix or add the capability. Here, the faulty decision rule lived in the evaluator prompt, which made it a reasonable GEPA target.

Make this check before every run. Prompt changes are easy to generate and can hide a deeper defect for a while. The trace keeps the change tied to the cause.

## Give the search useful feedback

A score ranks prompt versions. Written feedback tells GEPA what went wrong and which evidence should change the decision.

For the support example, the feedback can be short:

<div class="gepa-feedback"><dl><dt>Expected result</dt><dd>Flag the transfer as unnecessary.</dd><dt>Observed result</dt><dd>The evaluator treated the transfer as justified.</dd><dt>Cause</dt><dd>An optional detail was treated as a required field.</dd><dt>Evidence</dt><dd>The required information was complete and consistent, and the requested action was available.</dd><dt>General rule</dt><dd>Missing optional information should not justify a transfer when the required evidence is complete.</dd></dl></div>

The score tells GEPA which prompts performed better. The feedback supplies the missing decision rule. That record supports a focused revision. Broad instructions such as “reason more carefully” give the model no concrete rule to follow.

The feedback should describe a reusable rule. Copying the example into the prompt usually fixes a single row and makes the prompt longer. A clear rule can cover a family of similar cases.

## Search across candidate prompts

One failed example can support several plausible edits. A revision that treats the missing detail as optional may become too permissive when other required evidence is incomplete. A stricter revision may work for the original case and fail on a valid local variation.

GEPA can keep several prompt versions when each succeeds on different examples. That per-example comparison is the Pareto part of the method. It preserves useful branches long enough to show which rules each one adds.

In my evaluator experiment, the original prompt scored 8 of 10 on validation. The first major revision also scored 8 of 10, with each prompt missing different examples. A later revision reached 9 of 10. Further revisions fell to 7 of 10, so I kept the 9 of 10 version.

I then tested the selected prompt on 20 examples that had stayed out of development. The original prompt handled 17 correctly and the selected version handled 18. The sample is small, so I keep its size beside the result and limit the claim to this experiment.

<figure class="gepa-figure"><div class="gepa-visual"><div class="gepa-candidates"><div class="gepa-card"><p class="gepa-card-kicker">Original prompt</p><p class="gepa-card-title">Validation: 8 of 10</p><div class="gepa-case-row"><span>Example A</span><span class="gepa-pass">Pass</span></div><div class="gepa-case-row"><span>Example B</span><span class="gepa-fail">Fail</span></div></div><div class="gepa-card"><p class="gepa-card-kicker">Revision 1</p><p class="gepa-card-title">Validation: 8 of 10</p><div class="gepa-case-row"><span>Example A</span><span class="gepa-fail">Fail</span></div><div class="gepa-case-row"><span>Example B</span><span class="gepa-pass">Pass</span></div></div><div class="gepa-card"><p class="gepa-card-kicker">Revision 2</p><p class="gepa-card-title">Validation: 9 of 10</p><p class="gepa-card-copy">Final test: 18 of 20. Original prompt: 17 of 20.</p><span class="gepa-status gepa-status-retain">Retain</span></div><div class="gepa-card"><p class="gepa-card-kicker">Later revision</p><p class="gepa-card-title">Validation: 7 of 10</p><p class="gepa-card-copy">Keep the best previous version when a revision loses ground.</p><span class="gepa-status gepa-status-reject">Reject</span></div></div><p class="gepa-run-note">The strongest candidate appeared at iteration 15. The run continued to iteration 161 without improving it.</p></div><figcaption>Equal averages can hide different failures. For release, use the result earned by one prompt on the complete validation set.</figcaption></figure>

Together, the two 8 of 10 prompts covered 9 validation examples. Each individual prompt still scored 8 of 10 at that point. GEPA used the complementary results to guide later revisions, including the single prompt that eventually scored 9 of 10.

A new version can lose. Keeping the strongest known candidate avoids the common habit of equating the latest edit with progress.

## Keep the evaluator steady

GEPA improved the evaluator prompt in this experiment. The same process can later target an agent prompt, tool instructions, or a routing policy. Before using an evaluator to compare agent prompts, calibrate it and freeze its version.

Reviewers should agree on the behavior being measured. Inputs must map to the expected fields. The score direction should be clear. Repeated runs should reveal how much the evaluator varies. Once the agent comparison starts, freeze the evaluator version.

If the evaluator changes during search, run the baseline and every candidate again. The old results were produced under a different definition.

Some checks are better expressed in code. A required tool call, an accepted transfer, or a ban on actions after a terminal state can be checked directly. LLM evaluation is useful for judgments such as relevance, completeness, or whether an escalation made sense in context.

## Keep the results in your eval platform

If your team already uses an eval platform, keep its traces, datasets, annotations, prompt versions, and experiment results there. Feed reviewed examples and their feedback into GEPA, then write the new prompt versions and replay results back to the same workflow.

Store each version with its exact prompt changes and the examples that motivated them. Put its replay results next to the original prompt. Reviewers should be able to see which failures were fixed and whether earlier successes broke.

The eval platform records what happened under each version. GEPA explores instruction changes. People decide what reaches production.

## Know when to stop

In my experiment, the strongest prompt appeared at iteration 15. The search continued to iteration 161 and often drew small batches with no failures to learn from. The fixed budget allowed it to continue. A plateau rule would have stopped earlier and saved evaluator calls.

Useful stopping signals include:

- Validation has stopped improving across several candidate rounds.
- Small batches repeatedly contain no error signal.
- The improvement is within the normal variance of the evaluator.
- Prompt length, latency, or cost has grown beyond the value of the gain.
- The remaining failures point to a tool, data, retrieval, or runtime problem.

GEPA works on language-defined decisions when the system already has the capability to carry them out. Dropped fields, missing APIs, broken state boundaries, and authorization enforcement require engineering changes.

Sometimes the right result from an optimization run is an engineering ticket.

## Review the candidate before release

The strongest candidate should arrive with enough evidence for another person to review the decision:

- The exact prompt changes.
- The examples that motivated the change.
- Validation and final test results.
- Regressions by important category.
- Results from required safety and policy checks.
- Changes in prompt size, latency, and cost.
- A known rollback version.

Approval creates a versioned prompt. Deployment follows the normal release path, preferably through a limited rollout with monitoring. New production failures belong in a future dataset and a future run.

When an eval fails, keep the score attached to the trace and add feedback that explains the missed decision rule. GEPA uses that evidence to propose revisions. Those revisions go back through the existing eval workflow, where the results are stored and the team decides whether a change is ready for release.

</div>
