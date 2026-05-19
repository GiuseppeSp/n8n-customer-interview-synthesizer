# BUILD-LOG: Customer Interview Synthesis Pipeline

Working log for the n8n build. Updated as we go. Reverse chronological session entries at the bottom.

---

## Locked design (as of 2026-05-17)

**Option A: Trust-layer-first synthesizer.**

The product is a customer-interview synthesis pipeline. The artifact differentiator is the trust scaffolding around it (judge, HITL gate, drift monitor), not the synthesizer alone.

### Architecture (two workflows, same n8n project)

**Main workflow (triggered on transcript upload):**

| # | Block | What it does |
|---|---|---|
| 1 | Trigger | Manual, webhook, or Drive-upload event |
| 2 | Load Transcripts | Fetch transcript text from Drive or Sheets |
| 3a | Theme Extractor (LLM) | Clusters recurring topics |
| 3b | JTBD Tagger (LLM) | Tags jobs-to-be-done moments |
| 3c | Contradiction Surfacer (LLM) | Flags self-contradictions or cross-interview contradictions |
| 3d | Quote Miner (LLM) | Pulls best verbatim quotes |
| 4 | Merge | Waits for all 4 workers, combines outputs |
| 5a | Synthesizer v1 (LIVE) | Prompt A. Output ships. |
| 5b | Synthesizer v2 (SHADOW) | Prompt B. Graded but doesn't ship. |
| 6 | LLM-as-Judge | Scores synthesizer output vs golden-set rubric |
| 7 | Decision (IF node) | Score >= threshold? |
| 8a | Save + log (Yes) | Persist report, log full trace to LangWatch |
| 8b | HITL Wait (No) | Pause workflow |
| 9 | Slack approve/reject | Resume workflow on click |

**Drift monitor (daily cron, separate workflow):**

| # | Block | What it does |
|---|---|---|
| 10 | Cron (daily, 9am) | Schedule trigger |
| 11 | Re-run pipeline on Golden Set | Same workers + 5a (live only, NOT 5b) on frozen 3-transcript golden set |
| 12 | Compare scores to baseline | Judge today's output, compare to history |
| 13 | Drift detected? (IF) | Threshold check |
| 14a | Slack alert (Yes) | Page on quality regression |
| 14b | No-op (No) | Append score to history, no alert |

### What the drift monitor is measuring

Output quality of the LIVE synthesizer (5a) on a fixed input (golden set). Catches:
- Model vendor silently upgrading the underlying model
- Prompt edits that regressed something
- Input distribution shift (if real inputs start looking different from the golden set)

NOT measuring 5b (it's shadow, doesn't ship). NOT measuring per-worker drift (v2 enhancement).

### FigJam diagram of the architecture

https://www.figma.com/board/dRU9fDpnG3i6VO5WpDxQ7H

---

## Tech stack and cost ceiling

| Layer | Tool | Cost |
|---|---|---|
| Orchestration | n8n Cloud (14-day trial) | Free until 2026-05-31, then $20/mo or self-host |
| Worker agents | Claude Haiku 4.5 (or Groq Llama for cost) | Pay per token |
| Synthesizer | Claude Sonnet 4.6 | Pay per token |
| Judge | Claude Sonnet 4.6 | Pay per token |
| Storage / golden set | Google Sheets | Free |
| HITL | Slack | Free tier |
| Observability | LangWatch free tier | 50k observations/mo |
| Diagrams | FigJam | Free |

Estimated total LLM spend across the full build: under $30. Drift monitor adds ~$0.50/week.

---

## Build phases

Sequential. Don't skip ahead. Screenshot at the end of each phase.

### Phase 0: Setup (DONE 2026-05-17)
- [x] Sign up at app.n8n.cloud (trial: 14 days, 1000 executions)
- [x] Inspected "Build your first AI agent" template; saw error panel + Evaluations tab
- [x] Built hello-world: Manual Trigger → Basic LLM Chain → Google Gemini Chat Model (gemini-2.5-flash)
- [x] First successful execution: customer-quote pain-point extraction, 61 tokens, 4.1s

### Phase 1: Generate synthetic transcript data
- [x] Generate 5 synthetic PM-style interview transcripts (2026-05-17). Product: fictional AI meeting notes tool "Hindsight". 5 distinct PM personas. See `transcripts/README.md` for the planted themes / contradictions / quotes.
- [x] Draft reference syntheses for transcripts 01, 02, 04 (Maya, David, Tomás). Files under `transcripts/golden/`.
  - **KNOWN METHODOLOGY LIMITATION (to be stated explicitly in the eventual writeup):** Both the transcripts AND the golden syntheses are AI-generated within the same session. This means the reference-based eval is effectively scoring AI output against AI-written ground truth, with no real human PM judgment anchoring the eval. The eval here demonstrates the *methodology* (reference-based LLM-as-judge with drift monitoring), not production-grade validation. A real deployment would replace synthetic transcripts with real customer interviews AND replace AI-drafted golden references with PM-written ones (ideally with inter-rater agreement across multiple reviewers).
  - Decision (2026-05-17): accept this limitation and document it openly rather than pivot to rubric-based scoring. The portfolio piece's honesty about limitations is part of its value, per Kahn-template framing.
- [x] Store golden set in Google Sheets (imported via CSV: `transcripts/golden/golden_set.csv`). Sheet: "Hindsight Golden Set", URL: https://docs.google.com/spreadsheets/d/1XdFTZQo3P6UhprNNyr5s8z1G2FctqcqVMmXTPJGHe5E/edit?gid=0#gid=0 (n8n needs the full URL with `?gid=0` to resolve the specific tab)
  - Sheet ID for n8n: `1XdFTZQo3P6UhprNNyr5s8z1G2FctqcqVMmXTPJGHe5E`
  - Tab: Sheet1 (rename to `golden_set` if cleanup desired)
  - n8n will need Google Sheets credential authorized to read this sheet (set up in Phase 3 when wiring the judge node).

### Phase 2: Main workflow, happy path only (DONE 2026-05-17)
- [x] Build blocks 1-4 (trigger, load, 4 workers in parallel, merge)
- [x] Build block 5a only (no shadow yet)
- [x] Wire end-to-end on a single transcript (Maya, transcript_id=1)
- [x] Iterate on each worker prompt until output looks reasonable

**Architecture deviations from original plan:**
- Switched LLM provider from Gemini Flash to OpenAI `gpt-4o-mini` after hitting Gemini's free-tier daily quota (20 RPD on `gemini-2.5-flash`). Models in all 5 LLM nodes are now `gpt-4o-mini`.
- Replaced "Code node with cross-references" pattern with proper n8n Merge node. Reason: n8n's Code node has no "wait for all input branches" semantic; only Merge does. Code with multiple inputs fires on first input arrival, not after all complete. Documented as a real production-AI lesson: choose the right primitive for synchronization.
- Merge node configured: Mode=Combine, Combine By=Position, Number of Inputs=4. Acts as a sync barrier (the actual data combination happens via cross-references in the Synthesizer prompt).

**Known v1 quality issue:**
- Synthesizer tags all recommendations P0 instead of distributing across P0-P3. Prompt asks for prioritization but doesn't enforce a spread. Candidates for fix: (a) tighten prompt with explicit "use P0/P1/P2 sparingly" instruction, (b) upgrade Synthesizer to `gpt-4o`.

**Cost so far:**
- OpenAI usage to date: <$0.10. Workers + Synthesizer = ~7000 tokens/run on `gpt-4o-mini` ≈ $0.001/run. Plenty of headroom.

### Phase 3: Add the judge and gating
- [x] Build block 6 (judge) reading rubric from Sheets golden set. OpenAI gpt-4o-mini, prompt scores on coverage / specificity / quotes / prioritization (each 0-10, plus overall_score and justification).
- [x] Build block 7 (IF decision). Threshold = overall_score >= 7. Maya's run scored 4-5/10 → routes to False branch.
- [x] Build blocks 8b + 9 (HITL Wait + Slack approve/reject). n8n's "Slack > Send and Wait for Response" node with Approval response type. Channel: #hindsight-approvals. End-to-end tested: workflow paused on Slack node, message rendered cleanly with score breakdown and judge reasoning, Approve click resumed workflow.
- [x] Build block 8a (save + log path). Two `Google Sheets > Append Row` nodes: "Save (human-approved)" on Slack approve branch, "Save (auto-approved)" on Score Gate True branch. Both write to "Hindsight Synthesis Results" sheet (URL: https://docs.google.com/spreadsheets/d/1OyCoGAFBybvh2bKEYreLEpFFYvDejjj2BiLk5_8Z6gg/edit?gid=0#gid=0). Tested end-to-end: Maya's run scored 4 → Slack HITL → click Approve → row appended to results sheet with all 5 columns populated. (Auto-approved path not yet exercised with a real transcript that scores ≥ 7; will validate in a later run.)

### Phase 4: Add shadow A/B (DONE 2026-05-18)
- [x] Duplicate synthesizer node as Synthesizer v2 (shadow) with stricter prioritization rules (explicit "at most 1-2 P0, at least one P3" instructions, hard-coded "if you can't find a P3, you haven't synthesized hard enough" guidance)
- [x] Added Judge v2 (shadow), routed both v1 → Judge and v2 → Judge v2 in parallel from Merge Worker Outputs
- [x] Added Save Shadow Score node (logs v2 score with approval_path="shadow" to the same Hindsight Synthesis Results sheet)
- [x] End-to-end test on Maya transcript

**Real finding from the A/B test (worth surfacing in the writeup):**
| Score | v1 (live) | v2 (shadow) |
|---|---|---|
| overall_score | ~4-5 | ~5 |
| coverage | ~5-6 | 6 |
| prioritization | ~3-4 | 4 |

Prompt change moved the needle but barely. Judge specifically noted "recommendations are not consistently ranked in priority levels" even for v2 → marginal prompt tweaks weren't enough to fix the P0-spread issue. This is exactly the kind of finding the eval/shadow loop is supposed to produce: it caught that the candidate isn't yet meaningfully better than the live, so we WOULDN'T promote it.

Future v3 candidates worth trying: (a) structured output enforcement via JSON schema with priority enum + minOccurrences constraints, (b) few-shot examples showing a "good" priority spread, (c) different model (gpt-4o, claude-sonnet) for the synthesizer with same prompt.

**n8n quirk noted:** `Send and Wait for Response` node appears to pause the ENTIRE workflow including parallel branches, so Synthesizer v2 / Judge v2 / Save Shadow Score effectively run AFTER the human clicks Approve, not truly in parallel with v1's path. Functionally fine, but breaks the "shadow runs at same time as live" mental model. For real production this would matter for latency; for our portfolio it's a documentable observation, not a blocker.

### Phase 5: Drift monitor workflow (DONE 2026-05-18)
- [x] New workflow with Cron trigger ("Hindsight Drift Monitor", Schedule Trigger set to daily 9am, fires on manual Execute for testing)
- [x] Wire blocks 11-14:
  - Get rows in sheet (read Maya, transcript_id=1)
  - Drift Synthesizer (single-pass LLM Chain, gpt-4o-mini, themes + JTBDs + contradictions + quotes + recommendations in one prompt)
  - Drift Judge (same rubric as main workflow Judge, scores against the golden reference)
  - Save Drift Score (Google Sheets append, approval_path="drift_check", logs every run)
  - Drift Detected? (IF score < 5 for production; tested with threshold=7 to verify the alert path)
  - Slack Drift Alert (post message to #hindsight-approvals on True branch)
- [x] Tested end-to-end: Drift Synthesizer scored 6/10, alert fired correctly when threshold was 7, no alert when threshold is 5.

**Architectural deviations from original plan (documented honestly):**
- Drift monitor uses a single-pass synthesizer instead of the full multi-agent pipeline. This is a deliberate v1 simplification: drift monitoring needs to be cheap and fast, not necessarily replicate the entire production path. A v2 enhancement would use n8n's `Execute Workflow` node to call the main pipeline.
- Threshold is hardcoded (5). v2 enhancement: compare to a rolling baseline from the drift_history rows in the results sheet, alert if current score drops >2 standard deviations below mean.

**Real finding worth surfacing in writeup:**
The single-pass drift synthesizer (one LLM call doing everything) scored HIGHER than the main pipeline's multi-agent synthesizer (Drift score 6 vs main v1 score 4-5). This is a genuine architectural finding: the multi-agent decomposition is not strictly better than a single well-prompted call, especially when the single call includes strict prioritization rules. Worth investigating whether the multi-agent pipeline justifies its cost vs a single LLM call.

### Phase 6: Polish and document
- [ ] Add LangWatch traces to all LLM nodes
- [ ] Take rich screenshots of canvas, executions, judge scores, drift alerts
- [ ] Write architectural-decision notes (one per non-obvious choice)
- [ ] Document limitations honestly (what's mocked, what's brittle, what would break in prod)

---

## Open questions to revisit during build

- Which LLM provider for the workers? (Groq free Llama vs Claude Haiku 4.5 vs OpenAI GPT-4o-mini)
- How to format the judge rubric in Sheets? (per-criterion columns vs single JSON cell)
- Where to store score history for drift baseline? (Sheets vs LangWatch vs both)
- Worth doing per-worker drift monitoring in v2? (more diagnostic power, more setup)

---

## Session log

### 2026-05-18 (session 3)
- Hit Gemini free-tier daily quota (20 RPD); pivoted all LLM nodes to OpenAI gpt-4o-mini
- Solved the multi-input race condition by adopting n8n Merge node as a sync barrier (Code node has no wait-for-all-inputs semantic; only Merge does)
- Phase 3 COMPLETED: judge + score gate + Slack HITL approval (Send and Wait for Response, Approve/Reject buttons) + Save (auto-approved) + Save (human-approved). End-to-end tested on Maya's transcript: judge scored 4/10, routed to False, Slack pinged, click Approve, row appended to results sheet.
- Phase 4 COMPLETED: shadow A/B synthesizer. Synthesizer v2 with stricter prioritization rules runs in parallel with v1. Judge v2 scores v2 output. Save Shadow Score logs to results sheet with approval_path="shadow". Real finding: prompt change moved prioritization score from ~3-4 → 4 only (marginal). Shadow A/B correctly caught that v2 isn't meaningfully better than v1.
- Phase 5 COMPLETED: drift monitor workflow. Schedule Trigger → Get rows (Maya golden) → Drift Synthesizer (single-pass) → Drift Judge → Save Drift Score → Drift Detected? IF (score < 5) → Slack Drift Alert. Tested with threshold=7 to fire alert, then lowered to 5 for production behavior.
- Real finding from Phase 5: single-pass drift synthesizer scored 6/10 vs main multi-agent v1 at 4-5/10. The simpler architecture outperformed the complex one on the rubric. Documented as a candidate insight for the writeup.
- Phase 6 partial: pushed Git repo (https://github.com/GiuseppeSp/n8n-customer-interview-synthesizer) as public, added profile-table entry, expanded README to full case study, swapped Langfuse references → LangWatch (consistent with Cookbook RAG project).
- 7 portfolio screenshots captured (HITL message + canvas states + drift alert + drift monitor canvas).
- Decision logged: em dashes are allowed in this project's artifacts (override of global CLAUDE.md style rule).
- Next session: commit + push the Phase 4/5/6 changes, then the build is shippable.

### 2026-05-17 (session 2)
- Locked Option A architecture
- Built FigJam diagram of full flow (link above)
- Confirmed n8n Cloud trial path (vs self-host)
- Created this BUILD-LOG
- Phase 0 DONE: n8n Cloud signup, hello-world workflow (Manual Trigger → Basic LLM Chain → Gemini)
- Phase 1 DONE: 5 synthetic transcripts written, 3 golden references drafted, CSV generated, Google Sheet imported and live.
- Phase 2 DONE: Main workflow happy path, 4 parallel workers + Merge + Synthesizer v1, tested end-to-end on Maya.
- Initial Git push to GitHub (https://github.com/GiuseppeSp/n8n-customer-interview-synthesizer).
- Next session: Phase 3 (judge + HITL + save path)

### 2026-05-16 (session 1)
- Researched the n8n AI build landscape
- Surveyed templates, blog posts, community projects, n8n's own AI Trust direction
- Considered alternatives: trendy Reddit-scraper / market-analysis agents, vision-DOM agents, polished Figma frontends, local-LLM showpieces. Rejected each (wrong signal for the intended portfolio piece).
- Picked Option A (trust-layer-first) over Option B (Agent Reliability Stress Lab): smaller delta from initial scope, similar payoff.
