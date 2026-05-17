# Reference synthesis: Interview 02 — David Chen, Meriton Health (Enterprise health insurer)

This is the "ideal" synthesizer output for transcript 02. Used as ground truth by the LLM-as-judge.

---

## TL;DR
David is a high-affinity Hindsight user who is structurally blocked from using it. The product is technically superior to his current internal alternative, but the absence of a HIPAA BAA makes it legally unusable for ~95% of his work. His feature priorities differ sharply from typical Pro users: he wants less AI-generated content, more raw transcript control, and excerpt-level (not summary-level) integrations.

## Key themes
1. **HIPAA/BAA is a hard gate.** Without a BAA covering Hindsight AND downstream model providers, no PHI-adjacent use case is legally viable.
2. **Inferior alternative is already in production.** Internal Teams-based summarizer covers the legal requirement but is "worse in every way except the legal way." Switching cost would be zero; the legal cost is total.
3. **Inverted AI preference.** Most Pro users want more AI synthesis; David wants less. Hallucinated action items in healthcare are not "oops," they are potential regulatory findings.
4. **Speaker labels matter more here than elsewhere.** Meetings with 12+ participants; current 4-name guessing is insufficient.
5. **Integration depth must be excerpt-level, not summary-level.** Pushing full summaries to Jira ticket bodies is noise; pushing selected transcript excerpts as Jira comments is signal.

## Jobs-to-be-done surfaced
- "Use a best-in-class meeting notes tool without a regulatory violation."
- "Capture what was said with enough fidelity to defend in an audit."
- "Push selected discussion fragments to the project tracker without dumping the whole meeting."
- "Identify speakers correctly when many people are in the room."

## Contradictions (intra-interview)
- David pays for Pro personally and uses it for vendor calls, yet won't (can't) use it for anything internal. Reveals a strong PMF shadow: the product wins personal love but loses the org-purchase loop entirely on a single missing artifact (BAA).

## Notable quotes
- "I'd switch tomorrow if we could get the BAA signed."
- "Most PMs aren't dealing with auditors."
- "I'm not paying for AI features I can't legally turn on."
- "The worst case is a settlement."

## Prioritized recommendations
1. **P0 — Pursue HIPAA BAA.** Highest-leverage single artifact. Locks/unlocks all enterprise health revenue.
2. **P1 — Build an "audit mode" or "raw-transcript-first" UX.** Less AI, more manual highlighting, defaults inverted relative to consumer Pro.
3. **P2 — Editable speaker labels with persistence and large-meeting support (10+ speakers).**
4. **P3 — Excerpt-to-Jira-comment integration**, not summary-to-ticket-body.
5. **P4 (strategic) — Investigate whether downstream model providers (OpenAI, Anthropic) offer BAAs**; bake into Business+/Enterprise tier as a guarantee.
