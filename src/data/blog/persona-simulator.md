---
author: Ivaylo Bahtchevanov
pubDatetime: 2026-02-17T10:00:00Z
modDatetime: 2026-08-19T11:18:18-07:00
title: "Persona Simulator: Predicting How Users Will React Before You Ship"
slug: persona-simulator
featured: true
draft: false
tags: []
description: A grounded simulation system for testing product experiences and agents against realistic user contexts before launch.
---

<style>
  #article .persona-figure {
    position: relative;
    left: 50%;
    width: min(800px, calc(100vw - 2rem));
    margin: 2.25rem 0;
    transform: translateX(-50%);
  }

  #article .persona-shot {
    overflow: hidden;
    border: 1px solid var(--border);
    border-radius: 0.875rem;
    background: #111;
    box-shadow: 0 22px 55px -42px rgb(15 23 42 / 65%);
  }

  #article .persona-image-link {
    display: block;
    width: 100%;
    cursor: zoom-in;
    text-decoration: none;
  }

  #article .persona-image-link:focus-visible {
    outline: 3px solid var(--accent);
    outline-offset: -3px;
  }

  #article .persona-figure img {
    display: block;
    width: 100%;
    max-width: none;
    height: auto;
    margin: 0;
    padding: 0;
    border: 0;
    border-radius: 0;
  }

  #article .persona-figure--narrow {
    width: min(760px, calc(100vw - 2rem));
  }

  #article .persona-figure--preview {
    width: min(720px, calc(100vw - 2rem));
  }

  #article .persona-figure--preview .persona-shot {
    aspect-ratio: 16 / 10;
  }

  #article .persona-figure--preview .persona-image-link {
    height: 100%;
  }

  #article .persona-figure--audit .persona-shot {
    aspect-ratio: 16 / 9;
  }

  #article .persona-figure--preview img {
    height: 100%;
    object-fit: cover;
    object-position: center top;
  }

  #article .persona-figure figcaption {
    max-width: 44rem;
    margin: 0.9rem auto 0;
    color: var(--subtle);
    font-family: ui-sans-serif, system-ui, sans-serif;
    font-size: 0.925rem;
    line-height: 1.6;
  }

  #article .persona-figure figcaption strong {
    color: var(--foreground);
  }

  #article .persona-callout {
    margin: 2rem 0;
    border: 1px solid var(--border);
    border-left: 3px solid var(--accent);
    border-radius: 0.75rem;
    padding: 1rem 1.15rem;
    background: var(--surface);
    box-shadow: 0 16px 38px -36px rgb(15 23 42 / 70%);
  }

  #article .persona-callout p {
    margin: 0;
  }

  #article .persona-callout p + p {
    margin-top: 0.35rem;
  }

  #article .persona-callout-label {
    color: var(--accent);
    font-family: ui-sans-serif, system-ui, sans-serif;
    font-size: 0.75rem;
    font-weight: 700;
    letter-spacing: 0.08em;
    text-transform: uppercase;
  }

  #article .persona-process {
    position: relative;
    left: 50%;
    display: grid;
    width: min(1040px, calc(100vw - 2rem));
    grid-template-columns: repeat(4, minmax(0, 1fr));
    gap: 1rem;
    margin: 2.5rem 0;
    padding: 0;
    transform: translateX(-50%);
    list-style: none;
  }

  #article .persona-process li {
    position: relative;
    margin: 0;
    border: 1px solid var(--border);
    border-top: 3px solid var(--accent);
    border-radius: 0.75rem;
    padding: 1rem;
    background: var(--surface);
    line-height: 1.5;
  }

  #article .persona-process li:not(:last-child)::after {
    content: "→";
    position: absolute;
    top: 50%;
    right: -1.1rem;
    z-index: 1;
    width: 1.2rem;
    color: var(--subtle);
    font-size: 1rem;
    text-align: center;
    transform: translateY(-50%);
  }

  #article .persona-process strong,
  #article .persona-process span {
    display: block;
  }

  #article .persona-process strong {
    margin-bottom: 0.35rem;
    color: var(--foreground);
    font-size: 0.95rem;
  }

  #article .persona-process span {
    color: var(--subtle);
    font-size: 0.82rem;
    line-height: 1.55;
  }

  #article .persona-process .persona-step-number {
    width: 1.5rem;
    height: 1.5rem;
    margin-bottom: 0.7rem;
    border-radius: 999px;
    background: var(--accent);
    color: #1c1713;
    font-size: 0.75rem;
    font-weight: 700;
    line-height: 1.5rem;
    text-align: center;
  }

  @media (max-width: 900px) {
    #article .persona-process {
      grid-template-columns: repeat(2, minmax(0, 1fr));
    }

    #article .persona-process li::after {
      display: none;
    }
  }

  @media (max-width: 740px) {
    #article .persona-figure {
      width: calc(100vw - 1rem);
      margin: 2rem 0;
    }

    #article .persona-figure figcaption {
      padding: 0 0.5rem;
    }

    #article .persona-process {
      width: calc(100vw - 1rem);
      grid-template-columns: 1fr;
      margin: 2rem 0;
    }
  }
</style>

## Catching what you don't expect

The most valuable experiments are always the ones that help you uncover something unexpected, not the ones that confirm your hypothesis. When your user base covers a broad set of segments with different goals, an improvement for one user could be detrimental to another. For example, cross-sell offers for savings products can feel predatory for a stressed-out user trying to make ends meet, but could be helpful for another who is trying to improve their finances.

The problem is, you only discover these surprises after weeks of running a real experiment with real users. By then, the damage to trust, conversion, and revenue has already happened. I wanted to surface these unexpected user perspectives before committing a single user to the test.

When we redesigned our app experience, I built the Persona Simulator to help us navigate critical changes to major products. We used it across 12 teams, and we now use it heavily to create realistic user context for agent simulations.

This does not replace traditional A/B testing, user research, or other forms of analysis. It helped us catch critical issues before any users saw them, understand where personalization made sense, and see how the same experience landed differently across user groups.

The rest of this post details the inner workings of the tool.

## How it works

There are two starting points:

### 1. A new experience to explore

You want to understand how it will land with different users, how they will engage, what they will think of each component, and whether the experience makes sense to them.

### 2. A change to an existing experience

A redesign or feature update with a specific goal, such as changing a target behavior, improving an outcome, or shifting perception and usability.

The simulator supports both. You can add one scenario for a new concept or compare a control and treatment. Add the Figma flows directly, or describe the experience in text if there is no prototype yet. Then select a user profile, enter the questions you want answered, and optionally set a conversion target. Two minutes later, you get a structured analysis grounded in your company's own data.

<ol class="persona-process" aria-label="Persona Simulator workflow">
<li><span class="persona-step-number">1</span><strong>Define the experience</strong><span>Add a Figma flow or describe the concept in text.</span></li>
<li><span class="persona-step-number">2</span><strong>Ground the persona</strong><span>Choose a user profile and connect the evidence sources.</span></li>
<li><span class="persona-step-number">3</span><strong>Simulate the journey</strong><span>Trace behavior, confidence, emotion, and likely outcomes.</span></li>
<li><span class="persona-step-number">4</span><strong>Decide and iterate</strong><span>Review risks, make a change, and run the same test again.</span></li>
</ol>

The output includes:

- A verdict with the headline risk
- Grounded predictions and a behavioral walkthrough
- Conversion impact and launch guardrails
- Ranked recommendations and a design system audit

## What the simulator returns

The summary view puts the likely outcome, main risks, recommended changes, and confidence in one place.

<figure class="persona-figure">
<div class="persona-shot"><a class="persona-image-link" href="/assets/persona-simulator/new_res.png" target="_blank" rel="noopener" aria-label="Open the full-size results dashboard in a new tab"><img src="/assets/persona-simulator/new_res.png" alt="Results dashboard with a medium-risk verdict, prioritized recommendations, and 81 percent confidence" width="1306" height="730" loading="lazy" decoding="async"></a></div>
<figcaption><strong>Decision summary.</strong> A concise read on the likely outcome, highest risks, recommended changes, and confidence.</figcaption>
</figure>

---

## Grounding the system

The core value is grounding each simulation in a specific user profile and modeling how that user would behave.

### 1. Multimodal input

The simulator fetches the Figma design(s) and turns them into multimodal inputs to Claude. It captures the prototype as images, connects the widgets to the design, reads the visible text, assesses visual hierarchy, identifies which elements are prominent vs subtle, and notices things like a partially cut-off icon or a label change that the underlying component metadata doesn't capture (important because Figma layer names can lie).

- The **screenshot** is what the customer sees (primary)
- The **widget metadata** is the engineer's map for connecting visual elements to analytics data (secondary)
- The **analytics baselines** calibrate prediction magnitude (tertiary)

The simulator thinks at all three layers but communicates at the screenshot layer. Predictions read like a human user-research finding rather than a structured data dump.

For comparisons, the control and treatment are evaluated side by side against the same evidence sources.

### 2. Combined signals

Realistic behavior comes from combining the qualitative and quantitative signals that are available.

**Click data + research patterns + visual analysis.**
The balance widget has the highest tap rate on the homepage. Research shows 62.3% of late-debt users tap the credit card block first by label recognition. The screenshot shows the treatment replaced the explicit "Cartão de crédito" header with a compact modular card that contains the amount but removes the navigation affordances. Combining these three produces a recognition-failure prediction that none of them generates alone.

**Past experiment outcomes + current design + strategy rules.**
A prior cross-sell visibility experiment had to be rolled back because +72% visibility produced 0% conversion lift and negative downstream impact. A separate experiment showed that _reducing_ cross-sell dominance increased click-through by ~50%. The current treatment adds existing cross-sell surfaces to a screen designed for a user whose only goal is to pay their bill, directly contradicting both historical learnings and the official homepage placement guidelines.

**Emotional state modeling + conversion funnels.**
The behavioral walkthrough tracks the user's emotional progression: comfortable → hesitant → confused → frustrated. At each step it estimates time spent against the user's scroll-patience parameter. When patience exhausts, the user either abandons or falls back to a secondary path. The primary path converts at ~72%; the secondary at ~23%. The simulator goes beyond "the user will be frustrated" and predicts "frustration at step 6 will redirect ~35% of users from a 72% path to a 23% path, resulting in a -20-35% net impact on bill payment initiation from home."

<figure class="persona-figure persona-figure--preview">
<div class="persona-shot"><a class="persona-image-link" href="/assets/persona-simulator/new_behavior_walk.png" target="_blank" rel="noopener" aria-label="Open the full-size behavioral walkthrough in a new tab"><img src="/assets/persona-simulator/new_behavior_walk.png" alt="Behavioral walkthrough tracing what the simulated user notices and does over time" width="1300" height="1114" loading="lazy" decoding="async"></a></div>
<figcaption><strong>Simulated journey.</strong> A timed trace of what the persona notices, where friction appears, and what they do next.</figcaption>
</figure>

### 3. Calibrated confidence: every prediction tells you how grounded it is

Each prediction carries a confidence level (HIGH/MEDIUM/LOW) with a score, and a grounding label (GROUNDED in specific data versus HEURISTIC). The overall confidence score is a measurable ratio of enabled data-source weights to total available weights.

If you toggle off the past-experiments source, the confidence drops and the grounding labels shift. If you enable all eleven sources, the dashboard reads "10/11 sources active · 93% confidence." You can audit which sources fired for which claim by expanding the citations panel.

Why this matters: it means a reviewer can tell at a glance which predictions are anchored in real measurements and which are extrapolated reasoning, allowing the reviewer to decide how much to weigh in the launch decision.

<aside class="persona-callout" aria-label="What the reviewer sees in the confidence panel">
<p class="persona-callout-label">What the reviewer sees</p>
<p><strong>10 of 11 data sources active · 93% confidence</strong></p>
<p>Every prediction identifies the sources that fired, their weights, and whether the conclusion is grounded in measured data or inferred heuristically.</p>
</aside>

---

## Building a realistic user model

Every prediction traces to a specific source. Here's what's wired in and why each one earns its place.

**Past experiment outcomes from Slack.**
Via the Slack MCP, the simulator pulls experiment decision threads from a channel where every shipped experiment is reviewed. Each one is structured with hypothesis, treatment description, outcome metrics with statistical significance flags, organizational reactions (including the contentious debates), and synthesized learnings. This is the strongest signal: if someone tried something similar before, the outcome is the best predictor of what will happen again.

**Behavioral analytics from the warehouse.**
Via the Databricks SQL MCP, the simulator queries widget interaction metrics aggregated across millions of users, ie tap rates by widget and segment, section engagement curves, homepage request volumes by income segment, cross-sell impression data. These baselines calibrate prediction magnitude. When the simulator says "-20 to -35% on first-tap CC rate," it derives this from measured baselines combined with position-displacement data from prior experiments.

**User research patterns.**
Synthesized rules from Maze testing rounds and qualitative research, each with a confidence level and a source citation. _"Late-debt users prioritize CC content with <10% attention to cross-sell." "Position determines discovery." "Low-digital-sophistication users rely on label-matching rather than spatial memory."_

**Strategy rules from the official placement RFC.**
Via the Glean MCP, the simulator loads the valid configurations, rules, and constraints for each product surface. Every recommendation is checked against them, so the simulator does not suggest configurations the product cannot support.

**Post-click conversion funnel data.**
When connected, this transforms perception predictions into quantified conversion impact, which shows exactly which paths lose traffic and which paths absorb it.

**Figma design system via Claude Design.**
Via the Figma MCP, the simulator queries the design system _during each run_: checking component libraries, design tokens, variant availability, and deprecated components. Claude decides what to look up based on the user profile and the visual analysis. The tool calls aren't hardcoded.

---

## Using personas to test agents

We also use the simulator heavily for agent simulations. Each run produces a context bundle with the user's goals, journey history, emotional state, trust level, and remaining patience. That bundle becomes the simulated user context an agent encounters, so we can test the same agent against materially different situations instead of a single generic prompt.

A user who spent 28 seconds confused on the homepage should not approach a support agent like someone who found what they needed in three seconds. Carrying that history into the simulation gives us more realistic conversations by user group and shows where an agent's behavior breaks across contexts.

<ol class="persona-process" aria-label="How Persona Simulator supplies context to agent simulations">
<li><span class="persona-step-number">1</span><strong>Persona parameters</strong><span>Goals, trust, patience, product state, and behavioral tendencies.</span></li>
<li><span class="persona-step-number">2</span><strong>Journey context</strong><span>What happened before the conversation, including friction and failure state.</span></li>
<li><span class="persona-step-number">3</span><strong>Agent conversation</strong><span>The same agent meets realistic reactions from materially different users.</span></li>
<li><span class="persona-step-number">4</span><strong>Evaluation</strong><span>Measure task success, escalation behavior, and consistency across contexts.</span></li>
</ol>

---

## The closed loop: predict, fix, verify

This is where the simulator stops being an analysis tool and becomes a design workflow.

### Design system audit

The simulator checks each component against the design system and flags custom, deprecated, or missing states. In this run, it found that an existing overdue-state variant was not being used, turning a broad risk into a specific design fix.

<figure class="persona-figure persona-figure--preview persona-figure--audit">
<div class="persona-shot"><a class="persona-image-link" href="/assets/persona-simulator/design_audit_full.png" target="_blank" rel="noopener" aria-label="Open the full-size design system audit in a new tab"><img src="/assets/persona-simulator/design_audit_full.png" alt="Design system audit classifying approved, custom, deprecated, and missing component variants" width="1088" height="1092" loading="lazy" decoding="async"></a></div>
<figcaption><strong>Design system audit.</strong> Component-level findings turn a broad risk into a concrete fix.</figcaption>
</figure>

### Generate the fix in Figma

The simulator can clone the treatment in Figma and apply its recommendations as editable variants. Here, one variant restored the explicit overdue state; another replaced unrelated cross-sell content with debt-resolution guidance. The designer gets frames to refine, not text descriptions to interpret.

### Re-simulate to verify

Swap the treatment link for the generated variant. Same user profile, same question, same conversion target. Re-run.

The prediction improved. Risk dropped from HIGH to MEDIUM. Conversion impact moved from a predicted decline of 15% to 25% into the neutral-to-positive range. The simulator could see that the explicit overdue label was now present, and adjusted its predictions accordingly.

**Predict → Fix → Verify.** Closed loop iteration on design concepts.

---

## Conversion impact

The simulator's most useful output for an experiment owner is the analysis the simulator provides about the experience itself.

<figure class="persona-figure persona-figure--narrow">
<div class="persona-shot"><a class="persona-image-link" href="/assets/persona-simulator/new_conversion.png" target="_blank" rel="noopener" aria-label="Open the full-size conversion analysis in a new tab"><img src="/assets/persona-simulator/new_conversion.png" alt="Conversion analysis separating the elements that help and hinder bill payment conversion" width="1280" height="664" loading="lazy" decoding="async"></a></div>
<figcaption><strong>Conversion impact.</strong> Likely behavior translated into impact, contributing factors, and launch guardrails.</figcaption>
</figure>

**A predicted rate change with a causal chain.**
It gives a net impact, such as "down 20%," and a direct chain of events: (1) explicit label removal → (2) late-debt user can't pattern-match the CC widget → (3) first-tap on CC content drops → (4) users with low scroll patience who fail first-tap don't recover → (5) session abandonment before payment initiation increases.

**Helping and hindering elements.**
A two-column layout showing what pushes conversion up (the "Pagar" inline action sits prominently above the fold; the CC card appears higher in treatment than control) versus what pushes it down (label removal, cross-sell competition, unfamiliar header chrome).

**Affected entrypoints.** A table predicting changes per conversion path: CC widget → CC dashboard → Pay Area; Pagar inline → barcode payment; alternative routes that absorb displaced traffic at lower conversion rates.

**Guardrail metrics to instrument.** Specific metrics with thresholds: barcode payment funnel entries from home, credit card delinquent widget impressions and click rate, time-to-first-tap on CC content for the late-debt segment, lending revenue, contact rate and NPS for the affected cohort.

This is what turns a design review into a pre-experiment planning artifact. You know what to monitor, what thresholds to set, and what rollback triggers to define before a single user enters the test.

---

## How it was built

This was built over a weekend using Claude Code, orchestrating across seven internal systems via MCP:

- **Databricks SQL MCP:** widget metrics, section engagement, conversion funnels
- **Slack MCP:** experiment outcomes from the decisions channel, structured into a searchable library
- **Figma MCP + Claude Design:** reads screenshots, component trees, and design tokens; writes cloned frames that implement variant recommendations
- **Glean MCP:** strategy RFCs, RACI guidelines, and research documents
- **Confluence and Notion MCP:** design documentation and experiment tracking
- **Google Drive MCP:** strategy and planning documents
- **Anthropic API:** synthesizes past user behavior from data and predicts future behavior

Custom slash commands keep the data fresh. `/project:refresh-data` re-queries all warehouse tables and pulls new experiments from Slack. The UI shows staleness badges. Data older than 7 days is flagged.

The frontend is React + Vite + TypeScript. The prediction engine calls Claude Opus 4.7 with multimodal input: base64-encoded screenshots alongside a ~72,000-character structured prompt. The response parses into typed sections: executive summary, predictions, conversion impact, behavioral walkthrough, recommendations, and design audit.

One person, one weekend, Claude Code as the orchestrator. The same pattern works for any domain where institutional knowledge is scattered across systems and never gets synthesized against a specific decision.

---

## From one persona to portfolio impact

The next iteration runs the same launch scenario across multiple user segments. The comparison separates broad wins from segment-specific gains and potential harm, which supports a phased rollout with explicit guardrails.

### Automated iteration at scale

The predict → generate → verify loop can run across segments in parallel. Generate multiple variant frames, simulate each one across different profiles, and compare the predictions in a matrix. This makes it possible to ship the design that works across segments, with guardrails and rollback triggers identified before rollout.

---

## Guidance for building something similar

**Start with the data you already have.** I pulled experiments from a Slack channel, click rates from existing analytics tables, behavioral patterns from research docs, and strategy rules from a Google Doc. The data was there. It was just never synthesized against a specific decision.

**MCP connections compound.** Click rates alone are numbers. Click rates + past experiments + behavioral patterns + strategy guidelines = grounded predictions. Each connection makes every other connection more useful.

**Screenshots beat metadata.** If your tool analyzes designs, send the actual images. Layer names are unreliable. The screenshot is what the user sees. Make it the primary source of truth and put the metadata in service of it.

**Use the API for reasoning, not summarization.** The Anthropic API's value isn't in restating data. It's in combining signals across different data types and modalities to surface insights no single source would produce. Multimodal input, tool use during inference, structured output schemas, and weighted source ratios are what make the predictions non-obvious.

**Bound your recommendations.** Ingest your strategy guidelines and organizational constraints. Unbounded AI advice is noise. Bounded recommendations that cite evidence and respect structural rules are actionable.

**Close the loop.** Analysis is a report. Analysis → fix → verification is a workflow. The moment your tool can create the solution it recommends _and_ verify that the solution works, it changes how teams make decisions.
