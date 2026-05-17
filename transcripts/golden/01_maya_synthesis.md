# Reference synthesis: Interview 01 — Maya Khan, Riverbend (Seed-stage fintech)

This is the "ideal" synthesizer output for transcript 01. The LLM-as-judge will compare the live synthesizer's output against this reference and score on coverage, specificity, recommendation quality, and quote selection.

---

## TL;DR
Maya is a high-frequency, multi-context Hindsight user who treats the auto-summary as a fast triage layer she still validates manually on high-stakes calls. Her core pain is not aggregate summary quality, it's a specific systematic miss (conditional commitments). Her favorite feature, transcript search, is the silent retention driver.

## Key themes
1. **Auto-summary as draft, not source of truth.** Maya values speed of delivery but routinely re-validates on consequential calls. The trust ceiling caps the time she actually saves.
2. **Conditional commitments are systematically lost.** When customers frame asks as "we'd do X if you can do Y," Hindsight captures X and drops Y. Multiple cited examples.
3. **Notion integration is shallow.** Summary lands in a page, doesn't tag, link, or update existing customer records. Creates copy-paste work.
4. **Action item attribution is naive.** Default-to-self assignment misfires when the action was clearly on the other party.
5. **Transcript search is the retention driver.** Single most-loved feature; framed as worth the entire Pro subscription on its own.

## Jobs-to-be-done surfaced
- "Triage decisions from my last several calls so I can act fast without re-listening."
- "Recover what a specific customer told me weeks or months ago."
- "Get action items into Notion without manual rework."
- "Catch high-stakes commitments before I respond to a customer."

## Contradictions (intra-interview)
- Maya "loves" the summary in aggregate but explicitly doesn't trust it on high-stakes calls. Not a logical contradiction, but a meaningful tension: perceived value is high but conditional. The product team should understand that aggregate NPS may be masking the failure mode that hurts most.

## Notable quotes
- "I'm fourteen different roles in a trench coat. The tool needs to be one less thing I think about, not one more."
- "If the AI can't tell me what was decided, I might as well just listen to the recording again."
- "Those conditionals are where all my actual work hides."

## Prioritized recommendations
1. **P0 — Detect and flag conditional commitments.** "We'd do X if Y." Single biggest unlock for trust on high-stakes calls.
2. **P1 — Deepen Notion integration.** Tag-based linking to existing customer records, not just summary dump.
3. **P2 — Smarter action-item attribution.** Detect which party is being committed; don't default to recorder.
4. **P3 (delight) — Surface "this commitment has a dependency" callouts in the Slack summary ping.**
