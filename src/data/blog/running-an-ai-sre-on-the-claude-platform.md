---
author: Ivaylo Bahtchevanov
pubDatetime: 2026-03-17T10:00:00Z
title: Running an AI SRE on the Claude platform
featured: true
draft: false
ogImage: https://ivaylogb.github.io/assets/ai-sre/og.png
tags:
  - agents
  - sre
  - claude
  - applied-ai
description: How we built Ada, the agent that investigates production incidents using structured outputs, the Agent SDK, MCP, Skills, model routing, and a gated action plane.
---

<style>
  #article .sre-article figure {
    width: 100%;
    margin: 3.25rem 0;
  }

  #article .sre-article figure picture,
  #article .sre-article figure img {
    display: block;
    width: 100%;
  }

  #article .sre-article figure img {
    height: auto;
    margin: 0;
    padding: 0.75rem;
    border: 1px solid #dfe4eb;
    border-radius: 0.75rem;
    background: #f5f7fa;
    box-shadow: 0 18px 44px -34px rgb(15 23 42 / 45%);
  }

  #article .sre-article .sre-product-figure img {
    padding: 0;
    background: #f5f7fa;
  }

  #article .sre-article figcaption {
    max-width: 48rem;
    margin: 0.9rem auto 0;
    color: var(--subtle);
    font-family: ui-sans-serif, system-ui, sans-serif;
    font-size: 0.925rem;
    line-height: 1.6;
    text-align: center;
  }

  #article .sre-article pre code {
    display: block;
    white-space: pre;
  }

  #article .sre-article .sre-footnote {
    margin-top: 3.5rem;
    padding-top: 1.1rem;
    border-top: 1px solid var(--border);
    color: var(--subtle);
    font-family: ui-sans-serif, system-ui, sans-serif;
    font-size: 0.82rem;
    line-height: 1.6;
  }

  @media (max-width: 900px) {
    #article .sre-article figure img {
      padding: 0.45rem;
    }
  }
</style>

<div class="sre-article">
<p>Incident response at Nubank is largely automated. Bots open the incident, propose severity, create the Slack war room, page responders, file tickets, and schedule the review. The step that still depended on a senior engineer was investigation: using judgment and experience to turn the metrics, logs, traces, deploy history, and past incidents into a credible root&#8209;cause hypothesis, often at 3 a.m.</p>

<p>In 2025 we shipped a read&#8209;only investigation agent, internally called Ada, on a custom-built harness, because nothing hosted at the time had the durable, steerable sessions an investigation needs. Ada worked well enough to expose the problems that matter at the next stage. It could not search past incidents, could not be redirected mid&#8209;investigation, could not act on its conclusions, ran on infrastructure that consumed most of a two&#8209;person team, and had no reliable way to measure whether its diagnoses were correct.</p>

<p>This post describes the v2 design. Structured outputs type every artifact another system parses, the Agent SDK runs the investigation loop, MCP carries the operational reads, code in the middle keeps raw telemetry out of the model’s context, and Skills hold the runbooks service teams own. Two constraints shaped the design more than any feature choice: where session state can live, and which actions exist for the agent at all. Anthropic's cookbooks cover incident&#8209;response agents on both runtimes. This article focuses on the production design around those mechanics.</p>

<h2 id="architecture-at-a-glance">The architecture at a glance</h2>

<figure class="sre-figure">
<picture>
  <source media="(max-width: 900px)" srcset="/assets/ai-sre/architecture-mobile.svg">
  <img src="/assets/ai-sre/architecture.svg" alt="End-to-end architecture: alert to triage on the Messages API, to an Agent SDK investigator, to a gated action plane, over shared bank-owned components that persist across sessions" width="1240" height="540" loading="lazy" decoding="async">
</picture>
<figcaption>An alert acts as a typed envelope on the Messages API, an Agent SDK session investigates it, and any proposed action crosses a separate gated plane. Four shared components persist across sessions.</figcaption>
</figure>

<p>An alert enters through a fast path on the Messages API, which classifies it and emits a typed incident envelope. The envelope starts an Agent SDK session inside our infrastructure. The session loads the relevant Skills, gathers evidence through MCP read tools, clears a grading pass, and posts a structured diagnosis into the incident's Slack thread, where the on&#8209;call engineer can label it or redirect it. If the diagnosis warrants action, the proposal goes to a separate action plane, where a human approves a dry run before anything executes. Four components sit under all of this and outlive any single session: the incident index, the Skills registry, the eval store, and the action gateway.</p>

<p>We split the design by whether the work can be bounded in advance. Triage, judge scoring, and the grader each have clearly defined input and output, so they run as single Messages API calls with structured outputs and no session state. The investigation remains open&#8209;ended and runs as an Agent SDK session, with state held as files on our infrastructure. Runbooks are Skills, maintained in the owning team's repo. Writes go through the action gateway as typed proposals.</p>

<h2 id="operator-workflow">The operator workflow</h2>

<p>The operator sees that flow as one incident workspace. The default view combines the incident timeline, the current assessment, the evidence state, and the proposed mitigation. Raw source activity stays collapsed until it is needed. The screens in this post use fictional incident data to show the workflow.</p>

<figure class="sre-product-figure">
  <a href="/assets/ai-sre/product/investigation-overview.png" target="_blank" rel="noopener" aria-label="Open the full-size investigation workspace in a new tab">
    <img src="/assets/ai-sre/product/investigation-overview.png" alt="Incident investigation workspace showing a checkout alert, incident context, evidence review status, and the current deployment-regression assessment" width="1422" height="800" loading="lazy" decoding="async">
  </a>
  <figcaption>The incident workspace keeps the timeline, sourced assessment, and proposed mitigation together. Operators can expand the full source activity when they need it.</figcaption>
</figure>

<h2 id="triage-runs-on-the-messages-api">Triage runs on the Messages API</h2>

<p>A single API call classifies the alert, proposes a severity, and normalizes the payload into the incident envelope. The envelope starts a session, the judge&#8217;s score files into the eval store, and the grader&#8217;s verdict decides whether a diagnosis posts; the first reader of each is software, so a response that almost parses is a failure. Structured outputs remove that failure class at the API. The envelope schema goes in <code>output_config.format</code> and the response is guaranteed to validate against it:</p>

```python
from anthropic import Anthropic

client = Anthropic()

envelope = client.messages.create(
    model="claude-haiku-4-5-20251001",
    max_tokens=1024,
    messages=[{"role": "user", "content": alert_payload}],
    output_config={
        "format": {
            "type": "json_schema",
            "schema": {
                "type": "object",
                "properties": {
                    "service": {"type": "string"},
                    "severity": {
                        "type": "string",
                        "enum": ["sev1", "sev2", "sev3"],
                    },
                    "error_signature": {"type": "string"},
                    "affected_components": {
                        "type": "array",
                        "items": {"type": "string"},
                    },
                    "window_start": {"type": "string"},
                },
                "required": [
                    "service",
                    "severity",
                    "error_signature",
                    "affected_components",
                    "window_start",
                ],
                "additionalProperties": False,
            },
        }
    },
)
```

<p>The same pattern covers every downstream artifact: the diagnosis record, the action proposal, the Slack feedback record, and eval results. In the Python SDK, <code>client.messages.parse()</code> and the Pydantic model return typed objects with the same guarantee. Three operational details matter in practice. The first request with a new schema pays a grammar&#8209;compilation cost; the compiled grammar is then cached for 24 hours from last use. The schema guarantee has two documented exceptions, a safety refusal and hitting <code>max_tokens</code>, so we check <code>stop_reason</code> before trusting a parse. Prompts and responses are still processed with zero data retention when structured outputs are enabled, which the runtime boundary depends on.</p>

<h2 id="investigation-runs-on-the-agent-sdk">The investigation runs on the Agent SDK</h2>

<p>The runtime decision came down to where session state lives. Managed Agents, in beta as of late April 2026, fits a long investigation well: durable sessions, mid&#8209;run steering, and scheduled runs, and moving there from an SDK prototype is <a href="https://code.claude.com/docs/en/agent-sdk/overview"> a common transition path</a>. We did not start there because Managed Agents keeps the session, the conversation history, its outputs, and tool execution on Anthropic's infrastructure. The hosted form is also ineligible for zero data retention. For a bank, those execution and retention boundaries decided the initial implementation design.</p>

<figure class="sre-figure">
<picture>
  <source media="(max-width: 900px)" srcset="/assets/ai-sre/runtime-boundary-mobile.svg">
  <img src="/assets/ai-sre/runtime-boundary.svg" alt="Runtime boundary: harness, tools, sessions, secrets, and the action gateway inside the bank; only inference at Anthropic; Managed Agents as the conditional path when residency terms fit" width="1240" height="520" loading="lazy" decoding="async">
</picture>
<figcaption>Prompts and completions are the only traffic that crosses the boundary. Session state, tools, secrets, and write authority stay on bank infrastructure. Managed Agents remains a conditional path when the residency terms fit.</figcaption>
</figure>

<p>The Agent SDK provides the same loop, tools, and context management that power Claude Code, running in our process, with session state as JSONL files we hold. Skills load from the filesystem, MCP servers attach as configuration, and hooks let us enforce policy in code around every tool call. A simplified version of the investigator setup:</p>

```python
from claude_agent_sdk import ClaudeAgentOptions, HookMatcher, query

async def enforce_read_only(input_data, tool_use_id, context):
    # This session has no write tools. All writes go through the
    # action gateway. Block anything that could mutate state, and
    # log every attempt.
    tool = input_data.get("tool_name", "")
    if tool in {"Write", "Edit", "Bash"}:
        return {
            "hookSpecificOutput": {
                "hookEventName": "PreToolUse",
                "permissionDecision": "deny",
                "permissionDecisionReason": "writes go through the action gateway",
            }
        }
    audit(tool_use_id, input_data)
    return {}

options = ClaudeAgentOptions(
    system_prompt=INVESTIGATOR_PROMPT,
    setting_sources=["project"],  # loads .claude/skills: global, domain, service
    mcp_servers={
        "obs": OBS_SERVER,
        "incidents": INDEX_SERVER,
        "sandbox": SANDBOX_SERVER,
    },
    allowed_tools=[
        "Read", "Grep", "Skill",  # the Skill tool activates the runbook catalog
        "mcp__obs__query_metrics",
        "mcp__obs__search_logs",
        "mcp__obs__get_recent_deploys",
        "mcp__incidents__search_similar",
        "mcp__sandbox__run_analysis",
    ],
    hooks={"PreToolUse": [HookMatcher(matcher=".*", hooks=[enforce_read_only])]},
)

async for message in query(prompt=envelope_json, options=options):
    handle(message)
```

<p>Steering uses session resume. The first message carries a <code>session_id</code>, and a follow&#8209;up resumes it with <code>ClaudeAgentOptions(resume=session_id)</code>. The existing session continues with everything it has already read and concluded. We keep one investigation session with a single accountable hypothesis thread. Parallel work happens inside the analysis code.</p>

<p>In this example, the operator asks whether Kafka lag caused the failure. The resumed session checks lag onset, consumer throughput, and deployment history. Three additional sources and two findings support the original conclusion: the canary remains the cause and Kafka lag is downstream. The mitigation rationale is then refreshed against the revised assessment.</p>

<figure class="sre-product-figure">
  <a href="/assets/ai-sre/product/operator-steering.png" target="_blank" rel="noopener" aria-label="Open the full-size operator steering example in a new tab">
    <img src="/assets/ai-sre/product/operator-steering.png" alt="Incident workspace after an operator asks whether Kafka lag caused the failure, showing a confirmed assessment, three additional sources, two added findings, and a mitigation that needs review" width="1422" height="800" loading="lazy" decoding="async">
  </a>
  <figcaption>The operator challenges the current cause from the incident workspace. Ada checks the timing again, records why Kafka lag is downstream, and refreshes the mitigation against the latest assessment.</figcaption>
</figure>

<p>Sessions that run for hours eventually compact. The incident envelope, confirmed findings, open hypotheses, and action state have to survive summarization. The harness writes those structured records to the session workspace and reads them back on demand. Compaction can then shorten the transcript without dropping the case. Traces mark any conclusion reached after compaction, and reviewers weigh those accordingly. Cross&#8209;incident recall has to be shared across responders, audited, and governed on our side of the boundary. We keep it in the incident index and expose it through retrieval.</p>
<p>Those same files also handle escalation, cheap to detect and rare to pay for. An investigation stalls when the session exhausts its tool&#8209;call and thinking budget without producing a corroborated hypothesis, meaning one supported by the deploy diff, the metrics, and the logs together. The harness detects this mechanically: the structured record it maintains has a dedicated field for that hypothesis, so when the budget runs out, the field is either filled or the session has stalled.</p>
<p>On a stall, the harness escalates to an Opus&#8209;class consult. The consult receives the case file with its confirmed facts and open hypotheses. Failed attempts remain in the transcript and are omitted from the consult input. The consult stays advisory. It returns a structured record specifying which reads to run next and which hypotheses to prioritize, and that record is appended to the session for the Sonnet&#8209;class investigator to act on when it resumes.</p>
<p>In practice, about one investigation in twenty stalls. Roughly two&#8209;thirds of those reach a corroborated hypothesis after the consult; the rest are handed to the on&#8209;call engineer with the case file attached, which is where they were headed anyway. And because Opus runs only on stalled sessions, the typical investigation never pays for it: the consult's cost sits in the tail of the distribution.</p>
<h2 id="keeping-evidence-out-of-the-context-window">Keeping evidence out of the context window</h2>

<p>An investigation touches a lot of data the model should never hold. Six hours of error logs might be tens of thousands of lines; the useful signal is that 340 of them are OOM kills and the first one landed 90 seconds after a deploy. Context is for reasoning: fetching, filtering, and counting happen in code next to the data, and only conclusions come back. Claude writes a small program that calls the read tools, filters and joins the results, and returns one compact evidence object, and only that object enters the context window. On the Messages API, the managed version of the pattern is programmatic tool calling, enabled by adding <code>allowed_callers</code> to a tool definition alongside the code execution tool:</p>

<figure class="sre-product-figure">
  <a href="/assets/ai-sre/product/evidence-review.png" target="_blank" rel="noopener" aria-label="Open the full-size evidence review in a new tab">
    <img src="/assets/ai-sre/product/evidence-review.png" alt="Evidence view showing checkout error rate, canary memory, downstream consumer lag, and the source records used in the assessment" width="1422" height="800" loading="lazy" decoding="async">
  </a>
  <figcaption>The evidence view shows the timing and source records behind the assessment. Raw telemetry is reduced before it enters the investigation context, while the operator keeps a path back to every source used.</figcaption>
</figure>

```python
tools = [
    {"type": "code_execution_20260120", "name": "code_execution"},
    {
        "name": "search_logs",
        "description": "Search service logs. Returns JSON rows with "
                       "fields: ts, level, message, pod.",
        "input_schema": {
            "type": "object",
            "properties": {
                "service": {"type": "string"},
                "level": {"type": "string"},
                "hours": {"type": "integer"},
            },
            "required": ["service", "hours"],
        },
        "allowed_callers": ["code_execution_20260120"],
    },
    # get_recent_deploys and query_metrics are defined the same way
]
```

<p>We keep telemetry on bank infrastructure. MCP&#8209;connector tools cannot be called programmatically, so the snippet above defines the reads as plain custom tools. The limit applies to the connector; the MCP servers attached through the Agent SDK are unaffected. The managed container also retains execution artifacts for up to 30 days. For this data, the investigator runs the same pattern behind one bank&#8209;side tool. <code>run_analysis</code> takes the program Claude writes, executes it in a network&#8209;restricted sandbox next to the read APIs, and returns only what it prints. Claude's side is unchanged:</p>

```python
# Code Claude writes; executes in the bank-side sandbox.
import json

deploys = json.loads(await get_recent_deploys({"service": "checkout", "hours": 6}))
logs = await search_logs({"service": "checkout", "level": "ERROR", "hours": 6})
oom = [line for line in logs.splitlines() if "OOMKilled" in line]

print(
    json.dumps(
        {
            "deploy_ids": [d["id"] for d in deploys],
            "oom_count": len(oom),
            "first_oom_ts": oom[0].split()[0] if oom else None,
        }
    )
)
```

<p>We reserve the pattern for the read‑heavy phase and keep the triage fast path on direct calls, since programmatic calling adds overhead when a turn makes only one or two calls. Anthropic’s published benchmark shows the same split: a token reduction on a large read‑heavy tool agent, a small penalty on low‑call workloads.</p>

<p>Tool Search handles a catalog too large to load up front. Deferred tools, through <code>defer_loading</code>, stay out of context until Claude searches for them. A session opens with the incident envelope, the safety instructions, and a handful of common tools. The remaining connectors load when the investigation needs them.</p>

<p>The same discipline shapes the cost model. Prompt caching holds the stable prefix, the system prompt, tool definitions, and Skill metadata, so an hours&#8209;long session pays for its methodology once. Triage runs on Haiku and the investigator runs on a Sonnet&#8209;class model. Evidence objects keep token spend tied to the number of findings and largely independent of the original log volume. Triage runs with extended thinking off, while the investigator carries a thinking budget sized to the case. Calls, models, and thinking depth are sized together.</p>

<p>Writes go the other way. Rather than letting the model chain five small mutations, each write is a single tool, <code>rollback_deployment</code> for example, that owns its preconditions, dry run, approval routing, idempotency key, and inverse action.</p>

<p>The programmatic tool calling documentation treats <code>allowed_callers</code> as model guidance without an enforcement guarantee. We therefore keep the security boundary in code. Tool annotations, prompts, and Skills carry no write authority.</p>

<p>The gateway checks every write independently, which also protects the system from prompt injection arriving through log lines and tickets. A tool result can inform a hypothesis and carries no permission. Injected text can still steer the investigation toward a plausible, wrong proposal for a tired human at 3 a.m. Every claim in a proposal therefore carries its source link, and a mitigation requires corroboration across multiple sources.</p>

<h2 id="runbooks-as-skills">Runbooks as Skills</h2>

<p>Service teams own the knowledge that makes an investigation good, so it ships as Skills in their repositories. The central prompt stays focused on the investigation method. A Skill is a folder with a <code>SKILL.md</code>. Claude matches the frontmatter against the task and loads the body only when it applies:</p>

<!-- prettier-ignore -->
```markdown
---
name: service-checkout
description: Investigate checkout-svc incidents. Use when an alert names
  checkout-svc or symptoms include payment failures at checkout.
---

# What this service does
# Failure signatures
# Key dashboards and queries
# Dependencies and ownership
# Safe mitigations
# Escalate or prohibit
```

<p>Skills compose by specificity. A checkout OOM session loads the global incident discipline, the OOM domain playbook, and the checkout service Skill. The global Skill says to diagnose before proposing a change. The domain Skill explains how to tell a lowered memory limit from a leak. The service Skill knows checkout runs on cost&#8209;constrained nodes, so the obvious fix, raising the limit, is the wrong one there. Until a Skill matches, it costs only its metadata in context, the name and description above, which is what lets the catalog grow to hundreds of runbooks without inflating every session.</p>

<p>A service does not need a Skill to onboard; it starts on the global and domain tiers and gets its own the first time an incident teaches it something. Teams write and ship these without the platform team in the loop. <code>sre skill from-incident INC-4821</code> scaffolds a draft from a closed postmortem, local replay through the Agent SDK runs it against the service's recorded incidents, and CI will not publish a Skill without a named owner and a passing eval set. That division of labor is what lets the catalog scale: we run the tooling, service teams own what is true about their services, and the signal we watch is whether a team's second Skill ships without anyone from our team involved.</p>

<p>Policy Skills describe data&#8209;access boundaries and change freezes so the model can reason about them. The hook and gateway enforce those rules. A Skill can improve policy reasoning and carries no authority to approve an action.</p>

<h2 id="measuring-whether-a-diagnosis-was-right">Measuring whether a diagnosis was right</h2>

<p>A root&#8209;cause hypothesis needs an external label. A fluent explanation can be wrong, and at the moment the agent posts it, the true cause is often unknown to the humans too. Each diagnosis therefore gets two labels at different times.</p>

<figure class="sre-figure">
<picture>
  <source media="(max-width: 900px)" srcset="/assets/ai-sre/two-truth-loop-mobile.svg">
  <img src="/assets/ai-sre/two-truth-loop.svg" alt="The two-truth loop: RCA in Slack, provisional label, verified postmortem truth, reconciliation into eval cases and Skill updates, feeding the next investigation" width="1240" height="470" loading="lazy" decoding="async">
</picture>
<figcaption>Engineers label the diagnosis in the incident thread while it is live. After the review closes, the verified cause is reconciled against that label, and the pair becomes an eval case plus a drafted Skill update for the owning team.</figcaption>
</figure>

<p>The provisional label is a one&#8209;click verdict in the incident thread, accurate, partially accurate, incorrect, or still investigating, with an optional correction. The verified label comes later, reconciled against the post&#8209;incident review's confirmed cause and fix. Every reconciled pair becomes an eval case, and the replay path depends on the artifact. Bounded calls, triage envelopes and judge scoring, rerun through Message Batches at half the interactive price. Full investigations replay through the Agent SDK against recorded tool results: every MCP read a live session makes is captured as a fixture, so a closed incident becomes a reproducible case a Skill change can be tested against before it ships. A new trajectory that requests an uncaptured read fails loudly and never reaches live systems. The capture layer turns incident history into tests. A replay is scored in layers, cheap checks first: the record validates against its schema, every read it claims exists in the fixtures, and no denied tool was attempted. The judge runs last because it is the expensive opinion.</p>

<p>Two reporting rules keep the numbers honest. We never merge unknown into incorrect, and we only compute accuracy over incidents with a verified final cause. Time&#8209;to&#8209;hypothesis is the metric the agent directly moves. MTTR stays the north star, although most of an incident's wall clock is spent on waiting, coordination, and recovery. We publish an MTTR effect only after a controlled comparison.</p>

<h2 id="verifying-each-diagnosis">Verifying each diagnosis</h2>

<p>The corroboration rule, that the deploy diff, the metrics, and the logs have to agree before anything posts, started as a sentence in the system prompt and a habit of whoever read the thread. The rule exists because an investigator can fixate on the first plausible cause. It is now a mechanism. Before a diagnosis posts to Slack or a proposal reaches the gateway, a single Haiku&#8209;class grading call re&#8209;reads the record, follows each claim's source link, and checks that the excerpt supports the claim. A failure bounces the record back into the session with the failing claim named. The bounce uses the same resume mechanism as a human steer and is capped at two. A second failed pass posts the diagnosis with a flagged status so the engineer can see and correct it.</p>

<p>A bounce is itself a typed record:</p>

```json
{
  "record": "dx_4821",
  "verdict": "bounce",
  "failing_claim": "onset precedes the 14:02 deploy",
  "check": "cited excerpt starts at 14:02; the claim needs the 13:47 canary window",
  "bounce_count": 1,
  "disposition": "resume, failing claim named"
}
```

<p>Two properties keep the grader inside the trust model. Its interface contains block and bounce. Approval lives in the gateway. Grader verdicts land in the eval store and reconcile against verified causes like any other label, so its error rate is measured. In a domain where being wrong has consequences, the verification logic between the model and execution is the part of the harness most worth owning. The grader is that logic promoted from a rule to a job, at about two cents a pass.</p>

<h2 id="gating-actions">Gating actions</h2>

<p>Reads are autonomous where data policy permits. Writes are transactions, and each action class has its own promotion requirements:</p>

<figure class="sre-figure">
<picture>
  <source media="(max-width: 900px)" srcset="/assets/ai-sre/autonomy-ladder-mobile.svg">
  <img src="/assets/ai-sre/autonomy-ladder.svg" alt="Autonomy ladder from read-only through R2, with R3 capabilities shown struck through: absent as tools by design" width="1240" height="560" loading="lazy" decoding="async">
</picture>
<figcaption>Autonomy widens down the ladder only when a class's evidence supports it. R3 sits apart because its tools were never built.</figcaption>
</figure>

<p>R0 covers paper actions, posting a summary, updating a ticket, opening a draft PR, which run automatically with an audit trail because they do not touch production state. R1 covers bounded reversible writes behind approval, a dry run, and a tested inverse. R2 covers wider changes like rollbacks and traffic shifts, which add senior approval and an explicit blast&#8209;radius report. R3 covers irreversible or money&#8209;affecting actions, and its tools do not exist. The agent can recommend a ledger correction; a human executes it through existing systems. Promotion between tiers happens per action class and requires evidence: enough verified cases, eval thresholds met, shadow&#8209;mode performance, and low override rates. A high confidence score routes a case within its tier and never moves it across one.</p>

<figure class="sre-product-figure">
  <a href="/assets/ai-sre/product/mitigation-review.png" target="_blank" rel="noopener" aria-label="Open the full-size mitigation review in a new tab">
    <img src="/assets/ai-sre/product/mitigation-review.png" alt="Mitigation review showing a checkout rollback, five passed safety checks, expected impact, senior approval, expiry, and an approve-and-apply control" width="1422" height="800" loading="lazy" decoding="async">
  </a>
  <figcaption>The review exposes the target version, expected impact, safety checks, expiry, and required approver before a change can proceed. The companion interface uses fictional service state. The rollout described here keeps R2 execution in shadow.</figcaption>
</figure>

<p>The proposal that reaches the gateway is itself a structured output, and strict tool use holds the write tools' inputs to their schemas the same way, so the record is typed end to end. The gateway accepts typed records through one interface, which keeps approval, replay, and audit uniform across action classes:</p>

```json
{
  "proposal_id": "prop_01j9k2",
  "incident": "INC-4821",
  "action": "rollback_deployment",
  "target": {
    "service": "checkout-svc",
    "from": "2026.04.22-3",
    "to": "2026.04.22-2"
  },
  "risk_tier": "R2",
  "preconditions": {
    "deploy_correlated": true,
    "change_freeze": false,
    "inverse_tested": true
  },
  "dry_run": {
    "status": "passed",
    "blast_radius": "1 service, 12 pods, no dependent config"
  },
  "idempotency_key": "inc-4821-rollback-1",
  "approval": {
    "route": "senior-oncall",
    "state": "pending"
  },
  "expires_at": "2026-04-22T14:40:00Z"
}
```

<p>That record was generated in shadow, because R2 still runs there: every R2 proposal is evaluated end to end, dry run included, while execution is withheld. R1 is past that stage; restart, scale, and merge execute in production behind approval, each with its dry run and tested inverse.</p>

<h2 id="diagnosis-as-a-capability">Diagnosis as a capability</h2>

<p>The investigator's surface was already a function: a typed envelope in, a typed and sourced diagnosis out, with reads in between. In April we wrapped that function in an MCP server so other agents could call it. Alerts are one client of the capability. The deploy pipeline's agent asks whether its rollout caused the errors it is watching before deciding to proceed; the capacity planner asks for six hours of history on a service before recommending a scale&#8209;down; the next internal assistant gets root&#8209;cause analysis without rebuilding it.</p>

<p>The trust model stays the same for agent&#8209;called investigations. The caller receives a hypothesis and its evidence. Proposals still exit through the gateway, and agent&#8209;called sessions are admitted below alert&#8209;triggered ones so a chatty caller cannot starve an incident. The measurement contract adds a third category for these sessions alongside live and shadow. They are reported separately and stay outside the incident counts.</p>

<h2 id="when-the-assistant-is-down">When the assistant is down</h2>

<p>An AI SRE has to plan for its own absence. The assistant is most likely to be impaired during the largest incidents, when regional trouble degrades the same networks and providers everything else depends on. The incident process ran for years without it and still can. If the envelope call misses its timeout budget after one retry, the alert routes exactly as it did before. If a session dies mid&#8209;investigation, the thread says so and the on&#8209;call keeps working. A circuit breaker on the API path drops the whole system to the human&#8209;only flow and posts one line saying it did. The fast path and the investigator fail independently, and paging, severity, and escalation continue without either. Provider capacity remains a residual exposure. The system uses committed Priority Tier capacity, which targets 99.5 percent uptime with prioritized compute and falls back to the standard tier past the commitment.</p>

<p>Alert storms get the same treatment. A regional event can fire hundreds of alerts in minutes, and starting hundreds of investigations would be an incident of its own. Envelopes deduplicate on service and error signature into one investigation with a fan‑in list, concurrent sessions are capped and admitted by severity, and separate workspaces with independent rate limits keep triage from starving behind investigation traffic.</p>

<h2 id="keeping-it-current">Keeping it current</h2>

<p>The failure mode we planned for first is stale knowledge. When a post&#8209;incident review closes, the incident index updates, an eval case is created, and a single Messages API call drafts the corresponding Skill update with links back to the postmortem. The draft goes to the owning team as a pull request and requires review, because postmortems can be incomplete or wrong too. Change&#8209;impact mapping sends a checkout Skill change through the checkout and OOM cases. The full corpus runs on a schedule and on higher&#8209;risk changes.</p>

<p>Model upgrades get the same treatment as any other risky change. We rerun the full eval suite, compare tool trajectories and costs, and shadow before promoting, because a newer model can be better on average while quietly changing a behavior a workflow depended on. We treat a model swap as a migration. A headline metric can hold while the tool trajectory shifts underneath it, so the review covers trajectories and costs alongside accuracy.</p>

<p>The harness targets one model family at a time. A migration review asks what changed and which older steering text can be removed. Early prompt passages corrected habits of a weaker model, and some exemplars demonstrated formats that structured outputs now guarantee. In the last migration, the eval ran with the existing harness and with three such passages removed, including the sentence telling the model to avoid writes. The smaller harness matched and promoted. The hook continued to deny writes, and the denial counter stayed flat through the change. Authority stays in code and is excluded from migration experiments. This pruning keeps the harness aligned with the model it currently runs.</p>

<h2 id="measurement-contract">The measurement contract</h2>

<p>The design published before its operational numbers, so we fixed the measurement contract before compiling the inventory. An investigation is a session triggered by a real alert; replay executions in CI stay outside that count. Live means the diagnosis posted into an incident thread where an engineer could label or redirect it; shadow means it ran without posting. A session started by another agent through the diagnosis surface is agent&#8209;called and reported separately. A steer is a substantive human redirect. Bare resumes and label clicks do not count. A grader bounce is machine&#8209;initiated and counts as neither a steer nor a label. The eval corpus is the curated, deduplicated set used to test a change, with raw fixture capture reported separately. Reporting windows are trailing 90 days, with cumulative totals only where a store genuinely accumulates.</p>

<p>And here is what any published figures must satisfy, checkable by a reader. Reconciled pairs are fewer than labeled diagnoses, which are fewer than live investigations. Dry runs equal proposals. Executions never exceed proposals, flagged posts never exceed grader bounces, and R2 executions are zero until R2 is promoted, which has not happened and has no date.</p>

<p>The label rate publishes. The label distribution stays private because it would reconstruct an accuracy figure over an unverified denominator. The reconciliation crosstab publishes as a direction and design response, without percentages. Every figure ships rounded, dated, with its definition attached and its query retained. A number that fails these terms stays out of publication.</p>

<h2 id="where-it-stands">Where it stands</h2>

<p>As of April 2026, the rollout is deliberately sequenced: the v2 rolled out in phases behind the beta, earliest pieces first. The fast path and the investigator came up before any write capability. The action gateway is live for R0 paper actions and, having cleared its shadow period, for R1 reversible writes behind approval; R2 remains in shadow with no promotion date, deliberately. The eval library grows with every closed review.</p>

<p>The boring inventory, under the contract above. Over the trailing 90 days the agent ran roughly 96,000 investigations on real alerts, about 71,000 live in incident threads and 25,000 in shadow, across 240 of 264 tier&#8209;1 services, with about 55 tier&#8209;2 services in early rollout; the cumulative total since rollout began is around one million. About 12,000 live sessions, one in six, were steered mid&#8209;investigation. About 4,800 investigations stalled into an Opus consult, and about 3,200 of those reached a corroborated hypothesis on the resume. Since the diagnosis surface opened in April, other agents have started roughly 1,900 agent&#8209;called sessions, which the contract reports outside the investigation counts here.</p>

<p>The registry holds about 760 Skills, 6 global, 34 domain, and roughly 720 service, with 86 percent of the service Skills carrying no platform&#8209;team commits, and about 180 teams have shipped a second Skill with no platform&#8209;team commits or reviews, the signal we actually watch.</p>

<p>The grader bounced roughly 6,900 diagnoses at least once before they posted, and about 900 posted flagged after a second failed pass. 64 percent of live diagnoses received a one&#8209;click label. The eval store holds about 9,400 curated fixture&#8209;backed incidents and 5,200 reconciled pairs, both cumulative, with raw capture running far larger and continuous. Over that verified set, the diagnosis matched the postmortem's confirmed cause 96 percent of the time.</p>

<p>On the action side, about 168,000 R0 paper actions executed over the window, mostly summaries and ticket updates plus around 12,000 draft pull requests. Roughly 4,900 R1 proposals were generated, every one dry&#8209;run, and about 3,400 executed in production behind approval; the rest were rejected or expired at the gate. About 1,300 R2 proposals ran end to end in shadow, dry runs matching proposals one for one, with zero executions. The read&#8209;only hook denied and logged about 2,600 write attempts, most of them the model reaching for scratch files or a shell mid&#8209;investigation, and about 180 of them attempts to bypass the proposal path and apply a fix directly.</p>

<p>The median investigation costs about 70 cents, the ninetieth percentile about $3.20 for the long ones, and triage stays under two cents on Haiku. The circuit breaker has dropped the system to the human&#8209;only flow 14 times since rollout, twice in the window, and about 2,300 alert storms collapsed into single investigations, the largest folding roughly 4,800 alerts into one with a fan&#8209;in list.</p>

<p>Two numbers we do not report: an MTTR improvement, because we do not yet have a controlled comparison, and any accuracy figure over incidents without a verified cause.</p>

<h2 id="where-it-goes-next">Where it goes next</h2>

<p>The system splits work into separately measurable jobs. One classifies, one investigates, one verifies each diagnosis, one advises a stalled session, one scores replays, and one writes incident lessons back into the Skill catalog through pull requests. Each job has a narrow responsibility, a typed output, and bounded authority. Diagnosis itself now follows that shape and can be called by other agents. The next experiment asks whether multiple independent investigations improve the hardest cases.</p>

<p>Teams usually tune two levers once an agent works: model choice and run length. A third option is to run several independent attempts and select among them. We can test that strategy through replay before using it on live incidents. The eval store holds 5,200 reconciled incidents whose fixtures answer every read and whose true cause is verified. The experiment replays a stratified sample three ways through the Agent SDK with the same envelope and fixtures. A judge scores each final hypothesis against the verified cause and remains blind to attempt identity. Three runs across a 300&#8209;incident sample cost roughly six hundred dollars at the median investigation price.</p>

<figure class="sre-figure">
<picture>
  <source media="(max-width: 900px)" srcset="/assets/ai-sre/best-of-three-mobile.svg">
  <img src="/assets/ai-sre/best-of-three.svg" alt="Best-of-N experiment: the eval store's envelope and fixtures feed three independent investigator replays; a judge scores each final hypothesis against the verified cause, blind to attempt identity, and a pre-registered rule decides between adoption for sev1 and falsification" width="1240" height="560" loading="lazy" decoding="async">
</picture>
<figcaption>A stratified sample of reconciled incidents replays three ways against recorded fixtures. The judge alone sees the verified cause and scores blind to attempt identity, and the adoption rule is written before the first replay.</figcaption>
</figure>

<p>The measurement contract's habit applies to experiments too, so the protocol is fixed before the first replay:</p>

```text
# fixed before the first replay runs
sample        300 reconciled incidents, stratified by severity and tier
attempts      3 per incident; independent sessions; identical inputs
judge         corroboration rubric, scored against the verified cause,
              blind to which attempt produced which hypothesis
comparison    judge-selected best-of-3 vs. attempt 1 alone
adopt when    sev1 accuracy improves by 2+ points and the two added
              attempts stay under 5% of fleet investigation spend
either way    the result publishes with this protocol attached
```

<p>The 96 percent verified result leaves little headroom, so the experiment focuses on sev1 cases where single runs miss and added cost can be justified. The adoption rule uses a fleet spend ceiling because three attempts triple investigation cost by construction. If the result clears the bar, sev1 incidents receive several independent hypothesis threads and a selection pass. Any disagreement between those threads remains visible to the on&#8209;call engineer. If the result falls short, the current single&#8209;thread design stays. Each attempt still carries one accountable hypothesis; the judge compares their outputs after the investigations finish.</p>

<p>The conditional box in the boundary figure remains conditional as of late April. Managed Agents provides durable sessions, compaction, resume, and scheduling, while the session, transcript, and tool execution stay hosted. The Agent SDK keeps those elements on customer infrastructure. The gateway, incident index, eval store, and Skills registry remain customer&#8209;owned in either design. We would validate any runtime change by replaying the fixture corpus on both forms and promoting after they match.</p>

<h2 id="takeaways">Takeaways</h2>

<ol class="takeaways">
<li>Choose the runtime by deployment constraint first, capability second. For us, session&#8209;state residency settled the question before feature comparison started, which put the investigation on the Agent SDK, with Managed Agents as the hosted form of the same design.</li>
<li>Right&#8209;size every call. Bounded tasks run as single Messages API calls with structured outputs. The open&#8209;ended investigation uses an agent session, and the largest model is reserved for stalled cases.</li>
<li>Keep raw telemetry out of the context window. Code performs the filtering through programmatic tool calling where retention terms fit and a bank&#8209;side sandbox where they do not. Tool Search limits the initial catalog, and evidence objects keep telemetry dumps out of model context.</li>
<li>Put procedures in Skills and authority in code. A Skill, a prompt, or a tool annotation can make the agent smarter; none of them can authorize an action.</li>
<li>Keep approval outside the checker. The grader bounces unsupported claims back into the session, posts a flagged result after repeated failure, and reconciles its verdicts like any other label.</li>
<li>Label every diagnosis twice, once in the moment and once against the verified postmortem, and only compute accuracy over the verified set.</li>
<li>The Skill catalog scales through ownership. Postmortems draft the updates, CI gates publishing, and service teams keep the content true.</li>
<li>Test strategy claims on the corpus before adopting them. A replayable, verified incident set turns a roadmap debate, including best&#8209;of&#8209;N, into a pre&#8209;registered experiment with a fixed replay budget.</li>
</ol>

<h2 id="referenced-sources-and-further-reading">Referenced sources and further reading</h2>

<ul class="reading">
<li><a href="https://www.anthropic.com/engineering/managed-agents">Scaling Managed Agents: Decoupling the brain from the hands</a></li>
<li><a href="https://www.anthropic.com/engineering/advanced-tool-use">Advanced tool use</a> and the <a href="https://platform.claude.com/docs/en/agents-and-tools/tool-use/programmatic-tool-calling">programmatic tool calling</a> reference</li>
<li><a href="https://www.anthropic.com/engineering/writing-tools-for-agents">Writing effective tools for agents</a></li>
<li><a href="https://www.anthropic.com/engineering/code-execution-with-mcp">Code execution with MCP</a></li>
<li><a href="https://code.claude.com/docs/en/agent-sdk/overview">Claude Agent SDK overview</a></li>
<li><a href="https://platform.claude.com/docs/en/build-with-claude/structured-outputs">Structured outputs</a></li>
<li><a href="https://platform.claude.com/cookbook/managed-agents-sre-incident-responder">The SRE incident responder</a> in the Managed Agents cookbook and <a href="https://platform.claude.com/cookbook/claude-agent-sdk-03-the-site-reliability-agent">the site reliability agent</a> in the Agent SDK series</li>
<li><a href="https://platform.claude.com/docs/en/agents-and-tools/agent-skills/overview">Agent Skills overview</a></li>
<li><a href="https://www.anthropic.com/engineering/demystifying-evals-for-ai-agents">Demystifying evals for AI agents</a></li>
<li><a href="https://sequoiacap.com/podcast/anthropics-katelyn-lesse-angela-jiang-building-an-ecosystem-not-a-walled-garden/">Building an Ecosystem, not a Walled Garden</a>, Katelyn Lesse and Angela Jiang on Sequoia's Training Data</li>
</ul>

<p class="sre-footnote">Ada, the read&#8209;only beta, predates this work and proved the reasoning use case. The architecture here is the v2 design and phased implementation I led, with incident&#8209;response, observability, security, and service teams owning their parts. Platform statuses and API syntax checked against Anthropic documentation, April 2026.</p>

</div>
