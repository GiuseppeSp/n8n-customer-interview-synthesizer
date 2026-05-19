# n8n-customer-interview-synthesizer

A multi-agent n8n pipeline that turns customer-interview transcripts into prioritized, PM-actionable insights — with evaluation, human-in-the-loop approval, shadow A/B testing, and drift monitoring scaffolded around the production path.

The interesting part of this build isn't the synthesis itself. It's the scaffolding that decides when to trust the synthesis. Four worker agents, a synthesizer, and a judge would technically produce output without any trust layer. What makes the artifact production-shaped is the judge, the score gate, the Slack approval flow, the shadow A/B testing, and the daily drift monitor.

**Status:** Phases 0-5 complete; Phase 6 (polish + writeup) is what this README is part of. Granular decisions, deviations, and honest limitations live in [BUILD-LOG.md](BUILD-LOG.md).

---

## The problem this addresses

Product managers do customer interviews. Then they spend hours synthesizing those interviews into themes, jobs-to-be-done, contradictions, quotes, and prioritized recommendations. Doing this manually is slow and inconsistent. Doing it with a single LLM call is fast but unreliable — outputs vary in quality, lack prioritization, occasionally hallucinate quotes, and quietly drop critical conditional commitments.

The PM-shaped question isn't "can AI synthesize an interview?" It's "can AI synthesize an interview *reliably enough that a PM would ship the output without re-reading the transcript?*" That requires evals, not just generation. It requires a way to detect when the AI is having a bad day. It requires a human escape hatch for the cases the AI handles badly.

This build is a working answer to that question.

---

## Architecture

### Main workflow (per-interview processing)

```
Manual / Webhook Trigger
  → Load transcript (Google Sheets)
  → 4 Worker Agents in parallel
       ├── Theme Extractor
       ├── JTBD Tagger
       ├── Contradiction Surfacer
       └── Quote Miner
  → Merge (sync barrier)
  → Synthesizer v1 (live)     parallel with     Synthesizer v2 (shadow)
  → Judge v1                   parallel with     Judge v2
  → Score Gate (IF score >= 7)
       True  → Save to results sheet (auto-approved)
       False → Slack #hindsight-approvals → human Approve/Reject → Save (human-approved)

  (Shadow v2 score also saved to results sheet for offline A/B comparison)
```

### Drift Monitor workflow (separate, daily)

```
Schedule Trigger (daily 9am)
  → Load golden transcript (Google Sheets)
  → Drift Synthesizer (single-pass LLM call)
  → Drift Judge (scores against the golden reference)
  → Save Drift Score (results sheet, approval_path = "drift_check")
  → IF score < threshold → Slack #hindsight-approvals 🚨 drift alert
```

Full architecture diagram on FigJam: [Option A: Trust-Layer-First Interview Synthesis](https://www.figma.com/board/dRU9fDpnG3i6VO5WpDxQ7H).

Reference template that inspired the shape: [Raoul Kahn's PM brief](https://raoulkahn.github.io/pm-brief/).

---

## The trust scaffolding (the actual differentiator)

Five working components, each addressing one production-AI failure mode.

### 1. LLM-as-judge

After the synthesizer produces an output, a separate LLM call scores it against a hand-curated reference synthesis on four dimensions:

- **Coverage** — did it identify the same main themes, JTBDs, and contradictions as the reference?
- **Specificity** — concrete and actionable, or vague?
- **Quote selection** — did it surface the same notable verbatim lines?
- **Prioritization** — are recommendations meaningfully ranked across P0-P3, or all flat at one level?

The judge returns per-criterion 0-10 scores plus an `overall_score` and a written justification. The overall score becomes the routing signal for the score gate below.

### 2. Score-gated routing

A simple IF node: if `overall_score >= 7`, the synthesis ships automatically. If below, the workflow pauses and routes to a human reviewer via Slack. This is the gate that prevents low-quality output from reaching the PM.

### 3. Human-in-the-loop approval via Slack

When the score is below threshold, the workflow:
1. Sends a structured Slack message to `#hindsight-approvals` containing the score breakdown, the judge's reasoning, the transcript filename, and Approve / Reject buttons.
2. Pauses on n8n's `Send and Wait for Response` node.
3. Resumes only after a human clicks a button.
4. Logs the human's decision (approved or rejected) to the results sheet for auditing.

See [`screenshots/01_slack_hitl_message.png`](screenshots/01_slack_hitl_message.png).

### 4. Shadow A/B testing

Synthesizer v1 (live) runs in parallel with Synthesizer v2 (shadow). v2 has a different prompt — in our case, stricter prioritization rules. Both are judged. v1's output ships to the PM as usual; v2's output is logged but never ships.

Score deltas over time tell us if v2 is consistently better. If yes, we'd promote v2 to live and start shadowing a new v3 candidate. This is the standard production-AI pattern for safely deploying prompt changes without risking degraded output for real users.

### 5. Daily drift monitor

A separate workflow runs daily on the golden set. It re-synthesizes a known transcript, scores the output, saves the score to history, and fires a Slack alert if the score drops below threshold. Catches three production failure modes:

- The model vendor silently upgrades the underlying LLM (output behavior shifts under you)
- A prompt edit unintentionally regressed something
- Input distribution shifts and the prompt no longer fits the actual transcripts coming in

See [`screenshots/06_slack_drift_alert.png`](screenshots/06_slack_drift_alert.png).

---

## Real findings from the build

These are the things that actually surfaced during the build, not the things I planned to find. They're the most honest portfolio signal.

### Finding 1: the multi-agent decomposition wasn't strictly better than a single well-prompted call

The drift monitor uses a single-pass synthesizer — one LLM call doing themes + JTBDs + contradictions + quotes + recommendations together. It scored 6/10 overall on the judge rubric.

The main pipeline's multi-agent synthesizer (4 specialist workers → merge → synthesizer) scored 4-5/10.

The simpler architecture *outperformed* the more complex one on this rubric. Possible reasons:
- The single-pass prompt was more focused and included explicit prioritization rules
- The multi-agent workers introduced noise (each emitted JSON strings the synthesizer then had to interpret)
- gpt-4o-mini handles a holistic synthesis well when prompted clearly

Implication: the multi-agent pattern isn't automatically better. It needs to justify its complexity. For this use case at this model size, the simpler approach may be the right answer. Worth investigating further before declaring multi-agent the winning shape.

### Finding 2: the shadow A/B caught that a "fix" wasn't actually a fix

The first iteration of Synthesizer v1 scored 3-4/10 on prioritization — it tagged every recommendation as P0 with no spread. I built Synthesizer v2 with stricter prioritization rules in the prompt, expecting a meaningful jump.

v2 scored 4/10 on prioritization. Marginal lift, not statistically interesting.

The eval told me the prompt change wasn't sufficient. The shadow A/B did its job: it stopped me from promoting a candidate that wasn't actually meaningfully better. The next iteration would need a different approach — JSON schema constraints to enforce priority distribution, few-shot examples, or a different model entirely.

This is the eval layer earning its keep on a real iteration cycle, which is the strongest possible portfolio story.

### Finding 3: n8n's `Send and Wait for Response` pauses the entire workflow

When the Slack HITL node is in waiting state, all parallel branches in the same workflow execution pause too. So Synthesizer v2 / Judge v2 / Save Shadow Score effectively run *after* the human clicks Approve, not truly in parallel with v1's path. Functionally correct but breaks the "shadow runs at the same time as live" mental model. Worth documenting because real production deployments would care about this for latency.

### Finding 4: Google Sheets auto-coerces numeric IDs on CSV import

When the `transcript_id` column has values "01", "02", "04" in the source CSV, Google Sheets imports them as numbers 1, 2, 4 and strips leading zeros. n8n filters against the string "01" then return zero rows. Cost ~15 minutes of debugging. Documenting here so future-me doesn't pay the cost twice.

---

## Honest limitations

This is a portfolio build, not a production system. The limitations are real.

- **Synthetic data, AI-drafted ground truth.** Both the transcripts (5 fake PM-interview transcripts about a fictional product called Hindsight) and the golden references (3 of the 5) were AI-generated within the same build session. The reference-based eval is therefore scoring AI output against AI-written ground truth. The methodology demonstrates the pattern; it does not validate production quality. A real deployment would use real customer interviews and PM-written references, ideally with inter-rater agreement across multiple reviewers.

- **Single-transcript drift monitor.** The drift monitor currently checks one golden transcript daily. Real production would loop over the full golden set (3+ transcripts) and compute a robust average. v2 enhancement: use n8n's `Execute Workflow` node to call the main pipeline on each transcript in a loop.

- **Hardcoded thresholds.** The score gate threshold is hardcoded at 7 (for HITL gating) and 5 (for drift alerting). Real production would compare to a rolling baseline from drift history and alert on deviations from the trend, not a fixed cutoff.

- **No LangWatch / OpenTelemetry instrumentation.** Token cost and latency per node are visible in n8n's execution logs but not aggregated for trend analysis. Adding LangWatch traces is a documented v2 enhancement.

- **Single LLM provider.** Workers, synthesizers, and judges all use OpenAI `gpt-4o-mini`. Real production would consider multi-provider fallback (Anthropic Claude as backup) and model routing by cost-vs-quality.

These are noted not as apologies but as artifacts of the v1 scope. Documenting them honestly is part of the build's value: it signals product judgment about what's worth doing now vs later.

---

## Repo layout

- [`BUILD-LOG.md`](BUILD-LOG.md) — working notes, decisions made, deviations from plan, findings, and what's deferred
- [`transcripts/`](transcripts/) — 5 synthetic PM-interview transcripts (fictional product Hindsight)
- [`transcripts/README.md`](transcripts/README.md) — documents the *planted material* the worker agents are expected to find. Useful for judging worker output quality without re-reading every transcript.
- [`transcripts/golden/`](transcripts/golden/) — 3 hand-curated reference syntheses (Maya, David, Tomás) plus the CSV imported into the Google Sheets golden set
- [`screenshots/`](screenshots/) — n8n canvas captures and Slack messages across phases

---

## Tech stack

| Layer | Tool | Notes |
|---|---|---|
| Workflow orchestration | n8n Cloud | Trial then self-host or paid |
| LLM | OpenAI `gpt-4o-mini` | Workers, synthesizers, judges |
| Storage | Google Sheets | Golden set + Synthesis Results |
| Human-in-the-loop and drift alerts | Slack (`#hindsight-approvals` channel) | OAuth credential in n8n |
| Architecture diagram | FigJam | Linked above |

Total OpenAI spend across the full build (every iteration, every test): under $1.

---

## Reproducibility

To rebuild from scratch you need:
1. An n8n Cloud account (free trial works)
2. An OpenAI API key with a small spending limit
3. A Google account (for Sheets)
4. A Slack workspace (free tier works)

The transcripts and golden references in this repo are public and freely reusable.

The full step-by-step build narrative — including the decisions, the dead ends, and the n8n quirks discovered along the way — lives in [BUILD-LOG.md](BUILD-LOG.md). Reading the BUILD-LOG gives the closest thing to "watch me build this" without me actually narrating.
