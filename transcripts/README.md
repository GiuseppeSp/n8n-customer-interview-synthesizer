# Transcript test corpus

5 synthetic customer-interview transcripts about a fictional AI meeting notes tool. Used as input data for the n8n synthesis pipeline. Designed so the worker agents (Theme Extractor, JTBD Tagger, Contradiction Surfacer, Quote Miner) have rich, varied material to find.

## The fictional product: Hindsight

An AI meeting notes tool. Competes with Otter, Granola, Fireflies. Hypothetical feature set used consistently across all 5 transcripts:

- Records meetings (desktop app or bot in Zoom/Google Meet/Teams)
- Auto-transcribes
- AI-generated summaries with decisions + action items
- Searchable transcript history
- Integrations: Slack, Notion, Linear, Jira, Asana
- Newer features: AI-drafted follow-up emails, sentiment analysis, speaker identification, custom-vocabulary upload (Business tier only)
- Pricing: Free (5 meetings/mo), Pro ($20/user/mo), Business ($40/user/mo)
- Known issues: jargon mis-transcription, summaries miss conditional commitments, speaker labels often wrong with no edit affordance, no multi-workspace separation

## The 5 personas

| # | File | Persona | Company shape | Primary use case |
|---|------|---------|---------------|------------------|
| 01 | 01_maya_seed_startup.txt | Maya Khan, Head of Product | Seed-stage fintech, 4 ppl | Customer + investor calls |
| 02 | 02_david_enterprise_healthcare.txt | David Chen, Senior PM | Large health insurer (~4k ppl) | Blocked by compliance; uses inferior internal tool |
| 03 | 03_priya_consumer_fitness.txt | Priya Nair, Senior PM | Consumer fitness app (~2M users) | User research interviews only |
| 04 | 04_tomas_devtools_technical.txt | Tomás Ribeiro, Senior PM | Serverless devtools company | Sprint planning, RFC reviews, customer calls |
| 05 | 05_sarah_independent_consultant.txt | Sarah Whitfield, Independent PM | Solo, 5 active clients | Client meetings across multiple companies |

## Planted material for each worker agent

### Common themes (Theme Extractor should find these)
- Auto-summary quality is the central feature (all 5)
- Action item extraction quality (4 of 5)
- Integration depth vs. flexibility (4 of 5)
- Searchable transcript history (3 of 5: Maya, Tomás, Priya)
- Trust and control of AI outputs (4 of 5)

### JTBDs (JTBD Tagger should find these)
- "Remember what was decided so I don't have to re-listen" (Maya, David)
- "Share post-meeting summary with the right people" (Sarah)
- "Find a specific past discussion quickly" (Maya, Priya, Tomás)
- "Get action items into my project tracker automatically" (Maya, Tomás)
- "Stay legally compliant with recording" (David)
- "Preserve nuance from user interviews" (Priya)
- "Separate client contexts" (Sarah)

### Contradictions (Contradiction Surfacer should find these)
- Maya LOVES auto-summary; Priya thinks summaries are harmful. Different use cases (decision recall vs. user research nuance).
- David wants LESS AI; Tomás wants MORE AI. Different risk tolerance / domain.
- Sarah lives off the AI-drafted follow-up email; Tomás and Priya don't trust it. Same feature, opposite reactions.
- Tomás wants DEEP integrations (Linear-specific); Sarah wants SHALLOW integrations (clean copy-paste). Different work setups.
- Maya finds summary structure great; David won't use them without regulatory cover.

### Quotable lines (Quote Miner should find these)
- "I'm fourteen different roles in a trench coat" (Maya)
- "If the AI can't tell me what was decided, I might as well just listen to the recording again" (Maya)
- "I'd switch tomorrow if we could get the BAA signed" (David)
- "I'm not paying for AI features I can't legally turn on" (David)
- "Most PMs aren't dealing with auditors" (David)
- "Summaries kill nuance. In a user interview, the nuance IS the insight" (Priya)
- "It transcribed Kubernetes as 'good Burnett's' for a month" (Tomás)
- "Sentiment analysis on a sprint planning is comedy" (Tomás)
- "The AI follow-up email is the only feature I'd actually pay double for" (Sarah)
- "I bill by the hour, but I really earn by saving my clients time" (Sarah)

## Golden set selection (for the eval layer)

For the LLM-as-judge, we'll hand-write reference syntheses for 3 of these 5 transcripts:
- 01 Maya (covers most JTBDs, clearest quotes)
- 02 David (covers the compliance edge case)
- 04 Tomás (covers technical/jargon edge case)

The other 2 (Priya, Sarah) serve as held-out test data that the pipeline processes without a reference comparison.
