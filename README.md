# n8n-customer-interview-synthesizer

A multi-agent pipeline built in n8n that takes raw customer-interview transcripts and produces a prioritized, PM-actionable synthesis, with evaluation and human-in-the-loop scaffolding around the production path.

**Status:** active build. This README will expand into a full case study as the build completes (Phases 4-6 still pending). See [BUILD-LOG.md](BUILD-LOG.md) for the working notes and architectural decisions.

## What it does

Given a customer-interview transcript, the workflow:

1. Runs four specialist worker agents in parallel:
   - **Theme Extractor** clusters recurring topics
   - **JTBD Tagger** identifies jobs-to-be-done
   - **Contradiction Surfacer** flags tensions within or across interviews
   - **Quote Miner** pulls the most quotable verbatim lines
2. Merges their outputs and feeds them to a **Synthesizer** agent that produces a unified structured insight report.
3. Runs an **LLM-as-judge** that scores the synthesis against a reference set on coverage, specificity, quote selection, and recommendation prioritization.
4. If the score clears a threshold, the synthesis is saved to a results Sheet. If not, the workflow pauses and sends a Slack message with the score breakdown and Approve / Reject buttons.

## Why this shape

The interesting part of the build isn't the synthesizer. The interesting part is the scaffolding that decides when to trust its own output. The four worker agents and the synthesizer would technically work without any of it. What makes the artifact production-shaped is the judge, the threshold, the Slack human gate, and (planned) the daily drift monitor.

That framing maps directly onto what production AI teams actually struggle with: evals, observability, guardrails, reliability, and human-in-the-loop patterns. The synthesis use case is the substrate; the trust layer is the build.

## Architecture at a glance

```
Trigger
  → Load Transcript (Google Sheets)
  → 4 Worker Agents (parallel: themes, JTBDs, contradictions, quotes)
  → Merge (sync barrier)
  → Synthesizer v1
  → LLM-as-Judge (vs golden-set reference)
  → Score Gate (IF score >= 7)
       True  → Save to results Sheet
       False → Slack HITL approval → Save (with audit trail)
```

Full architectural decisions, deviations from the original plan, and known limitations are documented in [BUILD-LOG.md](BUILD-LOG.md).

## Repo layout

- `BUILD-LOG.md` — working notes, decisions, honest limitations
- `transcripts/` — 5 synthetic PM-interview transcripts (fictional product "Hindsight")
- `transcripts/README.md` — documents the planted material the worker agents are expected to surface
- `transcripts/golden/` — 3 hand-curated reference syntheses + a CSV imported into Google Sheets for use as eval ground truth
- `screenshots/` — n8n canvas, Slack approval message, full pipeline states

## Known limitations (will expand in the final writeup)

- Both transcripts and golden references are AI-generated within the same build session. The reference-based eval is therefore scoring AI output against AI-written ground truth, which is documented honestly. A production deployment would replace synthetic data with real customer interviews and PM-written references with inter-rater agreement.
- v1 currently uses sequential then parallel execution patterns depending on the section; see BUILD-LOG for the choice rationale.
- Drift monitoring workflow (Phase 5) is designed but not yet built.

## Tools

- n8n Cloud (workflow orchestration)
- OpenAI `gpt-4o-mini` (workers, synthesizer, judge)
- Google Sheets (transcript inputs, golden set, results log)
- Slack (HITL approval channel)
