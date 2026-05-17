# Reference synthesis: Interview 04 — Tomás Ribeiro, Substrate (Devtools, serverless runtime)

This is the "ideal" synthesizer output for transcript 04. Used as ground truth by the LLM-as-judge.

---

## TL;DR
Tomás is a high-affinity Pro/Business user whose pain points cluster around domain-specific transcription accuracy and the gap between action-item extraction and ticket creation. He values the structure of summaries but identifies a systematic miss on architectural conditional commitments. Most-loved feature: transcript search (consistent with Maya, suggesting a broader retention pattern across personas).

## Key themes
1. **Engineering jargon mis-transcription is a continuous tax.** Custom-vocabulary feature helps but requires constant maintenance as the stack evolves.
2. **Action-item → ticket gap is unaddressed.** When the team says "let's open a ticket," the action item lands in Slack and needs manual copy-paste into Linear. The data is already present; the integration is missing.
3. **Architectural conditional commitments are lost.** "Revisit when we move to Postgres 16" → summary captures "team discussed Postgres," drops the conditional. Same failure mode as Maya's customer-call conditionals: this is a systemic AI bug, not a domain-specific one.
4. **Sentiment analysis is unwanted feature noise.** Explicitly flagged as "useless theater" in engineering meetings.
5. **AI-drafted emails don't fit technical contexts.** Lack of domain awareness produces drafts that read as off-key. Tomás writes his own.
6. **Searchable transcript history is the retention driver.** Mirrors Maya. Used to recover settled decisions and avoid re-litigation.

## Jobs-to-be-done surfaced
- "Capture engineering jargon correctly without manual maintenance."
- "Get action items into Linear automatically with traceability to the meeting."
- "Recover settled technical decisions weeks later to avoid re-arguing them."
- "Skip features that don't help (sentiment, email drafts) without seeing them in my output."

## Contradictions (intra-interview)
- Tomás values the *shape* of summaries but is critical of specific AI features within them (sentiment, email drafts). Implicit tension: he wants the AI to be smarter and narrower, not broader. Product team should read this as "ship fewer, sharper AI features," not "do more AI."

## Cross-interview observations
(These would be picked up by a cross-batch synthesis run, noted here for completeness.)
- Tomás wants deeper integrations (Linear-specific). Sarah (interview 05) wants fewer integrations. Different work shapes; both are right for their context.
- Tomás distrusts AI-drafted emails. Sarah lives off them. Same feature, opposite reactions, different downstream stakeholders (engineers vs. clients).
- Conditional-commitment loss appears in both Maya's customer calls and Tomás's engineering meetings. Strong evidence the issue is in the summarizer's logic, not the domain.

## Notable quotes
- "It transcribed Kubernetes as 'good Burnett's' for a month."
- "Sentiment analysis on a sprint planning is comedy."
- "The drafts always sound like they were written by a sales person who showed up to the wrong meeting."
- "Saved me from re-litigating a settled decision."

## Prioritized recommendations
1. **P0 — Build first-class Linear integration.** Auto-create issues from action items, link back to the meeting timestamp.
2. **P1 — Auto-detect engineering jargon.** Parse repo metadata (package.json, terraform configs, etc.) instead of requiring manual vocab maintenance.
3. **P2 — Detect architectural conditional commitments.** Same root cause as Maya's customer-call conditionals; one fix benefits both segments.
4. **P3 — Make sentiment analysis togglable and default-off for meetings tagged "engineering."**
5. **P4 (strategic) — Improve AI-draft emails with context (linked Linear/GitHub issues, prior thread)** or remove the feature for technical workspaces.
