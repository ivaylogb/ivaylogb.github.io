---
author: Ivaylo Bahtchevanov
pubDatetime: 2026-03-14T10:00:00Z
title: Persona Simulator: Predicting How Users Will React Before You Ship
slug: persona-simulator
featured: false
draft: false
tags: []
description: I connected Claude to our analytics warehouse, experiment history, design system, and strategy guidelines, and asked it to predict how different users would react to a homepage redesign. The most useful predictions weren't the ones we expected.
---

# Persona Simulator: Predicting How Users Will React to Before You Ship

*We redesigned our app's homepage. Before launching the A/B test, I connected Claude to our analytics warehouse, experiment history, design system, strategy guidelines, and more, and asked it to predict how a specific user segment would engage with the product*

---

## Catching what you don't expect

The most valuable experiments are always the ones that help you uncover something unexpected, not the ones that confirm your hypothesis. When your user base covers a broad set of segments with different goals, an improvement for one user could be 
detrimental to another. For example, cross-sell offers for savings products can feel predatory for a stressed out user trying to make ends meet, but could be helpful for another when trying to improve their finances.

The problem is, you only discover these surprises after weeks of running a real experiment with real users. By then, the damage to trust, conversion, and revenue has already happened. I wanted to surface these unexpected user perspectives before committing a single user to the test. 

The Personal Simulator is a tool I built to solve this problem. This would not replace traditional AB testing, user research, and other forms of analysis. The goal is to use existing data to build realistic representations on users, and then run simulations that would show how these users would engage with the experience. The simulation needs to be strongly grounded in the users, capturing their expected behavior, goals/motivations, context on how they fell into the product surface area in the first place, and where that user is trying to get to.

---

## How it works 

There are two possibilities: 
(1) a net new experience that you'd like to explore - how will this land with different users, how will they engage, what will they think, etc
(2) a change in an existing experience - a redesign, feature update, etc, with some specific goal in mind on either changing a target behavior or outcome, perception, or overall usability

The simulator allows for both. You can add a single scenario or two. 


You paste two Figma links (current design vs proposed change), select a user segment profile, type the questions you want answered, and optionally set a conversion target. Two minutes later, you get a structured analysis grounded in your company's own data.

The results include the following:
- A **verdict** with risk level and headline finding
- **Predictions by question** — each question answered with directional impact, confidence level, and a grounding label (data-backed vs heuristic)
- **Conversion impact** — quantified rate change, causal chain, helping/hindering elements, affected entrypoints, guardrail metrics
- A **behavioral walkthrough** — second-by-second journey through the new design with emotional state tracking
- **Recommendations** — ranked by priority and effort, each citing the evidence
- A **design system audit** — every component checked against the org's component libraries

---

## Why the predictions aren't generic AI output

Three design decisions are what separate this from a generic "have AI review my design" tool. Each one is a craft choice with a concrete payoff.

### 1. Screenshots as primary intelligence, not metadata

The simulator fetches actual Figma screenshots and sends them as multimodal image inputs to Claude — Claude literally sees what the user would see. It reads the visible text, assesses visual hierarchy, identifies which elements are prominent versus subtle, and notices things like a partially cut-off icon or a label change that the underlying component metadata doesn't capture.

This matters because Figma layer names lie. A widget named `[MAGIC]_WIDGET_4XN` tells you nothing about what the user actually sees. I established a three-layer hierarchy inside the prompt:

- The **screenshot** is what the customer sees (primary)
- The **widget metadata** is the engineer's map for connecting visual elements to analytics data (secondary)
- The **analytics baselines** calibrate prediction magnitude (tertiary)

The simulator thinks at all three layers but communicates at the screenshot layer — describing "a card showing 'Cartão de crédito R$ 180,30' with a purple progress bar," not "BALANCE_WIDGET at position 3." This is why predictions read like a human user-research finding rather than a structured data dump.

![Control versus treatment Figma designs side-by-side as the simulator sees them](./02-figma-screens.png)

### 2. Creative signal combination — the predictions emerge from layering

The most useful predictions emerge from combining signals that no single team typically puts together. Three examples:

**Click data + research patterns + visual analysis.** The balance widget has the highest tap rate on the homepage. Research shows 62.3% of late-debt users tap the credit card block first by label recognition. The screenshot shows the treatment replaced the explicit "Cartão de crédito" header with a compact modular card that contains the amount but removes the navigation affordances. Combining these three produces a recognition-failure prediction that none of them generates alone.

**Past experiment outcomes + current design + strategy rules.** A prior cross-sell visibility experiment had to be rolled back because +72% visibility produced 0% conversion lift and negative downstream impact. A separate experiment showed that *reducing* cross-sell dominance increased click-through by ~50%. The current treatment adds four cross-sell surfaces to a screen designed for a user whose only goal is to pay their bill — directly contradicting both historical learnings and the official homepage placement guidelines.

**Emotional state modeling + conversion funnels.** The behavioral walkthrough tracks the user's emotional progression — comfortable → hesitant → confused → frustrated — and at each step estimates time spent against the user's scroll-patience parameter. When patience exhausts, the user either abandons or falls back to a secondary path. The primary path converts at ~72%; the secondary at ~23%. The simulator doesn't just predict "the user will be frustrated" — it predicts "frustration at step 6 will redirect ~35% of users from a 72% path to a 23% path, resulting in a -20-35% net impact on bill payment initiation from home."

### 3. Calibrated confidence — every prediction tells you how grounded it is

Each prediction carries a confidence level (HIGH/MEDIUM/LOW) and a grounding label (GROUNDED in specific data versus HEURISTIC). The overall confidence score isn't a vague AI certainty number — it's a measurable ratio of enabled data-source weights to total available weights.

If you toggle off the past-experiments source, the confidence drops and the grounding labels shift. If you enable all eleven sources, the dashboard reads "10/11 sources active · 93% confidence" — and you can audit which sources fired for which claim by expanding the citations panel.

This matters more than it sounds. It means a reviewer can tell at a glance which predictions are anchored in real measurements and which are extrapolated reasoning — and decide which ones to weight in the launch decision.

![Data sources panel showing 10 of 11 sources active with 93% confidence score and weighted source cards](./03-sources-panel.png)

---

## The data sources — what's actually connected

Every prediction traces to a specific source. Here's what's wired in and why each one earns its place.

**past experiment outcomes from Slack.** Via the Slack MCP, the simulator pulls experiment decision threads from a channel where every shipped experiment is reviewed. Each one is structured with hypothesis, treatment description, outcome metrics with statistical significance flags, organizational reactions (including the contentious 29-reply debates), and synthesized learnings. This is the strongest signal: if someone tried something similar before, the outcome is the best predictor of what will happen again.

**Behavioral analytics from the warehouse.** Via the Databricks SQL MCP, the simulator queries widget interaction metrics aggregated across millions of users — tap rates by widget and segment, section engagement curves, homepage request volumes by income segment, cross-sell impression data. These baselines calibrate prediction magnitude. When the simulator says "-20 to -35% on first-tap CC rate," it derives this from measured baselines combined with position-displacement data from prior experiments.

**User research patterns.** Eight synthesized rules from Maze testing rounds and qualitative research, each with a confidence level and a source citation. *"Late-debt users prioritize CC content with <10% attention to cross-sell." "Position determines discovery." "Low-digital-sophistication users rely on label-matching rather than spatial memory."*

**22 strategy rules from the official placement RFC.** Via the Glean MCP, I found and ingested the official Widget Placement RFC — defining the homepage's three-zone structure, which widgets are permanently fixed, which products are excluded from which zones, sizing constraints, and experimentation rules. Every recommendation is validated against these. The simulator won't suggest impossible configurations.

**Post-click conversion funnel data.** When connected, this transforms perception predictions into quantified conversion impact — showing exactly which paths lose traffic and which paths absorb it.

**Figma design system via Claude Design.** Via the Figma MCP, the simulator queries the design system *during each run* — checking component libraries, design tokens, variant availability, and deprecated components. Claude decides what to look up based on the user profile and the visual analysis. The tool calls aren't hardcoded.

---

## The closed loop: predict, fix, verify

This is where the simulator stops being an analysis tool and becomes a design workflow.

### Design system audit

The simulator automatically audits every component in the treatment against the design system's libraries and classifies each one: approved, custom (not in any library), deprecated (from an old library), or missing variant (the component exists but a required state isn't being used).

For the test design: 11 approved components, 3 custom, 2 deprecated, 4 missing state variants. The critical finding — the credit card widget exists with 7 variants, but the delinquent/overdue state variant isn't being used. A design system gap directly causing a user experience failure for the exact segment most affected.

![Design system audit showing approved, custom, deprecated, and missing variant component classifications](./04-design-audit.png)

### Generate the fix in Figma

The recommendations aren't just text. Using Claude Design's write capabilities, the simulator clones the treatment frame in Figma and creates variant frames implementing each recommendation — importing the real design system components, applying correct design tokens, and positioning the variants alongside the original.

For the top recommendation ("add an explicit overdue label to the CC widget"), Claude Design generated Variant B: "Cartão de crédito" becomes "Cartão de crédito · Fatura em atraso" in red, "Fecha 29 MAR" becomes "Vencida · 8 dias em atraso," the overdue state is visually emphasized. For the second recommendation, Variant C: same changes plus the cross-sell section replaced with contextual debt-resolution messaging.

The designer gets Figma frames to refine, not text descriptions to interpret.

![Generated variant frames in Figma implementing the simulator's recommendations alongside the original treatment](./05-figma-variants.png)

### Re-simulate to verify

Swap the treatment link for the generated variant. Same user profile, same question, same conversion target. Re-run.

The prediction improved. Risk dropped from HIGH to MEDIUM. Conversion impact moved from -15 to -25% into the neutral-to-positive range. The simulator could see — in the new screenshot — that the explicit overdue label was now present, and adjusted its predictions accordingly.

**Predict → Fix → Verify. One tool.** The insight became action became validation, without leaving the simulator.

---

## Conversion impact: from "interesting feedback" to "launch plan"

The simulator's most useful output for an experiment owner isn't the prediction — it's everything that comes with it.

**A predicted rate change with a causal chain.** Not just "down 20%" but a numbered chain: (1) explicit label removal → (2) late-debt user can't pattern-match the CC widget → (3) first-tap on CC content drops → (4) users with low scroll patience who fail first-tap don't recover → (5) session abandonment before payment initiation increases.

**Helping and hindering elements.** A two-column layout showing what pushes conversion up (the "Pagar" inline action sits prominently above the fold; the CC card appears higher in treatment than control) versus what pushes it down (label removal, cross-sell competition, unfamiliar header chrome).

**Affected entrypoints.** A table predicting changes per conversion path: CC widget → CC dashboard → Pay Area; Pagar inline → barcode payment; alternative routes that absorb displaced traffic at lower conversion rates.

**Guardrail metrics to instrument.** Specific metrics with thresholds: barcode payment funnel entries from home, credit card delinquent widget impressions and click rate, time-to-first-tap on CC content for the late-debt segment, lending revenue, contact rate and NPS for the affected cohort.

![Conversion impact panel with predicted rate change, causal chain, and helping versus hindering elements](./06-conversion-impact.png)

This is what turns a design review into a pre-experiment planning artifact. You know what to monitor, what thresholds to set, and what rollback triggers to define — before a single user enters the test.

---

## How it was built

This was built over a weekend using Claude Code, orchestrating across seven internal systems via MCP:

- **Databricks SQL MCP** — widget metrics, section engagement, conversion funnels
- **Slack MCP** — experiment outcomes from the decisions channel, structured into a searchable library
- **Figma MCP (Claude Design)** — bidirectional: reads screenshots, component trees, design tokens; writes cloned frames implementing variant recommendations
- **Glean MCP** — discovered and ingested strategy RFCs, RACI guidelines, research docs
- **Confluence MCP** — design documentation, experiment tracking
- **Google Drive MCP** — strategy and planning documents
- **LiteLLM proxy** — Claude API gateway with SSO authentication

Custom slash commands keep the data fresh. `/project:refresh-data` re-queries all warehouse tables and pulls new experiments from Slack. The UI shows staleness badges — data older than 7 days is flagged.

The frontend is React + Vite + TypeScript. The prediction engine calls Claude Opus 4.7 with multimodal input — base64-encoded screenshots alongside a ~72,000-character structured prompt. The response parses into typed sections: executive summary, predictions, conversion impact, behavioral walkthrough, recommendations, design audit.

One person, one weekend, Claude Code as the orchestrator. The same pattern works for any domain where institutional knowledge is scattered across systems and never gets synthesized against a specific decision.

---

## Where this goes next

### Upstream context for agent simulations

I lead agent platform work, and there's a gap in every agent-testing tool I've evaluated. SnowGlobe, ArkSim, Simile — they simulate customers interacting with AI support agents, but none of them know what the customer just experienced in the app.

The Persona Simulator outputs a context bundle: behavioral parameters, emotional state, journey history, trust level, patience remaining. A user who spent 28 seconds confused on the homepage interacts with a support agent fundamentally differently from someone who found everything in 3 seconds. The simulator becomes the upstream context layer that makes agent simulations realistic.

### Automated iteration at scale

The predict → generate → verify loop, run across segments in parallel. Generate multiple variant frames, simulate each across different profiles, compare predictions in a matrix. Ship the design that works for every segment, with pre-identified guardrails and rollback triggers — instead of the design that works on average and surprises you on rollout.

---

## What I'd tell someone building something like this

**Start with the data you already have.** I pulled experiments from a Slack channel, click rates from existing analytics tables, behavioral patterns from research docs, strategy rules from a Google Doc. The data was there — it was just never synthesized against a specific decision.

**MCP connections compound.** Click rates alone are numbers. Click rates + past experiments + behavioral patterns + strategy guidelines = grounded predictions. Each connection makes every other connection more useful.

**Screenshots beat metadata.** If your tool analyzes designs, send the actual images. Layer names are unreliable. The screenshot is what the user sees — make it the primary source of truth and put the metadata in service of it.

**Use the API for reasoning, not summarization.** The Anthropic API's value isn't in restating data — it's in combining signals across different data types and modalities to surface insights no single source would produce. Multimodal input, tool use during inference, structured output schemas, weighted source ratios — these are what make the predictions non-obvious.

**Bound your recommendations.** Ingest your strategy guidelines and organizational constraints. Unbounded AI advice is noise. Bounded recommendations that cite evidence and respect structural rules are actionable.

**Close the loop.** Analysis is a report. Analysis → fix → verification is a workflow. The moment your tool can create the solution it recommends *and* verify that the solution works, it changes how teams make decisions.

---

*I'm Ivaylo Bahtchevanov — I lead AI enablement and agent platform work at Nubank. If you're building something similar, or want to discuss the architecture, MCP patterns, or prompting approach, reach out.*
