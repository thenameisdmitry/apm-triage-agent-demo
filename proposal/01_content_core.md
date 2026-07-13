# AI-Assisted Operations for the APM Team: Content Core
*(Working draft; source of truth for the slide deck and demo)*

## 1. Problem Framing

### The incident lifecycle, decomposed

Every APM signal: whether it starts as a Datadog alert, a provider status update, or a merchant complaint: travels through the same stages. The right question is not "where can we use AI?" but "what kind of decision happens at each stage?"

| Stage | Nature of decision | Best fit | Why |
|---|---|---|---|
| 1. Detection | Numeric thresholds, anomaly stats | **Rules** | Deterministic, cheap, explainable. AI adds nothing here. |
| 2. Signal validation | Correlating noisy, multi-source context | **AI + rule gates** | Fuzzy matching across status pages, deploys, tickets: this is where humans burn most time. |
| 3. Evidence collection | Running known queries (Kibana, SQL) | **Rules (scripted)** | Queries are known in advance; automation, not intelligence. |
| 4. Impact & severity | Judgment under uncertainty | **AI proposes, human confirms** | AI drafts severity + impact estimate with confidence; a human owns the SEV call. |
| 5. Routing / escalation | Mapping issue → owner | **Rules first, AI for free text** | Routing tables cover 80%; AI classifies unstructured merchant complaints into the same taxonomy. |
| 6. Stakeholder comms | Wording, tone, accountability | **AI drafts, human approves: always** | External-facing text is never sent autonomously. |
| 7. Mitigation actions | Changes to money flow, routing, config | **Human only** | Anything touching live payment traffic stays manual by design. |
| 8. RCA & runbooks | Synthesis of what happened | **AI drafts, human reviews** | High-leverage summarization; review cost is low, time saved is high. |

### Where AI must NOT act autonomously
- Declaring or downgrading SEV1/SEV2 incidents.
- Any communication sent to merchants or providers.
- Any change to routing, cascading, provider configuration, or transaction processing.
- Closing incidents.

### Risks & limitations to design around
- **Hallucination** → the agent may only cite retrieved evidence (logs, metrics, status pages); every claim in its output links to a source. No source: no claim.
- **Calibration / false confidence** → explicit confidence score; below threshold, the agent hands off to a human instead of guessing.
- **Bad or stale input data** → freshness checks on every data source; a triage verdict built on a 40-minute-old status page is flagged as degraded.
- **Prompt injection via external text** → merchant tickets and provider status pages are untrusted input; they are data to be classified, never instructions to follow.
- **PII / compliance** → data minimization: masked PANs, no cardholder data in prompts; EU data residency respected in model choice.
- **Automation complacency** → weekly sampled audit of auto-triaged alerts; the team keeps verifying a random 10%.

## 2. Three AI Use Cases

### UC1: Alert Triage & Signal Validation Copilot
- **Input:** monitoring alert (e.g., Pix success rate drop), last-hour transaction metrics, provider status pages, deploy feed, currently open incidents.
- **AI action:** validate the signal: is it real, already known, or noise? Correlate across sources, propose probable cause category (provider outage / our release / merchant-side / traffic anomaly), attach confidence score.
- **Output:** structured triage card in Slack: verdict, evidence with links, proposed severity, suggested next runbook step.
- **User:** on-call Tech Ops engineer.
- **Operational value:** cuts time-to-validation from ~10–15 min of manual dashboard-hopping to <1 min; reduces alert fatigue. Measured by MTTA and % of alerts auto-triaged correctly.

### UC2: Incident Summarization & Comms Drafting
- **Input:** incident ticket, Slack incident channel thread, key log excerpts, impact metrics.
- **AI action:** maintain a live structured summary (affected payment methods, merchants, regions, success-rate delta, timeline of events); draft the next stakeholder update in the team's template.
- **Output:** summary block pinned to the incident channel + ready-to-review comms draft.
- **User:** incident commander; secondarily support/account managers.
- **Operational value:** during a SEV, summarizing and communicating is the biggest cognitive tax on the IC. Consistent updates every N minutes without pulling the IC away from resolution. Measured by update cadence adherence and IC feedback.

### UC3: RCA & Runbook Copilot
- **Input:** closed incident: timeline, actions taken, resolution; vector search over past incidents and existing runbooks.
- **AI action:** draft the RCA (sequence of events, contributing factors, detection gaps); detect recurrence patterns ("3rd iDeal callback delay this quarter"); propose concrete runbook diffs.
- **Output:** RCA draft + suggested runbook update as a reviewable PR.
- **User:** Tech Ops team; findings feed Product/Engineering backlog.
- **Operational value:** RCAs stop being a chore that slips; institutional knowledge compounds. Measured by RCA completion rate, repeat-incident rate.

### Why these three
They cover the full lifecycle (before / during / after an incident), each attacks a real time sink, and none of them lets the AI touch payment traffic: the risk profile is "wrong text a human reviews," never "wrong action in production."
