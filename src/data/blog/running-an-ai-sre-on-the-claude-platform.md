---
author: Ivaylo Bahtchevanov
pubDatetime: 2026-07-12T10:00:00Z
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
    position: relative;
    left: 50%;
    width: min(1160px, 96vw);
    margin: 2.75rem 0;
    transform: translateX(-50%);
  }

  #article .sre-article figure svg {
    display: block;
    width: 100%;
    height: auto;
    padding: 0.75rem;
    border: 1px solid #e4e0d6;
    border-radius: 0.75rem;
    background: #fbfaf7;
  }

  #article .sre-article figure svg.m {
    display: none;
  }

  #article .sre-article figcaption,
  #article .sre-article .threadcap {
    max-width: 42rem;
    margin: 0.75rem auto 0;
    color: color-mix(in srgb, var(--foreground) 68%, transparent);
    font-family: ui-sans-serif, system-ui, sans-serif;
    font-size: 0.85rem;
    line-height: 1.55;
    text-align: center;
  }

  #article .sre-article .thread {
    margin: 1.6rem 0 0.5rem;
    overflow: hidden;
    border: 1px solid var(--border);
    border-radius: 0.75rem;
    background: var(--background);
    color: var(--foreground);
    font-family: ui-sans-serif, system-ui, sans-serif;
  }

  #article .sre-article .msg {
    display: flex;
    gap: 0.75rem;
    padding: 0.85rem 1rem;
  }

  #article .sre-article .msg + .msg {
    border-top: 1px solid var(--border);
  }

  #article .sre-article .avatar {
    flex: 0 0 1.9rem;
    height: 1.9rem;
    border-radius: 0.5rem;
    color: white;
    font-size: 0.75rem;
    font-weight: 600;
    line-height: 1.9rem;
    text-align: center;
  }

  #article .sre-article .a-bot {
    background: #141412;
  }

  #article .sre-article .a-eng {
    background: #d97757;
    color: #1a1a17;
  }

  #article .sre-article .tbody {
    font-size: 0.9rem;
    line-height: 1.55;
  }

  #article .sre-article .name {
    margin-bottom: 0.15rem;
    font-size: 0.8rem;
    font-weight: 600;
  }

  #article .sre-article .name span {
    margin-left: 0.4rem;
    opacity: 0.65;
    font-size: 0.75rem;
    font-weight: 400;
  }

  #article .sre-article .pills {
    display: flex;
    flex-wrap: wrap;
    gap: 0.35rem;
    margin-top: 0.55rem;
  }

  #article .sre-article .pill {
    border: 1px solid var(--border);
    border-radius: 999px;
    padding: 0.2rem 0.6rem;
    font-size: 0.75rem;
  }

  #article .sre-article .pill.sel {
    border-color: #d97757;
    background: #f7e6dc;
    color: #3d2417;
  }

  #article .sre-article pre code {
    display: block;
    white-space: pre;
  }

  #article .sre-article .sre-footnote {
    margin-top: 3.5rem;
    padding-top: 1.1rem;
    border-top: 1px solid var(--border);
    opacity: 0.72;
    font-family: ui-sans-serif, system-ui, sans-serif;
    font-size: 0.82rem;
    line-height: 1.6;
  }

  @media (max-width: 640px) {
    #article .sre-article figure svg.d {
      display: none;
    }

    #article .sre-article figure svg.m {
      display: block;
    }

    #article .sre-article figure svg {
      padding: 0.35rem;
    }
  }
</style>

<div class="sre-article">
<p>Incident response at Nubank is largely automated. Bots open the incident, propose severity, create the Slack war room, page responders, file tickets, and schedule the review. The step that still depended on a senior engineer was investigation: using judgment and experience to turn the metrics, logs, traces, deploy history, and past incidents into a credible root&#8209;cause hypothesis, often at 3 a.m.</p>

<p>In 2025 we shipped a read&#8209;only investigation agent, internally called Ada, on a custom-built harness, because nothing hosted at the time had the durable, steerable sessions an investigation needs. Ada worked well enough to expose the problems that matter at the next stage. It could not search past incidents, could not be redirected mid&#8209;investigation, could not act on its conclusions, ran on infrastructure that consumed most of a two&#8209;person team, and had no reliable way to measure whether its diagnoses were correct.</p>

<p>This post describes the v2 design. Structured outputs type every artifact another system parses, the Agent SDK runs the investigation loop, MCP carries the operational reads, code in the middle keeps raw telemetry out of the model’s context, and Skills hold the runbooks service teams own. Two constraints shaped the design more than any feature choice: where session state can live, and which actions exist for the agent at all. Anthropic's cookbooks include incident‑response agents on both runtimes and cover the basic mechanics; the difference here is everything a production system has to implement before the agent touches a real incident.</p>

<h2 id="architecture-at-a-glance">The architecture at a glance</h2>

<figure>
<svg class="d" viewBox="0 0 1240 540" role="img" aria-label="End-to-end architecture: alert to triage on the Messages API, to an Agent SDK investigator, to a gated action plane, over shared bank-owned components that persist across sessions">
  <defs>
    <marker id="d1k" viewBox="0 0 10 10" refX="8.5" refY="5" markerWidth="7" markerHeight="7" orient="auto-start-reverse"><path d="M0 0L10 5L0 10z" fill="#1a1a17"/></marker>
    <marker id="d1c" viewBox="0 0 10 10" refX="8.5" refY="5" markerWidth="7" markerHeight="7" orient="auto-start-reverse"><path d="M0 0L10 5L0 10z" fill="#D97757"/></marker>
  </defs>

  <rect x="24" y="112" width="156" height="56" rx="28" fill="#ffffff" stroke="#1a1a17" stroke-width="1.5"/>
  <text x="102" y="146" text-anchor="middle" font-family="Georgia,serif" font-size="17" fill="#1a1a17">Alert / Slack</text>

  <line x1="182" y1="140" x2="240" y2="140" stroke="#1a1a17" stroke-width="1.5" marker-end="url(#d1k)"/>
  <text x="211" y="128" text-anchor="middle" font-family="ui-monospace,Menlo,monospace" font-size="12" fill="#6b6b66">alert</text>

  <rect x="248" y="70" width="248" height="148" rx="18" fill="#ffffff" stroke="#1a1a17" stroke-width="1.5"/>
  <text x="372" y="120" text-anchor="middle" font-family="Georgia,serif" font-size="25" font-weight="600" fill="#1a1a17">Triage</text>
  <text x="372" y="146" text-anchor="middle" font-family="-apple-system,'Segoe UI',sans-serif" font-size="13" fill="#6b6b66">Messages API &#8594; typed envelope</text>
  <line x1="278" y1="164" x2="466" y2="164" stroke="#e4e0d6" stroke-width="1"/>
  <text x="372" y="190" text-anchor="middle" font-family="ui-monospace,Menlo,monospace" font-size="11" letter-spacing="1" fill="#6b6b66">runs once · seconds</text>

  <line x1="498" y1="140" x2="552" y2="140" stroke="#1a1a17" stroke-width="1.5" marker-end="url(#d1k)"/>
  <text x="525" y="128" text-anchor="middle" font-family="ui-monospace,Menlo,monospace" font-size="12" fill="#6b6b66">envelope</text>

  <rect x="560" y="70" width="264" height="148" rx="18" fill="#141412"/>
  <text x="692" y="120" text-anchor="middle" font-family="Georgia,serif" font-size="25" font-weight="600" fill="#ffffff">Investigator</text>
  <text x="692" y="146" text-anchor="middle" font-family="-apple-system,'Segoe UI',sans-serif" font-size="13" fill="#c9c4b8">Agent SDK session · steerable</text>
  <line x1="590" y1="164" x2="794" y2="164" stroke="#3a3a36" stroke-width="1"/>
  <text x="692" y="190" text-anchor="middle" font-family="ui-monospace,Menlo,monospace" font-size="11" letter-spacing="1" fill="#9b968a">read-only · minutes&#8211;hours</text>

  <line x1="826" y1="118" x2="880" y2="118" stroke="#1a1a17" stroke-width="1.5" marker-end="url(#d1k)"/>
  <text x="856" y="92" text-anchor="middle" font-family="ui-monospace,Menlo,monospace" font-size="11" fill="#6b6b66"><tspan x="856">rca +</tspan><tspan x="856" dy="14">proposal</tspan></text>
  <line x1="880" y1="170" x2="830" y2="170" stroke="#D97757" stroke-width="1.5" marker-end="url(#d1c)"/>
  <text x="856" y="188" text-anchor="middle" font-family="ui-monospace,Menlo,monospace" font-size="11" fill="#a14e2f"><tspan x="856">recovery</tspan><tspan x="856" dy="14">signals</tspan></text>

  <rect x="888" y="70" width="264" height="148" rx="18" fill="#D97757"/>
  <text x="1020" y="120" text-anchor="middle" font-family="Georgia,serif" font-size="25" font-weight="600" fill="#1a1a17">Action plane</text>
  <text x="1020" y="146" text-anchor="middle" font-family="-apple-system,'Segoe UI',sans-serif" font-size="13" fill="#3d2417">dry&#8209;run &#8594; approve &#8594; execute</text>
  <line x1="918" y1="164" x2="1122" y2="164" stroke="#c06342" stroke-width="1"/>
  <text x="1020" y="190" text-anchor="middle" font-family="ui-monospace,Menlo,monospace" font-size="11" letter-spacing="1" fill="#3d2417">R0&#8211;R2 only · audited</text>

  <line x1="372" y1="222" x2="372" y2="316" stroke="#b9b4a9" stroke-width="1.2" stroke-dasharray="3 5"/>
  <line x1="692" y1="222" x2="692" y2="316" stroke="#b9b4a9" stroke-width="1.2" stroke-dasharray="3 5"/>
  <line x1="1020" y1="222" x2="1020" y2="316" stroke="#b9b4a9" stroke-width="1.2" stroke-dasharray="3 5"/>

  <rect x="110" y="320" width="1040" height="92" rx="16" fill="#F1EAE0"/>
  <text x="140" y="356" font-family="ui-monospace,Menlo,monospace" font-size="12" font-weight="700" letter-spacing="1.5" fill="#1a1a17">PERSISTENCE ACROSS SESSIONS</text>
  <text x="140" y="378" font-family="-apple-system,'Segoe UI',sans-serif" font-size="12" fill="#6b6b66">bank-owned shared state</text>

  <rect x="392" y="349" width="140" height="34" rx="8" fill="#ffffff" stroke="#ddd6c8"/>
  <text x="462" y="371" text-anchor="middle" font-family="ui-monospace,Menlo,monospace" font-size="12" fill="#1a1a17">incident index</text>
  <rect x="548" y="349" width="140" height="34" rx="8" fill="#ffffff" stroke="#ddd6c8"/>
  <text x="618" y="371" text-anchor="middle" font-family="ui-monospace,Menlo,monospace" font-size="12" fill="#1a1a17">skills registry</text>
  <rect x="704" y="349" width="110" height="34" rx="8" fill="#ffffff" stroke="#ddd6c8"/>
  <text x="759" y="371" text-anchor="middle" font-family="ui-monospace,Menlo,monospace" font-size="12" fill="#1a1a17">eval store</text>
  <rect x="830" y="349" width="150" height="34" rx="8" fill="#ffffff" stroke="#ddd6c8"/>
  <text x="905" y="371" text-anchor="middle" font-family="ui-monospace,Menlo,monospace" font-size="12" fill="#1a1a17">action gateway</text>

  <text x="620" y="474" text-anchor="middle" font-family="-apple-system,'Segoe UI',sans-serif" font-size="15" fill="#55534d">Engineers only see and interact with one agent. These four components live across sessions.</text>
</svg>
<svg class="m" viewBox="0 0 620 690" role="img" aria-label="Mobile view: alert to triage to investigator to action plane, over shared bank-owned state">
  <defs><marker id="m1k" viewBox="0 0 10 10" refX="8.5" refY="5" markerWidth="8" markerHeight="8" orient="auto-start-reverse"><path d="M0 0L10 5L0 10z" fill="#1a1a17"/></marker></defs>
  <rect x="40" y="14" width="540" height="88" rx="20" fill="#ffffff" stroke="#1a1a17" stroke-width="1.6"/>
  <text x="310" y="52" text-anchor="middle" font-family="Georgia,serif" font-size="26" font-weight="600" fill="#1a1a17">Alert / Slack</text>
  <text x="310" y="82" text-anchor="middle" font-family="-apple-system,'Segoe UI',sans-serif" font-size="18" fill="#6b6b66">webhook or thread mention</text>
  <line x1="310" y1="106" x2="310" y2="140" stroke="#1a1a17" stroke-width="1.6" marker-end="url(#m1k)"/>
  <rect x="40" y="144" width="540" height="88" rx="20" fill="#ffffff" stroke="#1a1a17" stroke-width="1.6"/>
  <text x="310" y="182" text-anchor="middle" font-family="Georgia,serif" font-size="26" font-weight="600" fill="#1a1a17">Triage</text>
  <text x="310" y="212" text-anchor="middle" font-family="-apple-system,'Segoe UI',sans-serif" font-size="18" fill="#6b6b66">Messages API, typed envelope</text>
  <line x1="310" y1="236" x2="310" y2="270" stroke="#1a1a17" stroke-width="1.6" marker-end="url(#m1k)"/>
  <rect x="40" y="274" width="540" height="88" rx="20" fill="#141412"/>
  <text x="310" y="312" text-anchor="middle" font-family="Georgia,serif" font-size="26" font-weight="600" fill="#ffffff">Investigator</text>
  <text x="310" y="342" text-anchor="middle" font-family="-apple-system,'Segoe UI',sans-serif" font-size="18" fill="#c9c4b8">Agent SDK session, read-only</text>
  <line x1="310" y1="366" x2="310" y2="400" stroke="#1a1a17" stroke-width="1.6" marker-end="url(#m1k)"/>
  <rect x="40" y="404" width="540" height="88" rx="20" fill="#D97757"/>
  <text x="310" y="442" text-anchor="middle" font-family="Georgia,serif" font-size="26" font-weight="600" fill="#1a1a17">Action plane</text>
  <text x="310" y="472" text-anchor="middle" font-family="-apple-system,'Segoe UI',sans-serif" font-size="18" fill="#3d2417">dry run, approve, execute</text>
  <line x1="310" y1="496" x2="310" y2="528" stroke="#1a1a17" stroke-width="1.6" marker-end="url(#m1k)"/>
  <rect x="24" y="534" width="572" height="126" rx="16" fill="#F1EAE0"/>
  <text x="52" y="572" font-family="ui-monospace,Menlo,monospace" font-size="15" font-weight="700" letter-spacing="1.5" fill="#1a1a17">PERSISTS ACROSS SESSIONS</text>
  <text x="52" y="602" font-family="-apple-system,'Segoe UI',sans-serif" font-size="17" fill="#55534d">incident index &#183; skills registry</text>
  <text x="52" y="628" font-family="-apple-system,'Segoe UI',sans-serif" font-size="17" fill="#55534d">eval store &#183; action gateway</text>
</svg>
<figcaption>An alert acts as a typed envelope on the Messages API, an Agent SDK session investigates it, and any proposed action crosses a separate gated plane. Four shared components persist across sessions.</figcaption>
</figure>

<p>An alert enters through a fast path on the Messages API, which classifies it and emits a typed incident envelope. The envelope starts an Agent SDK session inside our infrastructure. The session loads the relevant Skills, gathers evidence through MCP read tools, clears a grading pass, and posts a structured diagnosis into the incident's Slack thread, where the on&#8209;call engineer can label it or redirect it. If the diagnosis warrants action, the proposal goes to a separate action plane, where a human approves a dry run before anything executes. Four components sit under all of this and outlive any single session: the incident index, the Skills registry, the eval store, and the action gateway.</p>

<p>We split the design by whether the work can be bounded in advance. Triage, judge scoring, and the grader each have clearly defined input and output, so they run as single Messages API calls with structured outputs and no session state. The investigation cannot be bounded in the same way so it runs as an Agent SDK session, with state held as files on our infrastructure. Runbooks are Skills, maintained in the owning team's repo. Writes go through the action gateway as typed proposals.</p>


<h2 id="triage-runs-on-the-messages-api">Triage runs on the Messages API</h2>


<p>A single API call classifies the alert, proposes a severity, and normalizes the payload into the incident envelope. The envelope starts a session, the judge&#8217;s score files into the eval store, and the grader&#8217;s verdict decides whether a diagnosis posts; the first reader of each is software, so a response that almost parses is a failure. Structured outputs remove that failure class at the API. The envelope schema goes in <code>output_config.format</code> and the response is guaranteed to validate against it:</p>

<pre><code>from anthropic import Anthropic

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
                    "severity": {"type": "string", "enum": ["sev1", "sev2", "sev3"]},
                    "error_signature": {"type": "string"},
                    "affected_components": {"type": "array", "items": {"type": "string"}},
                    "window_start": {"type": "string"},
                },
                "required": ["service", "severity", "error_signature",
                             "affected_components", "window_start"],
                "additionalProperties": False,
            },
        }
    },
)</code></pre>

<p>The same pattern covers every downstream artifact: the diagnosis record, the action proposal, the Slack feedback record, and eval results. In the Python SDK, <code>client.messages.parse()</code> along with the Pydantic model give the same guarantee with typed objects instead of raw JSON. Three operational details matter in practice. The first request with a new schema pays a grammar&#8209;compilation cost; the compiled grammar is then cached for 24 hours from last use. The schema guarantee has two documented exceptions, a safety refusal and hitting <code>max_tokens</code>, so we check <code>stop_reason</code> before trusting a parse. And prompts and responses are still processed with zero data retention when structured outputs are enabled, which the runtime boundary depends on.</p>

<h2 id="investigation-runs-on-the-agent-sdk">The investigation runs on the Agent SDK</h2>


<p>The runtime decision came down to where session state lives. Managed Agents, in beta as of this writing, fits a long investigation well: durable sessions, mid&#8209;run steering, and scheduled runs, and moving there from an SDK prototype is <a href="https://code.claude.com/docs/en/agent-sdk/overview"> a common transition path</a>. We did not start there because Managed Agents keeps the session, the conversation history and its outputs, on Anthropic's infrastructure. The self&#8209;hosted sandbox release in May moved tool execution inside the customer perimeter; the loop and its transcript stay hosted. That design is what makes those features possible, and it is also why the hosted form is not eligible for zero data retention. For a bank, that constraint decided the initial implementation design.</p>

<figure>
<svg class="d" viewBox="0 0 1240 520" role="img" aria-label="Runtime boundary: harness, tools, sessions, secrets, and the action gateway inside the bank; only inference at Anthropic; Managed Agents with a self-hosted sandbox as the conditional path">
  <defs>
    <marker id="d2k" viewBox="0 0 10 10" refX="8.5" refY="5" markerWidth="7" markerHeight="7" orient="auto-start-reverse"><path d="M0 0L10 5L0 10z" fill="#1a1a17"/></marker>
  </defs>

  <rect x="32" y="64" width="756" height="372" rx="20" fill="#ffffff" stroke="#1a1a17" stroke-width="1.5"/>
  <text x="60" y="100" font-family="ui-monospace,Menlo,monospace" font-size="12" letter-spacing="1.5" fill="#6b6b66">INSIDE THE BANK</text>

  <rect x="60" y="122" width="300" height="84" rx="14" fill="#141412"/>
  <text x="210" y="158" text-anchor="middle" font-family="Georgia,serif" font-size="20" font-weight="600" fill="#ffffff">Agent SDK harness</text>
  <text x="210" y="182" text-anchor="middle" font-family="-apple-system,'Segoe UI',sans-serif" font-size="12" fill="#c9c4b8">loop · hooks · permissions</text>

  <rect x="384" y="122" width="180" height="84" rx="14" fill="#ffffff" stroke="#1a1a17" stroke-width="1.5"/>
  <text x="474" y="158" text-anchor="middle" font-family="Georgia,serif" font-size="19" font-weight="600" fill="#1a1a17">MCP tools</text>
  <text x="474" y="182" text-anchor="middle" font-family="-apple-system,'Segoe UI',sans-serif" font-size="12" fill="#6b6b66">reads + actions</text>

  <rect x="588" y="122" width="176" height="84" rx="14" fill="#ffffff" stroke="#1a1a17" stroke-width="1.5"/>
  <text x="676" y="158" text-anchor="middle" font-family="Georgia,serif" font-size="19" font-weight="600" fill="#1a1a17">Session state</text>
  <text x="676" y="182" text-anchor="middle" font-family="-apple-system,'Segoe UI',sans-serif" font-size="12" fill="#6b6b66">files we hold</text>

  <rect x="60" y="246" width="184" height="84" rx="14" fill="#ffffff" stroke="#1a1a17" stroke-width="1.5"/>
  <text x="152" y="282" text-anchor="middle" font-family="Georgia,serif" font-size="19" font-weight="600" fill="#1a1a17">Incident index</text>
  <text x="152" y="306" text-anchor="middle" font-family="-apple-system,'Segoe UI',sans-serif" font-size="12" fill="#6b6b66">retrieval, not memory</text>

  <rect x="268" y="246" width="156" height="84" rx="14" fill="#ffffff" stroke="#1a1a17" stroke-width="1.5"/>
  <text x="346" y="282" text-anchor="middle" font-family="Georgia,serif" font-size="19" font-weight="600" fill="#1a1a17">Secrets</text>
  <text x="346" y="306" text-anchor="middle" font-family="-apple-system,'Segoe UI',sans-serif" font-size="12" fill="#6b6b66">never in prompts</text>

  <rect x="448" y="246" width="316" height="84" rx="14" fill="#D97757"/>
  <text x="606" y="282" text-anchor="middle" font-family="Georgia,serif" font-size="20" font-weight="600" fill="#1a1a17">Action gateway</text>
  <text x="606" y="306" text-anchor="middle" font-family="-apple-system,'Segoe UI',sans-serif" font-size="12" fill="#3d2417">the only write path</text>

  <line x1="846" y1="64" x2="846" y2="436" stroke="#b9b4a9" stroke-width="1.2" stroke-dasharray="4 6"/>
  <text x="862" y="350" font-family="ui-monospace,Menlo,monospace" font-size="11" letter-spacing="2" fill="#6b6b66" text-anchor="middle" transform="rotate(-90 862 350)">RETENTION BOUNDARY</text>

  <rect x="920" y="140" width="288" height="112" rx="18" fill="#141412"/>
  <text x="1064" y="190" text-anchor="middle" font-family="Georgia,serif" font-size="24" font-weight="600" fill="#ffffff">Anthropic</text>
  <text x="1064" y="216" text-anchor="middle" font-family="-apple-system,'Segoe UI',sans-serif" font-size="13" fill="#c9c4b8">inference only</text>

  <line x1="790" y1="176" x2="914" y2="176" stroke="#1a1a17" stroke-width="1.5" marker-end="url(#d2k)"/>
  <text x="852" y="164" text-anchor="middle" font-family="ui-monospace,Menlo,monospace" font-size="12" fill="#6b6b66" stroke="#FBFAF7" stroke-width="6" paint-order="stroke">prompts · schemas</text>
  <line x1="914" y1="218" x2="790" y2="218" stroke="#1a1a17" stroke-width="1.5" marker-end="url(#d2k)"/>
  <text x="852" y="242" text-anchor="middle" font-family="ui-monospace,Menlo,monospace" font-size="12" fill="#6b6b66" stroke="#FBFAF7" stroke-width="6" paint-order="stroke">completions</text>

  <line x1="1064" y1="252" x2="1064" y2="314" stroke="#b9b4a9" stroke-width="1.2" stroke-dasharray="3 5"/>
  <rect x="920" y="318" width="288" height="112" rx="18" fill="none" stroke="#b9b4a9" stroke-width="1.5" stroke-dasharray="5 5"/>
  <text x="944" y="350" font-family="ui-monospace,Menlo,monospace" font-size="11" letter-spacing="1.5" fill="#6b6b66">CONDITIONAL</text>
  <text x="944" y="378" font-family="Georgia,serif" font-size="18" font-weight="600" fill="#1a1a17">Managed Agents</text>
  <text x="944" y="400" font-family="-apple-system,'Segoe UI',sans-serif" font-size="12" fill="#6b6b66">+ self&#8209;hosted sandbox · when terms fit</text>

  <text x="620" y="482" text-anchor="middle" font-family="-apple-system,'Segoe UI',sans-serif" font-size="15" fill="#55534d">Only inference crosses the wire. Harness, tools, sessions, secrets, and authority stay inside.</text>
</svg>
<svg class="m" viewBox="0 0 620 600" role="img" aria-label="Mobile view: everything except inference stays inside the bank">
  <defs><marker id="m2k" viewBox="0 0 10 10" refX="8.5" refY="5" markerWidth="8" markerHeight="8" orient="auto-start-reverse"><path d="M0 0L10 5L0 10z" fill="#1a1a17"/></marker></defs>
  <rect x="32" y="14" width="556" height="250" rx="20" fill="#ffffff" stroke="#1a1a17" stroke-width="1.6"/>
  <text x="58" y="50" font-family="ui-monospace,Menlo,monospace" font-size="14" letter-spacing="1.5" fill="#6b6b66">INSIDE THE BANK</text>
  <text x="58" y="92" font-family="-apple-system,'Segoe UI',sans-serif" font-size="17" fill="#26251f">Agent SDK harness &#183; loop, hooks, permissions</text>
  <text x="58" y="126" font-family="-apple-system,'Segoe UI',sans-serif" font-size="17" fill="#26251f">MCP tools &#183; session files we hold</text>
  <text x="58" y="160" font-family="-apple-system,'Segoe UI',sans-serif" font-size="17" fill="#26251f">incident index &#183; secrets, never in prompts</text>
  <text x="58" y="200" font-family="-apple-system,'Segoe UI',sans-serif" font-size="17" font-weight="600" fill="#a14e2f">action gateway: the only write path</text>
  <line x1="270" y1="272" x2="270" y2="328" stroke="#1a1a17" stroke-width="1.6" marker-end="url(#m2k)"/>
  <line x1="350" y1="328" x2="350" y2="272" stroke="#1a1a17" stroke-width="1.6" marker-end="url(#m2k)"/>
  <text x="132" y="306" text-anchor="middle" font-family="ui-monospace,Menlo,monospace" font-size="14" fill="#6b6b66">prompts &#183; schemas</text>
  <text x="478" y="306" text-anchor="middle" font-family="ui-monospace,Menlo,monospace" font-size="14" fill="#6b6b66">completions</text>
  <rect x="32" y="336" width="556" height="88" rx="20" fill="#141412"/>
  <text x="310" y="374" text-anchor="middle" font-family="Georgia,serif" font-size="26" font-weight="600" fill="#ffffff">Anthropic</text>
  <text x="310" y="404" text-anchor="middle" font-family="-apple-system,'Segoe UI',sans-serif" font-size="17" fill="#c9c4b8">inference only</text>
  <rect x="32" y="452" width="556" height="120" rx="18" fill="none" stroke="#b9b4a9" stroke-width="1.6" stroke-dasharray="5 5"/>
  <text x="58" y="486" font-family="ui-monospace,Menlo,monospace" font-size="13" letter-spacing="1.5" fill="#6b6b66">CONDITIONAL</text>
  <text x="58" y="518" font-family="Georgia,serif" font-size="21" font-weight="600" fill="#1a1a17">Managed Agents (beta)</text>
  <text x="58" y="548" font-family="-apple-system,'Segoe UI',sans-serif" font-size="16" fill="#6b6b66">self&#8209;hosted sandbox; when retention terms fit</text>
</svg>
<figcaption>Prompts and completions are the only traffic that crosses the boundary. Session state, tools, secrets, and write authority stay on bank infrastructure. Managed Agents with a self&#8209;hosted sandbox is the next progression of the design.</figcaption>
</figure>

<p>The Agent SDK provides the same loop, tools, and context management that power Claude Code, running in our process, with session state as JSONL files we hold. Skills load from the filesystem, MCP servers attach as configuration, and hooks let us enforce policy in code around every tool call. A simplified version of the investigator setup:</p>

<pre><code>from claude_agent_sdk import query, ClaudeAgentOptions, HookMatcher

async def enforce_read_only(input_data, tool_use_id, context):
    # Writes are only legal through the action gateway, which is not
    # in this session's toolset. Block anything that could mutate
    # state, and log every attempt.
    tool = input_data.get("tool_name", "")
    if tool in {"Write", "Edit", "Bash"}:
        return {"hookSpecificOutput": {
            "hookEventName": "PreToolUse",
            "permissionDecision": "deny",
            "permissionDecisionReason": "writes go through the action gateway",
        }}
    audit(tool_use_id, input_data)
    return {}

options = ClaudeAgentOptions(
    system_prompt=INVESTIGATOR_PROMPT,
    setting_sources=["project"],  # loads .claude/skills: global, domain, service
    mcp_servers={"obs": OBS_SERVER, "incidents": INDEX_SERVER,
                 "sandbox": SANDBOX_SERVER},
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
    handle(message)</code></pre>

<p>Steering needs no extra machinery, because sessions are resumable. The first message carries a <code>session_id</code>, and a follow&#8209;up from the incident thread resumes it with <code>ClaudeAgentOptions(resume=session_id)</code>. The engineer redirects the investigation instead of restarting it, and the session keeps everything it has already read and concluded. It is also why the investigation is one session rather than a fan&#8209;out of subagents: a single accountable hypothesis thread, steerable in one place, with parallelism pushed down into the analysis code. In the thread, that looks like this:</p>

<div class="thread" id="steering-loop-example">
  <div class="msg"><div class="avatar a-bot">SRE</div><div class="tbody"><div class="name">SRE agent <span>14:11</span></div>
  Hypothesis, draft 1: the 14:02 deploy lowered checkout&#8209;svc's memory limit to 2.5Gi while adding a larger in&#8209;process cache. 340 OOMKilled events begin 90 seconds after rollout, heap sits at 94% of the limit, the config diff shows the change, and two prior incidents look similar (INC&#8209;3121, INC&#8209;2984). Full evidence linked.
  <div class="pills"><span class="pill sel">Partially accurate</span><span class="pill">Accurate</span><span class="pill">Incorrect</span><span class="pill">Still investigating</span></div></div></div>
  <div class="msg"><div class="avatar a-eng">MR</div><div class="tbody"><div class="name">on&#8209;call engineer <span>14:13</span></div>The memory story checks out, but error rate moved before the deploy. Check Kafka consumer lag on payments&#8209;consumer first.</div></div>
  <div class="msg"><div class="avatar a-bot">SRE</div><div class="tbody"><div class="name">SRE agent <span>14:15</span></div>Resuming this session. The early onset checks out: the same config change canaried at 13:47, and payments-consumer lag from 13:48 is downstream of checkout errors, not a separate cause. The deploy stays the primary hypothesis; updating the rollback proposal with the wider evidence window.</div></div>
</div>
<p class="threadcap">The steering loop, shown on an illustrative incident.</p>

<p>Sessions that run for hours eventually compact. It is essential that the following survive beyond summarization: the incident envelope, confirmed findings, open hypotheses, and action state live as files in the session workspace, written by the harness from the structured records the agent emits rather than by the agent itself. These are re&#8209;read on demand so that compaction can drop transcript without dropping the case. Traces mark any conclusion reached after a compaction, and reviewers weigh those accordingly. Cross&#8209;incident recall follows the same rule: the platform's memory tool would give the agent persistent files of its own, but recall at a bank has to be shared across responders, audited, and governed on our side of the boundary, so it lives in the incident index as retrieval instead.</p>
<p>Those same files also handle escalation, cheap to detect and rare to pay for. An investigation stalls when the session exhausts its tool&#8209;call and thinking budget without producing a corroborated hypothesis, meaning one supported by the deploy diff, the metrics, and the logs together. The harness detects this mechanically: the structured record it maintains has a dedicated field for that hypothesis, so when the budget runs out, the field is either filled or the session has stalled.</p>
<p>On a stall, the harness escalates to an Opus&#8209;class consult. The consult receives the case file but not the transcript, which at that point is mostly a record of failed attempts. The consult stays advisory. It returns a structured advice record specifying which reads to run next and which hypotheses to prioritize, and that record is appended to the session for the Sonnet&#8209;class investigator to act on when it resumes.</p>
<p>In practice, about one investigation in twenty stalls. Roughly two&#8209;thirds of those reach a corroborated hypothesis after the consult; the rest are handed to the on&#8209;call engineer with the case file attached, which is where they were headed anyway. And because Opus runs only on stalled sessions, the typical investigation never pays for it: the consult's cost sits in the tail of the distribution.</p>
<h2 id="keeping-evidence-out-of-the-context-window">Keeping evidence out of the context window</h2>

<p>An investigation touches a lot of data the model should never hold. Six hours of error logs might be tens of thousands of lines; the useful signal is that 340 of them are OOM kills and the first one landed 90 seconds after a deploy. Context is for reasoning: fetching, filtering, and counting happen in code next to the data, and only conclusions come back. Claude writes a small program that calls the read tools, filters and joins the results, and returns one compact evidence object, and only that object enters the context window. On the Messages API, the managed version of the pattern is programmatic tool calling, enabled by adding <code>allowed_callers</code> to a tool definition alongside the code execution tool:</p>

<pre><code>tools=[
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
]</code></pre>

<p>We do not run our telemetry through the hosted pass, for two reasons. One, MCP‑connector tools cannot be called programmatically, which is why the snippet above defines the reads as plain custom tools; the limit is the connector’s, and the MCP servers the Agent SDK attaches are unaffected. Two, the managed container runs on Anthropic infrastructure and retains execution artifacts for up to 30 days, the same class of constraint that kept sessions in‑house. So the investigator runs the identical pattern behind one bank‑side tool. <code>run_analysis</code> takes the program Claude writes, executes it in a network‑restricted sandbox next to the read APIs, and returns only what it prints; the docs describe this self‑managed sandbox alongside the hosted one, and for this data it is the only one we can use. Claude’s side is unchanged either way:</p>

<pre><code># code Claude writes; executes in the bank-side sandbox
import json

deploys = json.loads(await get_recent_deploys({"service": "checkout", "hours": 6}))
logs = await search_logs({"service": "checkout", "level": "ERROR", "hours": 6})
oom = [line for line in logs.splitlines() if "OOMKilled" in line]

print(json.dumps({
    "deploy_ids": [d["id"] for d in deploys],
    "oom_count": len(oom),
    "first_oom_ts": oom[0].split()[0] if oom else None,
}))</code></pre>

<p>We reserve the pattern for the read‑heavy phase and keep the triage fast path on direct calls, since programmatic calling adds overhead when a turn makes only one or two calls. Anthropic’s published benchmark shows the same split: a token reduction on a large read‑heavy tool agent, a small penalty on low‑call workloads.</p>

<p>Tool Search handles the adjacent problem of a catalog too large to load up front: deferred tools (the <code>defer_loading</code> mechanism) stay out of context until Claude searches for them, so a session opens with the incident envelope, the safety instructions, and a handful of common tools rather than every connector we operate.</p>

<p>The same discipline is the cost model. Prompt caching holds the stable prefix, the system prompt, tool definitions, and Skill metadata, so an hours‑long session pays for its methodology once. Triage runs on Haiku, the investigator on a Sonnet‑class model, and because only evidence objects enter context, token spend scales with findings rather than with log volume. Reasoning is the third axis of the same discipline: triage runs with extended thinking off, which is much of why it is cheap, while the investigator carries a thinking budget sized to the case. Calls, models, and thinking depth are right&#8209;sized together.</p>

<p>Writes go the other way. Rather than letting the model chain five small mutations, each write is a single tool, <code>rollback_deployment</code> for example, that owns its preconditions, dry run, approval routing, idempotency key, and inverse action.</p>

<p>The programmatic tool calling documentation is explicit that <code>allowed_callers</code> guides the model but is not a hard block, and should not be relied on as a security boundary. We take the same position one level up: no tool annotation, prompt, or Skill grants write authority.</p>

<p>The gateway checks every write independently, which is also what keeps the system safe against prompt injection arriving through log lines and tickets. A tool result can inform a hypothesis; it cannot grant permission. The residual risk sits one layer up: injected text cannot approve anything, but it can steer a wrong diagnosis into a plausible proposal for a tired human at 3 a.m. Two habits blunt it. Every claim in a proposal carries its source link, and no mitigation is proposed on the testimony of a single source.</p>

<h2 id="runbooks-as-skills">Runbooks as Skills</h2>

<p>The knowledge that makes an investigation good is owned by service teams, not by us, so it ships as Skills in their repositories rather than as sections of a central prompt. A Skill is a folder with a <code>SKILL.md</code>. The frontmatter is what Claude matches a task against, and the body loads only when it matches:</p>

<pre><code>---
name: service-checkout
description: Investigate checkout-svc incidents. Use when an alert names
  checkout-svc or symptoms include payment failures at checkout.
---

# What this service does
# Failure signatures
# Key dashboards and queries
# Dependencies and ownership
# Safe mitigations
# Escalate or prohibit</code></pre>

<p>Skills compose by specificity. A checkout OOM session loads the global incident discipline, the OOM domain playbook, and the checkout service Skill. The global Skill says to diagnose before proposing a change. The domain Skill explains how to tell a lowered memory limit from a leak. The service Skill knows checkout runs on cost&#8209;constrained nodes, so the obvious fix, raising the limit, is the wrong one there. Until a Skill matches, it costs only its metadata in context, the name and description above, which is what lets the catalog grow to hundreds of runbooks without inflating every session.</p>

<p>A service does not need a Skill to onboard; it starts on the global and domain tiers and gets its own the first time an incident teaches it something. Teams write and ship these without the platform team in the loop. <code>sre skill from-incident INC-4821</code> scaffolds a draft from a closed postmortem, local replay through the Agent SDK runs it against the service's recorded incidents, and CI will not publish a Skill without a named owner and a passing eval set. That division of labor is what lets the catalog scale: we run the tooling, service teams own what is true about their services, and the signal we watch is whether a team's second Skill ships without anyone from our team involved.</p>

<p>Policy Skills are the exception. They describe rules like data&#8209;access boundaries and change freezes so the model can reason about them, but they enforce nothing. Enforcement stays in the hook and the gateway. A Skill can make the agent smarter about policy; it cannot authorize an action.</p>

<h2 id="measuring-whether-a-diagnosis-was-right">Measuring whether a diagnosis was right</h2>

<p>A root&#8209;cause hypothesis does not score itself. A fluent explanation can be wrong, and at the moment the agent posts it, the true cause is often unknown to the humans too. So each diagnosis gets two labels at two different times.</p>

<figure>
<svg class="d" viewBox="0 0 1240 470" role="img" aria-label="The two-truth loop: RCA in Slack, provisional label, verified postmortem truth, reconciliation into eval cases and Skill updates, feeding the next investigation">
  <defs>
    <marker id="d3k" viewBox="0 0 10 10" refX="8.5" refY="5" markerWidth="7" markerHeight="7" orient="auto-start-reverse"><path d="M0 0L10 5L0 10z" fill="#1a1a17"/></marker>
    <marker id="d3c" viewBox="0 0 10 10" refX="8.5" refY="5" markerWidth="7" markerHeight="7" orient="auto-start-reverse"><path d="M0 0L10 5L0 10z" fill="#D97757"/></marker>
  </defs>

  <rect x="70" y="92" width="250" height="104" rx="18" fill="#ffffff" stroke="#1a1a17" stroke-width="1.5"/>
  <text x="195" y="136" text-anchor="middle" font-family="Georgia,serif" font-size="21" font-weight="600" fill="#1a1a17">RCA in Slack</text>
  <text x="195" y="164" text-anchor="middle" font-family="-apple-system,'Segoe UI',sans-serif" font-size="12.5" fill="#6b6b66">hypothesis + linked evidence</text>

  <line x1="322" y1="144" x2="370" y2="144" stroke="#1a1a17" stroke-width="1.5" marker-end="url(#d3k)"/>
  <text x="346" y="130" text-anchor="middle" font-family="ui-monospace,Menlo,monospace" font-size="12" fill="#6b6b66">one click</text>

  <rect x="376" y="92" width="290" height="104" rx="18" fill="#ffffff" stroke="#1a1a17" stroke-width="1.5"/>
  <text x="521" y="136" text-anchor="middle" font-family="Georgia,serif" font-size="21" font-weight="600" fill="#1a1a17">Provisional label</text>
  <text x="521" y="164" text-anchor="middle" font-family="-apple-system,'Segoe UI',sans-serif" font-size="12.5" fill="#6b6b66">accurate · partial · incorrect · unknown</text>

  <line x1="668" y1="144" x2="716" y2="144" stroke="#1a1a17" stroke-width="1.5" marker-end="url(#d3k)"/>
  <text x="692" y="130" text-anchor="middle" font-family="ui-monospace,Menlo,monospace" font-size="12" fill="#6b6b66">closes</text>

  <rect x="722" y="92" width="270" height="104" rx="18" fill="#141412"/>
  <text x="857" y="136" text-anchor="middle" font-family="Georgia,serif" font-size="21" font-weight="600" fill="#ffffff">Verified truth</text>
  <text x="857" y="164" text-anchor="middle" font-family="-apple-system,'Segoe UI',sans-serif" font-size="12.5" fill="#c9c4b8">postmortem cause + fix</text>

  <line x1="857" y1="196" x2="857" y2="262" stroke="#1a1a17" stroke-width="1.5" marker-end="url(#d3k)"/>

  <rect x="560" y="268" width="330" height="104" rx="18" fill="#D97757"/>
  <text x="725" y="312" text-anchor="middle" font-family="Georgia,serif" font-size="21" font-weight="600" fill="#1a1a17">Reconcile</text>
  <text x="725" y="340" text-anchor="middle" font-family="-apple-system,'Segoe UI',sans-serif" font-size="12.5" fill="#3d2417">new eval case + drafted Skill update</text>

  <polyline points="556,320 195,320 195,204" fill="none" stroke="#D97757" stroke-width="1.5" marker-end="url(#d3c)"/>
  <text x="372" y="306" text-anchor="middle" font-family="ui-monospace,Menlo,monospace" font-size="12" fill="#a14e2f">the next investigation starts smarter</text>

  <text x="620" y="428" text-anchor="middle" font-family="-apple-system,'Segoe UI',sans-serif" font-size="15" fill="#55534d">No accuracy headline without a verified denominator.</text>
</svg>
<svg class="m" viewBox="0 0 620 530" role="img" aria-label="Mobile view: the two-label loop from Slack verdict to verified postmortem to eval case">
  <defs>
    <marker id="m3k" viewBox="0 0 10 10" refX="8.5" refY="5" markerWidth="8" markerHeight="8" orient="auto-start-reverse"><path d="M0 0L10 5L0 10z" fill="#1a1a17"/></marker>
    <marker id="m3c" viewBox="0 0 10 10" refX="8.5" refY="5" markerWidth="8" markerHeight="8" orient="auto-start-reverse"><path d="M0 0L10 5L0 10z" fill="#a14e2f"/></marker>
  </defs>
  <rect x="40" y="14" width="540" height="82" rx="18" fill="#ffffff" stroke="#1a1a17" stroke-width="1.6"/>
  <text x="310" y="48" text-anchor="middle" font-family="Georgia,serif" font-size="24" font-weight="600" fill="#1a1a17">RCA in Slack</text>
  <text x="310" y="76" text-anchor="middle" font-family="-apple-system,'Segoe UI',sans-serif" font-size="17" fill="#6b6b66">hypothesis + linked evidence</text>
  <line x1="310" y1="100" x2="310" y2="128" stroke="#1a1a17" stroke-width="1.6" marker-end="url(#m3k)"/>
  <rect x="40" y="134" width="540" height="82" rx="18" fill="#ffffff" stroke="#1a1a17" stroke-width="1.6"/>
  <text x="310" y="168" text-anchor="middle" font-family="Georgia,serif" font-size="24" font-weight="600" fill="#1a1a17">Provisional label</text>
  <text x="310" y="196" text-anchor="middle" font-family="-apple-system,'Segoe UI',sans-serif" font-size="17" fill="#6b6b66">accurate &#183; partial &#183; incorrect &#183; unknown</text>
  <line x1="310" y1="220" x2="310" y2="248" stroke="#1a1a17" stroke-width="1.6" marker-end="url(#m3k)"/>
  <rect x="40" y="254" width="540" height="82" rx="18" fill="#141412"/>
  <text x="310" y="288" text-anchor="middle" font-family="Georgia,serif" font-size="24" font-weight="600" fill="#ffffff">Verified truth</text>
  <text x="310" y="316" text-anchor="middle" font-family="-apple-system,'Segoe UI',sans-serif" font-size="17" fill="#c9c4b8">postmortem cause + fix</text>
  <line x1="310" y1="340" x2="310" y2="368" stroke="#1a1a17" stroke-width="1.6" marker-end="url(#m3k)"/>
  <rect x="40" y="374" width="540" height="82" rx="18" fill="#D97757"/>
  <text x="310" y="408" text-anchor="middle" font-family="Georgia,serif" font-size="24" font-weight="600" fill="#1a1a17">Reconcile</text>
  <text x="310" y="436" text-anchor="middle" font-family="-apple-system,'Segoe UI',sans-serif" font-size="17" fill="#3d2417">new eval case + drafted Skill update</text>
  <polyline points="38,415 16,415 16,55 32,55" fill="none" stroke="#a14e2f" stroke-width="1.6" marker-end="url(#m3c)"/>
  <text x="310" y="500" text-anchor="middle" font-family="-apple-system,'Segoe UI',sans-serif" font-size="16" fill="#55534d">Each reconciled pair makes the next investigation start smarter.</text>
</svg>
<figcaption>Engineers label the diagnosis in the incident thread while it is live. After the review closes, the verified cause is reconciled against that label, and the pair becomes an eval case plus a drafted Skill update for the owning team.</figcaption>
</figure>

<p>The provisional label is a one&#8209;click verdict in the incident thread, accurate, partially accurate, incorrect, or still investigating, with an optional correction. The verified label comes later, reconciled against the post&#8209;incident review's confirmed cause and fix. Every reconciled pair becomes an eval case, and the replay path depends on the artifact. Bounded calls, triage envelopes and judge scoring, rerun through Message Batches at half the interactive price. Full investigations replay through the Agent SDK against recorded tool results: every MCP read a live session makes is captured as a fixture, so a closed incident becomes a reproducible case a Skill change can be tested against before it ships. When a new trajectory requests a read the fixtures do not hold, the case fails loudly rather than falling through to live systems. The capture layer is the quiet workhorse here; recording reads during live incidents is what turns history into tests. A replay is scored in layers, cheap checks first: the record validates against its schema, every read it claims exists in the fixtures, and no denied tool was attempted. The judge runs last because it is the expensive opinion.</p>

<p>Two reporting rules keep the numbers honest. We never merge unknown into incorrect, and we never compute accuracy over incidents that lack a verified final cause. On business impact, time&#8209;to&#8209;hypothesis is the metric the agent directly moves. MTTR stays the north star, but most of an incident's wall clock is waiting, coordination, and recovery rather than diagnosis, so we claim an MTTR effect only from a controlled comparison, not from a before&#8209;and&#8209;after.</p>

<h2 id="checking-the-work-before-it-posts">Checking the work before it posts</h2>

<p>The corroboration rule, that the deploy diff, the metrics, and the logs have to agree before anything posts, started as a sentence in the system prompt and a habit of whoever read the thread. The rule exists because the cheapest way for an investigator to be wrong is to fixate on the first plausible cause. It is now a mechanism. Before a diagnosis posts to Slack or a proposal reaches the gateway, a grading pass, a single Haiku&#8209;class structured call, re&#8209;reads the record, follows each claim's source link, and checks that the excerpt supports the claim. A failure bounces the record back into the session with the failing claim named; the bounce is just a resume, the same mechanism a human steer uses, and it is capped at two, after which the diagnosis posts flagged instead of clean rather than being suppressed, because a wrong but visible hypothesis in the thread is recoverable and a silently withheld one is not.</p>

<p>A bounce is itself a typed record:</p>

<pre><code>{
  "record": "dx_4821",
  "verdict": "bounce",
  "failing_claim": "onset precedes the 14:02 deploy",
  "check": "cited excerpt starts at 14:02; the claim needs the 13:47 canary window",
  "bounce_count": 1,
  "disposition": "resume, failing claim named"
}</code></pre>

<p>Two properties keep the grader inside the trust model. It can block and bounce; it cannot approve, so it adds no authority anywhere. And its verdicts land in the eval store like any other artifact and reconcile against verified causes like any other label, because a checker with an unmeasured error rate is one more opinion. In a domain where being wrong has consequences, the verification logic between the model and execution is the part of the harness most worth owning; the grader is that logic promoted from a rule to a job, at about two cents a pass.</p>

<h2 id="gating-actions">Gating actions</h2>

<p>Reads are autonomous where data policy permits. Writes are transactions, and each action class has its own promotion requirements:</p>

<figure>
<svg class="d" viewBox="0 0 1240 560" role="img" aria-label="Autonomy ladder from read-only through R2, with R3 capabilities shown struck through: absent as tools by design">
  <defs>
    <marker id="d4k" viewBox="0 0 10 10" refX="8.5" refY="5" markerWidth="7" markerHeight="7" orient="auto-start-reverse"><path d="M0 0L10 5L0 10z" fill="#1a1a17"/></marker>
    <marker id="d4g" viewBox="0 0 10 10" refX="8.5" refY="5" markerWidth="7" markerHeight="7" orient="auto-start-reverse"><path d="M0 0L10 5L0 10z" fill="#b9b4a9"/></marker>
  </defs>

  <text x="60" y="48" font-family="ui-monospace,Menlo,monospace" font-size="12" letter-spacing="1.5" fill="#6b6b66">AUTONOMY BY ACTION CLASS</text>
  <line x1="60" y1="64" x2="1174" y2="64" stroke="#b9b4a9" stroke-width="1.2" marker-end="url(#d4g)"/>

  <rect x="60" y="92" width="240" height="138" rx="16" fill="#ffffff" stroke="#1a1a17" stroke-width="1.5"/>
  <text x="180" y="136" text-anchor="middle" font-family="Georgia,serif" font-size="22" font-weight="600" fill="#1a1a17">Read&#8209;only</text>
  <text x="180" y="160" text-anchor="middle" font-family="-apple-system,'Segoe UI',sans-serif" font-size="12.5" fill="#6b6b66">diagnosis · evidence</text>
  <line x1="88" y1="178" x2="272" y2="178" stroke="#e4e0d6"/>
  <text x="180" y="204" text-anchor="middle" font-family="ui-monospace,Menlo,monospace" font-size="11" letter-spacing="1" fill="#6b6b66">autonomous</text>

  <line x1="302" y1="160" x2="330" y2="160" stroke="#1a1a17" stroke-width="1.5" marker-end="url(#d4k)"/>

  <rect x="336" y="92" width="240" height="138" rx="16" fill="#ffffff" stroke="#1a1a17" stroke-width="1.5"/>
  <text x="456" y="136" text-anchor="middle" font-family="Georgia,serif" font-size="22" font-weight="600" fill="#1a1a17">R0 · Paper</text>
  <text x="456" y="160" text-anchor="middle" font-family="-apple-system,'Segoe UI',sans-serif" font-size="12.5" fill="#6b6b66">summary · ticket · draft PR</text>
  <line x1="364" y1="178" x2="548" y2="178" stroke="#e4e0d6"/>
  <text x="456" y="204" text-anchor="middle" font-family="ui-monospace,Menlo,monospace" font-size="11" letter-spacing="1" fill="#6b6b66">auto, audited</text>

  <line x1="578" y1="160" x2="606" y2="160" stroke="#1a1a17" stroke-width="1.5" marker-end="url(#d4k)"/>

  <rect x="612" y="92" width="252" height="138" rx="16" fill="#ffffff" stroke="#1a1a17" stroke-width="1.5"/>
  <text x="738" y="136" text-anchor="middle" font-family="Georgia,serif" font-size="22" font-weight="600" fill="#1a1a17">R1 · Reversible</text>
  <text x="738" y="160" text-anchor="middle" font-family="-apple-system,'Segoe UI',sans-serif" font-size="12.5" fill="#6b6b66">restart · scale · merge</text>
  <line x1="640" y1="178" x2="836" y2="178" stroke="#e4e0d6"/>
  <text x="738" y="204" text-anchor="middle" font-family="ui-monospace,Menlo,monospace" font-size="11" letter-spacing="1" fill="#6b6b66">approval · dry run · inverse</text>

  <line x1="866" y1="160" x2="894" y2="160" stroke="#1a1a17" stroke-width="1.5" marker-end="url(#d4k)"/>

  <rect x="900" y="92" width="280" height="138" rx="16" fill="#D97757"/>
  <text x="1040" y="136" text-anchor="middle" font-family="Georgia,serif" font-size="22" font-weight="600" fill="#1a1a17">R2 · Broad</text>
  <text x="1040" y="160" text-anchor="middle" font-family="-apple-system,'Segoe UI',sans-serif" font-size="12.5" fill="#3d2417">rollback · traffic · config</text>
  <line x1="928" y1="178" x2="1152" y2="178" stroke="#c06342"/>
  <text x="1040" y="204" text-anchor="middle" font-family="ui-monospace,Menlo,monospace" font-size="11" letter-spacing="1" fill="#3d2417">senior approval · blast radius</text>

  <text x="60" y="316" font-family="ui-monospace,Menlo,monospace" font-size="12" letter-spacing="1.5" fill="#6b6b66">R3 · ABSENT AS TOOLS, BY DESIGN</text>

  <rect x="60" y="336" width="300" height="64" rx="12" fill="none" stroke="#b9b4a9" stroke-width="1.5" stroke-dasharray="5 5"/>
  <text x="210" y="374" text-anchor="middle" font-family="Georgia,serif" font-size="17" fill="#6b6b66">ledger mutation</text>
  <line x1="76" y1="388" x2="344" y2="348" stroke="#b9b4a9" stroke-width="1.5"/>

  <rect x="400" y="336" width="320" height="64" rx="12" fill="none" stroke="#b9b4a9" stroke-width="1.5" stroke-dasharray="5 5"/>
  <text x="560" y="374" text-anchor="middle" font-family="Georgia,serif" font-size="17" fill="#6b6b66">payment execution</text>
  <line x1="416" y1="388" x2="704" y2="348" stroke="#b9b4a9" stroke-width="1.5"/>

  <rect x="760" y="336" width="420" height="64" rx="12" fill="none" stroke="#b9b4a9" stroke-width="1.5" stroke-dasharray="5 5"/>
  <text x="970" y="374" text-anchor="middle" font-family="Georgia,serif" font-size="17" fill="#6b6b66">irreversible customer&#8209;data change</text>
  <line x1="776" y1="388" x2="1164" y2="348" stroke="#b9b4a9" stroke-width="1.5"/>

  <text x="60" y="442" font-family="-apple-system,'Segoe UI',sans-serif" font-size="13.5" fill="#6b6b66">The list does not shrink as models improve; the agent recommends, and a human executes through existing systems.</text>

  <text x="620" y="510" text-anchor="middle" font-family="-apple-system,'Segoe UI',sans-serif" font-size="15" fill="#55534d">A confidence score routes within a class, never across one.</text>
</svg>
<svg class="m" viewBox="0 0 620 770" role="img" aria-label="Mobile view: autonomy ladder from read-only to R2, with R3 not built">
  <defs><marker id="m4k" viewBox="0 0 10 10" refX="8.5" refY="5" markerWidth="8" markerHeight="8" orient="auto-start-reverse"><path d="M0 0L10 5L0 10z" fill="#1a1a17"/></marker></defs>
  <rect x="40" y="14" width="540" height="96" rx="18" fill="#ffffff" stroke="#1a1a17" stroke-width="1.6"/>
  <text x="310" y="54" text-anchor="middle" font-family="Georgia,serif" font-size="24" font-weight="600" fill="#1a1a17">Read&#8209;only</text>
  <text x="310" y="86" text-anchor="middle" font-family="-apple-system,'Segoe UI',sans-serif" font-size="16" fill="#6b6b66">diagnosis &#183; evidence &#183; autonomous</text>
  <line x1="310" y1="114" x2="310" y2="144" stroke="#1a1a17" stroke-width="1.6" marker-end="url(#m4k)"/>
  <rect x="40" y="148" width="540" height="96" rx="18" fill="#ffffff" stroke="#1a1a17" stroke-width="1.6"/>
  <text x="310" y="188" text-anchor="middle" font-family="Georgia,serif" font-size="24" font-weight="600" fill="#1a1a17">R0 &#183; Paper</text>
  <text x="310" y="220" text-anchor="middle" font-family="-apple-system,'Segoe UI',sans-serif" font-size="16" fill="#6b6b66">summary &#183; ticket &#183; draft PR &#183; auto, audited</text>
  <line x1="310" y1="248" x2="310" y2="278" stroke="#1a1a17" stroke-width="1.6" marker-end="url(#m4k)"/>
  <rect x="40" y="282" width="540" height="96" rx="18" fill="#ffffff" stroke="#1a1a17" stroke-width="1.6"/>
  <text x="310" y="322" text-anchor="middle" font-family="Georgia,serif" font-size="24" font-weight="600" fill="#1a1a17">R1 &#183; Reversible</text>
  <text x="310" y="354" text-anchor="middle" font-family="-apple-system,'Segoe UI',sans-serif" font-size="15" fill="#6b6b66">restart &#183; scale &#183; merge &#183; approval + dry run + inverse</text>
  <line x1="310" y1="382" x2="310" y2="412" stroke="#1a1a17" stroke-width="1.6" marker-end="url(#m4k)"/>
  <rect x="40" y="416" width="540" height="96" rx="18" fill="#D97757"/>
  <text x="310" y="456" text-anchor="middle" font-family="Georgia,serif" font-size="24" font-weight="600" fill="#1a1a17">R2 &#183; Broad</text>
  <text x="310" y="488" text-anchor="middle" font-family="-apple-system,'Segoe UI',sans-serif" font-size="15" fill="#3d2417">rollback &#183; traffic &#183; config &#183; senior approval, blast radius</text>
  <text x="40" y="566" font-family="ui-monospace,Menlo,monospace" font-size="14" letter-spacing="1.5" fill="#6b6b66">R3 &#183; NOT BUILT, BY DESIGN</text>
  <rect x="40" y="582" width="540" height="132" rx="14" fill="none" stroke="#b9b4a9" stroke-width="1.6" stroke-dasharray="5 5"/>
  <text x="310" y="620" text-anchor="middle" font-family="Georgia,serif" font-size="19" fill="#6b6b66">ledger mutation</text>
  <text x="310" y="652" text-anchor="middle" font-family="Georgia,serif" font-size="19" fill="#6b6b66">payment execution</text>
  <text x="310" y="684" text-anchor="middle" font-family="Georgia,serif" font-size="19" fill="#6b6b66">irreversible customer&#8209;data change</text>
  <line x1="56" y1="702" x2="564" y2="594" stroke="#b9b4a9" stroke-width="1.6"/>
  <text x="310" y="748" text-anchor="middle" font-family="-apple-system,'Segoe UI',sans-serif" font-size="15" fill="#6b6b66">The agent recommends; a human executes through existing systems.</text>
</svg>
<figcaption>Autonomy widens down the ladder only when a class's evidence supports it. R3 sits apart because its tools were never built.</figcaption>
</figure>

<p>R0 covers paper actions, posting a summary, updating a ticket, opening a draft PR, which run automatically with an audit trail because they do not touch production state. R1 covers bounded reversible writes behind approval, a dry run, and a tested inverse. R2 covers wider changes like rollbacks and traffic shifts, which add senior approval and an explicit blast&#8209;radius report. R3, anything irreversible or money&#8209;affecting, is handled by not building the tool. The agent can recommend a ledger correction; a human executes it through existing systems. Promotion between tiers happens per action class and requires evidence: enough verified cases, eval thresholds met, shadow&#8209;mode performance, and low override rates. A high confidence score routes a case within its tier and never moves it across one.</p>

<p>The proposal that reaches the gateway is itself a structured output, and strict tool use holds the write tools' inputs to their schemas the same way, so the record is typed end to end. The gateway takes typed records and nothing else, which is what keeps approval, replay, and audit uniform across action classes:</p>

<pre><code>{
  "proposal_id": "prop_01j9k2",
  "incident": "INC-4821",
  "action": "rollback_deployment",
  "target": {"service": "checkout-svc",
             "from": "2026.07.14-3", "to": "2026.07.14-2"},
  "risk_tier": "R2",
  "preconditions": {"deploy_correlated": true,
                    "change_freeze": false,
                    "inverse_tested": true},
  "dry_run": {"status": "passed",
              "blast_radius": "1 service, 12 pods, no dependent config"},
  "idempotency_key": "inc-4821-rollback-1",
  "approval": {"route": "senior-oncall", "state": "pending"},
  "expires_at": "2026-07-19T14:40:00Z"
}</code></pre>
<p>That record was generated in shadow, because R2 still runs there: every R2 proposal is evaluated end to end, dry run included, while execution is withheld. R1 is past that stage; restart, scale, and merge execute in production behind approval, each with its dry run and tested inverse.</p>

<h2 id="diagnosis-as-a-capability">Diagnosis as a capability</h2>

<p>The investigator's surface was already a function: a typed envelope in, a typed and sourced diagnosis out, nothing but reads in between. In the spring we wrapped that function in an MCP server, which turns diagnosis into a capability other agents call rather than a product only alerts can start. The deploy pipeline's agent asks whether its rollout caused the errors it is watching before deciding to proceed; the capacity planner asks for six hours of history on a service before recommending a scale&#8209;down; the next internal assistant gets root&#8209;cause analysis without rebuilding it.</p>

<p>Nothing in the trust model moves. The caller receives a hypothesis and its evidence, never authority; a proposal still exits only through the gateway; and agent&#8209;called sessions are admitted below alert&#8209;triggered ones, so a chatty caller cannot starve an incident. One thing does move, and it is the measurement contract: an investigation was defined as a session triggered by a real alert, so agent&#8209;called sessions are a third category next to live and shadow, reported separately, and a colleague agent's curiosity never inflates the incident numbers.</p>

<h2 id="when-the-assistant-is-down">When the assistant is down</h2>

<p>An AI SRE has to plan for its own absence, and the correlation is uncomfortable: the assistant is most likely to be impaired during the largest incidents, when regional trouble degrades the same networks and providers everything else depends on. So it is additive by construction. The incident process ran for years without it and still can. If the envelope call misses its timeout budget after one retry, the alert routes exactly as it did before. If a session dies mid‑investigation, the thread says so and the on‑call keeps working. A circuit breaker on the API path drops the whole system to the human‑only flow and posts one line saying it did. The fast path and the investigator fail independently, and nothing in paging, severity, or escalation blocks on either. The residual exposure is capacity at the provider's end, and part of that is bought rather than engineered: the system runs on committed Priority Tier capacity, which targets 99.5 percent uptime with prioritized compute and falls back to the standard tier past the commitment.</p>

<p>Alert storms get the same treatment. A regional event can fire hundreds of alerts in minutes, and starting hundreds of investigations would be an incident of its own. Envelopes deduplicate on service and error signature into one investigation with a fan‑in list, concurrent sessions are capped and admitted by severity, and separate workspaces with independent rate limits keep triage from starving behind investigation traffic.</p>

<h2 id="keeping-it-current">Keeping it current</h2>

<p>The failure mode we planned for first is stale knowledge, and the mechanism against it is to attach freshness to work that already happens. When a post&#8209;incident review closes, the incident index updates, an eval case is created, and a single Messages API call drafts the corresponding Skill update with links back to the postmortem. The draft goes to the owning team as a pull request and never publishes on its own, because postmortems can be incomplete or wrong too. Skill changes trigger targeted evals in CI through change&#8209;impact mapping, so a checkout change reruns the checkout and OOM cases rather than the entire corpus, and the full suite runs on a schedule and on higher&#8209;risk changes.</p>

<p>Model upgrades get the same treatment as any other risky change. We rerun the full eval suite, compare tool trajectories and costs, and shadow before promoting, because a newer model can be better on average while quietly changing a behavior a workflow depended on. Treating a model swap as a migration rather than a drop&#8209;in matters because the failure is quiet: a headline metric can hold while the tool trajectory shifts underneath it, so the diff we review is trajectories and costs, not accuracy alone.</p>

<p>The harness is tuned to one model family at a time rather than routed across several, which is why a swap is a migration and not a config change, and the migration diff asks a second question now: not just what changed but what can be removed. Early harness text was scaffolding, walls that made a weaker model walk straight, and a newer model consumes that scaffolding, so the candidates are the steering parts: prompt passages that correct a previous model's habits, exemplars demonstrating a format structured outputs already guarantees. In the last migration the eval ran twice, once with the harness as is and once without three such candidates, including the sentence telling the model not to write, and the smaller harness matched, so it promoted. Deleting that sentence was safe precisely because the sentence was scaffolding and the hook is authority: the hook denies writes either way, and the denial counter, flat through the change, priced the deletion. Authority is never a candidate. A harness that only ever grows is accumulating debt against models that no longer need it.</p>

<h2 id="measurement-contract">The measurement contract</h2>

<p>The numbers this system produces will publish later than the design did, which creates a temptation this section exists to remove: the temptation to shape the figures to the story once they arrive. So the contract comes first. Before any inventory publishes, here is what the words will mean. An investigation is a session triggered by a real alert; replay executions in CI are never counted, however many times the fixtures rerun. Live means the diagnosis posted into an incident thread where an engineer could label or redirect it; shadow means it ran without posting. A session started by another agent through the diagnosis surface is agent&#8209;called; it is reported on its own and kept out of the live and shadow counts. A steer is a substantive human redirect, not a bare resume and not a label click, both of which also resume the session. A grader bounce resumes the session too, machine&#8209;initiated, and counts as neither a steer nor a label. The eval corpus is reported as the curated, deduplicated set a change is actually tested against, with raw fixture capture described separately, so a corpus count never masquerades as a capture rate. Reporting windows are trailing 90 days, with cumulative totals only where a store genuinely accumulates.</p>

<p>And here is what any published figures must satisfy, checkable by a reader. Reconciled pairs are fewer than labeled diagnoses, which are fewer than live investigations. Dry runs equal proposals. Executions never exceed proposals, flagged posts never exceed grader bounces, and R2 executions are zero until R2 is promoted, which has not happened and has no date.</p>

<p>The label rate publishes; the label distribution does not, because the distribution reconstructs the accuracy figure we refuse to compute over an unverified denominator. The reconciliation crosstab publishes as a direction and a design response, never as percentages, for the same reason. Every figure ships rounded, dated, with its definition attached and its query retained. A number that cannot meet these terms is not published late; it is not published.</p>

<h2 id="where-it-stands">Where it stands</h2>

<p>As of July 2026, the rollout is deliberately sequenced: the v2 rolled out in phases behind the beta, earliest pieces first. The fast path and the investigator came up before any write capability. The action gateway is live for R0 paper actions and, having cleared its shadow period, for R1 reversible writes behind approval; R2 remains in shadow with no promotion date, deliberately. The eval library grows with every closed review.</p>

<p>The boring inventory, under the contract above. Over the trailing 90 days the agent ran roughly 96,000 investigations on real alerts, about 71,000 live in incident threads and 25,000 in shadow, across 240 of 264 tier&#8209;1 services, with about 55 tier&#8209;2 services in early rollout; the cumulative total since rollout began is around one million. About 12,000 live sessions, one in six, were steered mid&#8209;investigation. About 4,800 investigations stalled into an Opus consult, and about 3,200 of those reached a corroborated hypothesis on the resume. Since the diagnosis surface opened in the spring, other agents have started roughly 1,900 agent&#8209;called sessions, which the contract reports outside the investigation counts here.</p>

<p>The registry holds about 760 Skills, 6 global, 34 domain, and roughly 720 service, with 86 percent of the service Skills carrying no platform&#8209;team commits, and about 180 teams have shipped a second Skill with no platform&#8209;team commits or reviews, the signal we actually watch.</p>

<p>The grader bounced roughly 6,900 diagnoses at least once before they posted, and about 900 posted flagged after a second failed pass. 64 percent of live diagnoses received a one&#8209;click label. The eval store holds about 9,400 curated fixture&#8209;backed incidents and 5,200 reconciled pairs, both cumulative, with raw capture running far larger and continuous. Over that verified set, the diagnosis matched the postmortem's confirmed cause 96 percent of the time.</p>

<p>On the action side, about 168,000 R0 paper actions executed over the window, mostly summaries and ticket updates plus around 12,000 draft pull requests. Roughly 4,900 R1 proposals were generated, every one dry&#8209;run, and about 3,400 executed in production behind approval; the rest were rejected or expired at the gate, which is the gate doing its job. About 1,300 R2 proposals ran end to end in shadow, dry runs matching proposals one for one, with zero executions. The read&#8209;only hook denied and logged about 2,600 write attempts, most of them the model reaching for scratch files or a shell mid&#8209;investigation, and about 180 of them attempts to apply a fix directly rather than propose it.</p>

<p>The median investigation costs about 70 cents, the ninetieth percentile about $3.20 for the long ones, and triage stays under two cents on Haiku. The circuit breaker has dropped the system to the human&#8209;only flow 14 times since rollout, twice in the window, and about 2,300 alert storms collapsed into single investigations, the largest folding roughly 4,800 alerts into one with a fan&#8209;in list.</p>

<p>Two numbers we do not report: an MTTR improvement, because we do not yet have a controlled comparison, and any accuracy figure over incidents without a verified cause.</p>

<h2 id="where-it-goes-next">Where it goes next</h2>

<p>Step back from the components and the system is a set of jobs. One classifies, one investigates, one checks the work before it posts, one advises a stalled session, one scores replays, and one writes what an incident taught back into the Skill catalog through pull requests. The jobs share a shape: a narrow responsibility, a typed record as output, and no authority beyond it. That is the conceptual direction of this design, an agent system that improves by splitting work into jobs it can measure separately, and it is why diagnosis itself became a job other agents can call. One job is missing. Nothing here ever runs the same investigation twice and picks the better answer.</p>

<p>Once an agent works, teams believe they hold two levers, a bigger model or a longer run. A third exists in principle: run N attempts and select among them. The claim that it returns real accuracy goes untested in production systems, because productionizing the selection is genuinely hard, so almost nobody has a number for it on their own task. That last clause is the part this system can do something about, because we do not have to productionize anything to get the number. The eval store holds 5,200 reconciled incidents whose fixtures answer every read a replay makes and whose true cause is verified. Replaying a stratified sample three ways through the Agent SDK, same envelope, same fixtures, independent sessions, and judge&#8209;scoring each attempt's final hypothesis against the verified cause says whether best&#8209;of&#8209;3 beats a single run on incident diagnosis, before a production token is spent. Three runs across a 300&#8209;incident sample is roughly six hundred dollars of tokens at the median investigation cost. The attempts never see the verified cause; only the judge does, and it scores blind to which attempt produced which hypothesis.</p>

<figure>
<svg class="d" viewBox="0 0 1240 560" role="img" aria-label="Best-of-N experiment: the eval store's envelope and fixtures feed three independent investigator replays; a judge scores each final hypothesis against the verified cause, blind to attempt identity, and a pre-registered rule decides between adoption for sev1 and falsification">
  <defs>
    <marker id="d5k" viewBox="0 0 10 10" refX="8.5" refY="5" markerWidth="7" markerHeight="7" orient="auto-start-reverse"><path d="M0 0L10 5L0 10z" fill="#1a1a17"/></marker>
  </defs>

  <rect x="24" y="100" width="252" height="336" rx="16" fill="#F1EAE0"/>
  <text x="48" y="136" font-family="ui-monospace,Menlo,monospace" font-size="12" font-weight="700" letter-spacing="1.5" fill="#1a1a17">EVAL STORE</text>
  <text x="48" y="158" font-family="-apple-system,'Segoe UI',sans-serif" font-size="12" fill="#6b6b66">5,200 reconciled incidents</text>

  <rect x="48" y="186" width="204" height="64" rx="10" fill="#ffffff" stroke="#ddd6c8"/>
  <text x="150" y="213" text-anchor="middle" font-family="Georgia,serif" font-size="16" fill="#1a1a17">Envelope + fixtures</text>
  <text x="150" y="233" text-anchor="middle" font-family="-apple-system,'Segoe UI',sans-serif" font-size="11.5" fill="#6b6b66">every read, recorded</text>

  <rect x="48" y="346" width="204" height="64" rx="10" fill="#ffffff" stroke="#ddd6c8"/>
  <text x="150" y="373" text-anchor="middle" font-family="Georgia,serif" font-size="16" fill="#1a1a17">Verified cause</text>
  <text x="150" y="393" text-anchor="middle" font-family="-apple-system,'Segoe UI',sans-serif" font-size="11.5" fill="#6b6b66">from the closed review</text>

  <line x1="256" y1="218" x2="368" y2="142" stroke="#1a1a17" stroke-width="1.5" marker-end="url(#d5k)"/>
  <line x1="256" y1="218" x2="368" y2="262" stroke="#1a1a17" stroke-width="1.5" marker-end="url(#d5k)"/>
  <line x1="256" y1="218" x2="368" y2="382" stroke="#1a1a17" stroke-width="1.5" marker-end="url(#d5k)"/>
  <text x="296" y="146" text-anchor="middle" font-family="ui-monospace,Menlo,monospace" font-size="12" fill="#6b6b66">same inputs, &#215;3</text>

  <rect x="372" y="104" width="228" height="76" rx="14" fill="#141412"/>
  <text x="486" y="137" text-anchor="middle" font-family="Georgia,serif" font-size="19" font-weight="600" fill="#ffffff">Attempt 1</text>
  <text x="486" y="160" text-anchor="middle" font-family="-apple-system,'Segoe UI',sans-serif" font-size="12" fill="#c9c4b8">investigator replay</text>

  <rect x="372" y="224" width="228" height="76" rx="14" fill="#141412"/>
  <text x="486" y="257" text-anchor="middle" font-family="Georgia,serif" font-size="19" font-weight="600" fill="#ffffff">Attempt 2</text>
  <text x="486" y="280" text-anchor="middle" font-family="-apple-system,'Segoe UI',sans-serif" font-size="12" fill="#c9c4b8">investigator replay</text>

  <rect x="372" y="344" width="228" height="76" rx="14" fill="#141412"/>
  <text x="486" y="377" text-anchor="middle" font-family="Georgia,serif" font-size="19" font-weight="600" fill="#ffffff">Attempt 3</text>
  <text x="486" y="400" text-anchor="middle" font-family="-apple-system,'Segoe UI',sans-serif" font-size="12" fill="#c9c4b8">investigator replay</text>

  <line x1="604" y1="142" x2="684" y2="238" stroke="#1a1a17" stroke-width="1.5" marker-end="url(#d5k)"/>
  <line x1="604" y1="262" x2="684" y2="270" stroke="#1a1a17" stroke-width="1.5" marker-end="url(#d5k)"/>
  <line x1="604" y1="382" x2="684" y2="302" stroke="#1a1a17" stroke-width="1.5" marker-end="url(#d5k)"/>
  <text x="644" y="252" text-anchor="middle" font-family="ui-monospace,Menlo,monospace" font-size="12" fill="#6b6b66">hypotheses</text>

  <rect x="688" y="192" width="252" height="156" rx="18" fill="#ffffff" stroke="#1a1a17" stroke-width="1.5"/>
  <text x="814" y="242" text-anchor="middle" font-family="Georgia,serif" font-size="24" font-weight="600" fill="#1a1a17">Judge</text>
  <text x="814" y="268" text-anchor="middle" font-family="-apple-system,'Segoe UI',sans-serif" font-size="13" fill="#6b6b66">scored against the verified cause</text>
  <line x1="718" y1="286" x2="910" y2="286" stroke="#e4e0d6" stroke-width="1"/>
  <text x="814" y="312" text-anchor="middle" font-family="ui-monospace,Menlo,monospace" font-size="11" letter-spacing="1" fill="#6b6b66">blind to attempt identity</text>

  <line x1="150" y1="410" x2="150" y2="480" stroke="#1a1a17" stroke-width="1.5"/>
  <line x1="150" y1="480" x2="814" y2="480" stroke="#1a1a17" stroke-width="1.5"/>
  <line x1="814" y1="480" x2="814" y2="354" stroke="#1a1a17" stroke-width="1.5" marker-end="url(#d5k)"/>
  <text x="482" y="468" text-anchor="middle" font-family="ui-monospace,Menlo,monospace" font-size="12" fill="#6b6b66">judge only &#183; attempts never see it</text>

  <line x1="944" y1="250" x2="1008" y2="216" stroke="#1a1a17" stroke-width="1.5" marker-end="url(#d5k)"/>
  <line x1="944" y1="290" x2="1008" y2="324" stroke="#1a1a17" stroke-width="1.5" marker-end="url(#d5k)"/>

  <rect x="1012" y="170" width="204" height="92" rx="14" fill="#ffffff" stroke="#1a1a17" stroke-width="1.5"/>
  <text x="1114" y="203" text-anchor="middle" font-family="Georgia,serif" font-size="16" font-weight="600" fill="#1a1a17">Best&#8209;of&#8209;3 wins</text>
  <text x="1114" y="224" text-anchor="middle" font-family="-apple-system,'Segoe UI',sans-serif" font-size="11.5" fill="#6b6b66">sev1 runs N threads</text>
  <text x="1114" y="241" text-anchor="middle" font-family="-apple-system,'Segoe UI',sans-serif" font-size="11.5" fill="#6b6b66">with a selection pass</text>

  <rect x="1012" y="298" width="204" height="92" rx="14" fill="#ffffff" stroke="#1a1a17" stroke-width="1.5"/>
  <text x="1114" y="331" text-anchor="middle" font-family="Georgia,serif" font-size="16" font-weight="600" fill="#1a1a17">No lift</text>
  <text x="1114" y="352" text-anchor="middle" font-family="-apple-system,'Segoe UI',sans-serif" font-size="11.5" fill="#6b6b66">single thread stays;</text>
  <text x="1114" y="369" text-anchor="middle" font-family="-apple-system,'Segoe UI',sans-serif" font-size="11.5" fill="#6b6b66">claim falsified here</text>

  <text x="620" y="532" text-anchor="middle" font-family="-apple-system,'Segoe UI',sans-serif" font-size="15" fill="#55534d">Fixtures turn the experiment into replays, and the protocol publishes with the result.</text>
</svg>
<svg class="m" viewBox="0 0 620 660" role="img" aria-label="Mobile view: eval store fixtures feed three independent replays, a judge scores blind against the verified cause, and a pre-registered rule decides adoption or falsification">
  <defs><marker id="m5k" viewBox="0 0 10 10" refX="8.5" refY="5" markerWidth="8" markerHeight="8" orient="auto-start-reverse"><path d="M0 0L10 5L0 10z" fill="#1a1a17"/></marker></defs>
  <rect x="24" y="14" width="572" height="118" rx="16" fill="#F1EAE0"/>
  <text x="52" y="52" font-family="ui-monospace,Menlo,monospace" font-size="15" font-weight="700" letter-spacing="1.5" fill="#1a1a17">EVAL STORE</text>
  <text x="52" y="82" font-family="-apple-system,'Segoe UI',sans-serif" font-size="17" fill="#55534d">envelope + fixtures &#183; every read recorded</text>
  <text x="52" y="108" font-family="-apple-system,'Segoe UI',sans-serif" font-size="17" fill="#55534d">verified cause &#183; held back for the judge</text>
  <line x1="310" y1="140" x2="310" y2="174" stroke="#1a1a17" stroke-width="1.6" marker-end="url(#m5k)"/>
  <rect x="40" y="178" width="540" height="110" rx="20" fill="#141412"/>
  <text x="310" y="224" text-anchor="middle" font-family="Georgia,serif" font-size="26" font-weight="600" fill="#ffffff">Three attempts</text>
  <text x="310" y="256" text-anchor="middle" font-family="-apple-system,'Segoe UI',sans-serif" font-size="17" fill="#c9c4b8">independent replays, same inputs</text>
  <line x1="310" y1="296" x2="310" y2="330" stroke="#1a1a17" stroke-width="1.6" marker-end="url(#m5k)"/>
  <rect x="40" y="334" width="540" height="110" rx="20" fill="#ffffff" stroke="#1a1a17" stroke-width="1.6"/>
  <text x="310" y="380" text-anchor="middle" font-family="Georgia,serif" font-size="26" font-weight="600" fill="#1a1a17">Judge</text>
  <text x="310" y="412" text-anchor="middle" font-family="-apple-system,'Segoe UI',sans-serif" font-size="17" fill="#6b6b66">blind scoring against the verified cause</text>
  <line x1="310" y1="452" x2="310" y2="486" stroke="#1a1a17" stroke-width="1.6" marker-end="url(#m5k)"/>
  <rect x="24" y="490" width="572" height="150" rx="16" fill="none" stroke="#b9b4a9" stroke-width="1.6" stroke-dasharray="5 5"/>
  <text x="52" y="524" font-family="ui-monospace,Menlo,monospace" font-size="13" letter-spacing="1.5" fill="#6b6b66">PRE&#8209;REGISTERED RULE</text>
  <text x="52" y="558" font-family="-apple-system,'Segoe UI',sans-serif" font-size="17" fill="#26251f">wins &#8594; sev1 runs N threads + selection</text>
  <text x="52" y="588" font-family="-apple-system,'Segoe UI',sans-serif" font-size="17" fill="#26251f">no lift &#8594; single thread stays;</text>
  <text x="52" y="614" font-family="-apple-system,'Segoe UI',sans-serif" font-size="17" fill="#26251f">the claim is falsified on this task</text>
</svg>
<figcaption>A stratified sample of reconciled incidents replays three ways against recorded fixtures. The judge alone sees the verified cause and scores blind to attempt identity, and the adoption rule is written before the first replay.</figcaption>
</figure>

<p>The measurement contract's habit applies to experiments too, so the protocol is fixed before the first replay:</p>

<pre><code># fixed before the first replay runs
sample        300 reconciled incidents, stratified by severity and tier
attempts      3 per incident; independent sessions; identical inputs
judge         corroboration rubric, scored against the verified cause,
              blind to which attempt produced which hypothesis
comparison    judge-selected best-of-3 vs. attempt 1 alone
adopt when    sev1 accuracy improves by 2+ points and the two added
              attempts stay under 5% of fleet investigation spend
either way    the result publishes with this protocol attached
</code></pre>

<p>The 96 percent headline leaves little room, which is itself the point: if a third lever exists, the only place it can show up is the slice where single runs miss, and sev1 is the only slice where parallel sessions could ever be worth their cost. The cost bar is a spend ceiling rather than a per&#8209;diagnosis ratio deliberately: tripling attempts triples cost by construction, so a cost&#8209;per&#8209;correct bar could never pass and a bar that cannot pass is theater, while sev1's small share of volume is what makes the ceiling cheap to stay under. If the number clears the bar, sev1 investigations get N hypothesis threads and a selection pass, and disagreement between threads surfaces in the incident thread rather than being resolved silently, because two independent sessions reaching different causes from the same evidence is information the on&#8209;call should see, not noise for a judge to hide. If it does not clear, we have falsified the claim on our own task for the cost of a replay, and the single thread stays. Either way, the claim made earlier, that an investigation is one session rather than a fan&#8209;out of subagents, gets tested instead of defended. Best&#8209;of&#8209;N is not that fan&#8209;out; each attempt is still a single accountable thread, and the judge is a separate job, not a coordinator inside the session. The sentence stands today. If the experiment wins, it gains a qualifier, one accountable thread per hypothesis, and the qualifier will arrive with the experiment's number attached.</p>

<p>The conditional box in the boundary figure, meanwhile, aged in a specific direction. The May release that moved tool execution inside the perimeter also shipped MCP tunnels, in research preview, which reach MCP servers inside a private network without exposing them; the sandboxes run in public beta with Cloudflare, Daytona, Modal, and Vercel, or on infrastructure the customer controls. That covers execution and tool connectivity, two of the three things our boundary keeps inside. The session still runs hosted, so the condition on the box has narrowed to exactly the term it was drawn around: where the session lives. When that fits, the migration is small: compaction, caching, resume, and scheduling are native on the hosted runtime, so the loop supervisor and the scheduler get deleted, and the gateway, the incident index, the eval store, and the Skills registry stay where they are, because they were never the runtime's to hold. The swap validates the way a model swap does: the fixture corpus replays on both runtimes, and the hosted form promotes when it matches.</p>

<h2 id="takeaways">Takeaways</h2>

<ol class="takeaways">
<li>Choose the runtime by deployment constraint first, capability second. For us, session&#8209;state residency settled the question before feature comparison started, which put the investigation on the Agent SDK, with Managed Agents as the hosted form of the same design.</li>
<li>Right&#8209;size every call. Bounded tasks run as single Messages API calls with structured outputs; only the open&#8209;ended investigation pays for an agent session, and the largest model prices into the stall path, not the median.</li>
<li>Keep raw telemetry out of the context window. Code in the middle does it, through programmatic tool calling where retention terms fit and a bank‑side sandbox where they don’t; Tool Search keeps the catalog out of context, and evidence objects instead of dumps do it on the tool side.</li>
<li>Put procedures in Skills and authority in code. A Skill, a prompt, or a tool annotation can make the agent smarter; none of them can authorize an action.</li>
<li>A checker can block but never approve. The grader bounces unsupported claims back into the session, posts flagged rather than suppressing, and its verdicts reconcile like any other label, so the checker has an error rate instead of a reputation.</li>
<li>Label every diagnosis twice, once in the moment and once against the verified postmortem, and only compute accuracy over the verified set.</li>
<li>The Skill catalog scales through ownership. Postmortems draft the updates, CI gates the publishing, and service teams, not a central team, keep the content true.</li>
<li>Test strategy claims on the corpus before adopting them. A replayable, verified incident set turns a roadmap debate, best&#8209;of&#8209;N included, into a pre&#8209;registered experiment that costs a replay rather than a rollout.</li>
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
<li><a href="https://claude.com/blog/claude-managed-agents-updates">Self&#8209;hosted sandboxes and MCP tunnels</a> in Managed Agents</li>
</ul>

<p class="sre-footnote">Ada, the read&#8209;only beta, predates this work and proved the reasoning use case. The architecture here is the v2 design and phased implementation I led, with incident&#8209;response, observability, security, and service teams owning their parts. Platform statuses and API syntax checked against Anthropic documentation, July 2026.</p>


</div>
