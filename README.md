# APM Triage Agent — take-home package

*Tech Ops — APM team · AI & automation proposal · Dmytro · July 2026*

> **Automate the noise. Assist the judgment. Never touch the money.**

This repository accompanies my take-home proposal for the Technical Support
Engineer (APM) role: how AI and automation can improve the APM team's incident
operations — and, just as importantly, where they must not act. The centerpiece
is a **triage agent** designed end-to-end (trigger → gates → AI reasoning →
allowlisted actions → audit trail) and implemented here twice: as a runnable
Python prototype and as an interactive browser simulator.

## Try it in 30 seconds — no API key needed

```bash
python3 triage_agent.py --list
python3 triage_agent.py --scenario pix_provider_outage
```

Mock mode is the default: the reasoning layer is a transparent heuristic stub
with the same output contract as a real model, so the full pipeline runs
offline. With an `ANTHROPIC_API_KEY` set, `--live` switches Layer 2 to an
actual LLM (`pip install -r requirements.txt` first).

**Interactive simulator (same logic, in the browser):**
https://thenameisdmitry.github.io/apm-triage-agent-demo/ — or just open
`docs/index.html` locally.

## The design in one diagram

Rules where decisions are deterministic, AI where correlation and language
add value, humans wherever money or external parties are involved:

```mermaid
flowchart TD
    classDef rules fill:#dbeafe,stroke:#2563eb,color:#1e3a5f
    classDef ai fill:#ede9fe,stroke:#7c3aed,color:#3b1d6e
    classDef human fill:#fef3c7,stroke:#d97706,color:#7c3d00
    classDef stop fill:#f1f5f9,stroke:#94a3b8,color:#334155

    subgraph LEGEND[" Legend "]
        direction LR
        L1["Deterministic rules"]:::rules
        L2["AI reasoning"]:::ai
        L3["Human decision"]:::human
    end

    A["Metrics pipeline<br/>success rate, latency, decline codes"]:::rules --> B{"Threshold breach vs<br/>dynamic baseline?"}:::rules
    B -- "no" --> Z1["No action"]:::stop
    B -- "yes" --> G1{"G1 — volume gate"}:::rules
    G1 -- "below floor" --> Z2["Suppress: statistical noise,<br/>logged for threshold tuning"]:::stop
    G1 -- "pass" --> G2{"G2 — dedup gate"}:::rules
    G2 -- "matches open incident" --> Z3["Attach as evidence<br/>to existing incident"]:::rules
    G2 -- "pass" --> G3{"G3 — maintenance gate"}:::rules
    G3 -- "active window" --> Z4["Annotate and downgrade"]:::stop
    G3 -- "pass" --> E["Evidence assembly:<br/>saved Kibana queries, provider status,<br/>deploy feed, merchant tickets"]:::rules
    E --> AI1["AI: correlate sources, classify cause,<br/>propose severity + confidence"]:::ai
    AI1 --> C{"confidence ≥ 0.6?"}:::rules
    C -- "no" --> H1["Human validates manually —<br/>card flagged LOW CONFIDENCE"]:::human
    C -- "yes" --> CARD["Triage card to Slack + ticket<br/>with evidence links"]:::rules
    CARD --> P{"SEV1/2 and<br/>confidence ≥ 0.7?"}:::rules
    P -- "yes" --> PG["Page on-call"]:::rules
    P -- "no" --> H2["Human: confirm or override severity,<br/>declare incident"]:::human
    PG --> H2
    H2 --> AI2["AI: live incident summary +<br/>stakeholder comms drafts"]:::ai
    AI2 --> H3["Human approves and sends comms;<br/>decides mitigation: rollback,<br/>reroute, provider escalation"]:::human
    H3 --> AI3["AI: RCA draft +<br/>runbook update diff"]:::ai
    AI3 --> H4["Human review — merge runbook PR"]:::human
    H1 --> FB["Feedback loop: overrides + weekly samples<br/>drive rubric, prompt and gate tuning"]:::rules
    H2 --> FB
    H4 --> FB
```

## What the prototype demonstrates

- **Gates run before any AI.** Three deterministic gates (volume, dedup,
  maintenance) stop roughly two thirds of noise — cheap, explainable, and no
  tokens spent. Only validated, enriched signals reach the model.
- **The AI verdict is always a proposal.** Paging happens only for SEV1/SEV2
  with confidence ≥ 0.7; low-confidence verdicts are flagged for human
  validation, and the agent is comfortable saying "unknown".
- **The agent has no write path to anything.** Its four actions are a fixed
  allowlist; in this demo they only print or append to local files. In
  production the same constraint lives at the IAM level, not in the prompt.
- **External text is untrusted.** Provider status pages and merchant tickets
  go into the prompt wrapped as `<untrusted_data>` — evidence to classify,
  never instructions to follow.
- **Every run is reconstructable.** An append-only JSONL decision trace
  (`traces/`) records inputs, gate results, the full prompt, the raw verdict,
  and every action taken or skipped.

## Scenarios

| Scenario | Path exercised |
|---|---|
| `pix_provider_outage` | full pipeline → provider_outage, SEV2, page on-call |
| `ideal_release_regression` | full pipeline → our_release (deploy correlation) |
| `sofort_low_volume` | stopped at G1 — statistical noise, no AI invoked |
| `klarna_duplicate` | stopped at G2 — attached to open incident INC-2041 |
| `pix_maintenance` | stopped at G3 — scheduled maintenance window |
| `wallet_unknown` | full pipeline → unknown cause, low confidence, no page |

## Repository layout

```
triage_agent.py       the prototype: gates → reasoning → allowlisted actions → trace
scenarios/            six synthetic alerts, one per pipeline branch
docs/index.html       interactive simulator (GitHub Pages serves this folder)
diagrams/             end-to-end automation flow (Mermaid source)
proposal/             written design: problem framing, use cases, agent deep-dive
requirements.txt      one optional dependency, used only by --live
```

The full proposal deck (problem framing, three use cases, agent design,
guardrails, KPI framework, staged rollout) is submitted alongside this repo.

---

*This package was itself built AI-assisted — the same working style it
proposes for the team.*
