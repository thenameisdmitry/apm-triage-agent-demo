# 3. Deep Dive: APM Triage Agent

*One agent, designed end-to-end. UC1 from the previous section, taken to implementation depth.*

## Trigger
- A monitoring alert fires: success rate for an APM (method × provider × region) drops below its dynamic baseline, OR callback delivery lag exceeds threshold, OR a spike in a specific decline code.
- Manual trigger: an engineer can also invoke the agent on a suspicious signal ("triage this").

## Data inputs (all read-only)
1. Transaction metrics API — success rate, volume, latency, decline-code distribution (last 60 min vs. 7-day baseline).
2. Log search (Kibana/ELK) — pre-defined saved queries per APM; the agent selects which to run, never writes free-form queries against prod.
3. Provider status pages / status APIs.
4. Deploy & config-change feed (CI/CD events for payment services).
5. Incident registry — open and recent incidents (dedup).
6. Ticket system — recent merchant complaints tagged to the same APM.

## Decision logic (two layers, strictly ordered)

**Layer 1 — deterministic gates (no AI):**
- G1 Volume gate: below N transactions/hour → statistical noise, log & suppress.
- G2 Dedup gate: matches an open incident signature → attach as evidence to that incident, stop.
- G3 Maintenance gate: provider has a scheduled maintenance window → annotate, downgrade.

Only signals passing all gates reach Layer 2. Rationale: gates are cheap, explainable, and remove ~60–70% of noise before a single model token is spent.

**Layer 2 — AI reasoning:**
- Correlate evidence across the six inputs.
- Classify probable cause: `provider_outage | our_release | merchant_side | traffic_anomaly | unknown`.
- Propose severity (SEV1–SEV4) using a written rubric (impact × scope × trend) included in the prompt.
- Produce a confidence score (0–1) with stated reasons for uncertainty.

## Actions (allowlisted — the agent can do these and nothing else)
1. Post a triage card to the on-call Slack channel (verdict, cause, severity proposal, confidence, evidence links, suggested runbook step).
2. Create/update a ticket with the structured triage record.
3. Page on-call — only for proposed SEV1/SEV2 with confidence ≥ 0.7.
4. Attach evidence to an existing incident (dedup path).

## Human approval points
- **Severity is always a proposal.** A human confirms or overrides; the agent never declares an incident.
- Confidence < 0.6 → the card is explicitly marked "LOW CONFIDENCE — needs human validation" and no page is sent.
- `unknown` cause → straight to human, no auto-routing.
- Any external communication (merchant, provider) — drafted at most, never sent.

## Guardrails
- Read-only credentials to every data source; no write access to payment systems, routing, or configuration — enforced at the IAM level, not the prompt level.
- Fixed action allowlist (the 4 actions above); anything else is impossible by construction.
- External text (status pages, merchant tickets) is treated as untrusted data: parsed and classified, never executed as instructions.
- Data minimization in prompts: transaction IDs and masked identifiers only, no cardholder data.
- Rate limiting: max K triage runs per signal per hour → no alert storms of the agent's own making.
- Kill switch: one env flag reverts the team to fully manual triage.

## Auditability
- Every run writes a decision trace: inputs retrieved (with timestamps), gates passed/failed, model prompt + raw output, final card, human confirm/override.
- Traces are append-only and attached to the ticket → every RCA can reconstruct *why* the agent said what it said.
- Human overrides are labeled and become the evaluation set for improving the rubric and prompts (see feedback loop).

## Feedback loop (bonus)
- Weekly: sample 10% of auto-triaged alerts + 100% of overrides; measure verdict precision and severity agreement.
- Disagreements feed a small labeled dataset → prompt/rubric tuning; gate thresholds re-tuned monthly from false-suppress stats.

## Cost / latency notes (bonus)
- Layer 2 runs on a fast, cheap model (triage is classification + summarization, not deep reasoning); target < 15 s per run, ~$0.01–0.03 per triage.
- RCA drafting (UC3) uses a stronger model — it runs post-incident where latency is irrelevant and quality matters.
