# Team Management Dashboard — IWS-Adapted Concept Summary

## Core idea

A team management dashboard adapting IWS (issue/pillar/route framing) concepts, built around tasks and issues as first-class objects. The goal is to streamline day-to-day team operations while accumulating a durable, searchable knowledge base of how problems actually got solved.

## Task/issue model

- Tasks and issues each get their own Discord-style channel (threaded chat + media) for discussion tied to that specific item.
- Tagging system, plus IWS-style routes and pillars, for categorization.
- Bite-size sub-actions can be created and assigned under a task/issue, and resolved independently.
- Ownership can be transferred between people.
- Problems can be escalated to higher-responsible parties, or reassigned to other teams entirely.

Open design question: model tasks and issues as one unified `item` entity with a `kind` field (simpler shared logic for ownership/escalation/channels) vs. two separate entities (cleaner IWS-specific semantics, but duplicated workflow logic). Not yet decided.

## Knowledge base

- Purpose: capture team know-how so it isn't lost when a task/issue closes.
- Linking strategy: both **tag-based** (precise, cheap filtering) and **semantic** (embeddings + vector similarity, to surface related know-how that doesn't share exact tags). Running both is worth the added complexity of maintaining a vector index alongside tag columns.
- **Resolve → summarize → save flow**: a button on a resolved task/issue triggers a small/cheap LLM call that summarizes the item and appends it to the knowledge base.
  - Open question: auto-publish the summary immediately, or land it as a draft for human approval/edit before it becomes searchable (auto-publish risks polluting the KB with bad summaries).

## Cost estimate for the summarize-on-resolve call

Assumptions: ~2KB issue text (~500 tokens at ~4 bytes/token) + ~100 tokens system prompt = ~600 input tokens; ~200–300 output tokens per issue.

| Model | Input / Output $ per MTok | Cost per issue | Cost per 1,000 issues (₺, @ ~48.6 TRY/USD) |
|---|---|---|---|
| Claude Haiku 4.5 | $1.00 / $5.00 | ~$0.0019 | ~₺90 |
| GPT-5-nano | $0.05 / $0.40 | ~$0.00013 | ~₺6 |
| GPT-5.6 Luna | $0.20 / $1.20 | ~$0.00042 | ~₺20 |
| Gemini 3.5 Flash-Lite | $0.30 / $2.50 | ~$0.00081 | ~₺39 |
| Gemini 3.8 Flash | $0.75 / $3.75 | ~$0.00139 | ~₺67 |

Non-Claude figures are from third-party pricing aggregators (not providers' own docs) — treat as approximate. At this project's expected scale, per-call cost is negligible across all providers; it only starts to matter at very high issue volume, and shouldn't be the deciding factor over summarization quality/consistency for a KB the team will actually rely on.

## Hosting note

A 6-month, $300 Google Cloud trial credit is available and is more than enough to cover this project's infrastructure: Cloud Run (or a small Compute Engine VM) for hosting, Cloud SQL/AlloyDB with the `pgvector` extension for the tagged + semantic knowledge base, and Vertex AI for embeddings/summarization calls — all billed against the same credit pool. Routing the summarize button through Vertex AI (Gemini) keeps spend inside the free credits; using Claude's API directly would be a separate bill outside the trial.

## Open decisions (not yet made)

1. Unified `item` entity vs. separate `task`/`issue` entities.
2. Auto-publish vs. draft-and-approve for KB summaries.
3. Which model/provider serves the summarize button (cost is a non-factor either way; pick by quality/consistency, and whether staying inside GCP credits matters).
4. GCP-based architecture details (Cloud Run + Cloud SQL/pgvector + Vertex AI) — not yet sketched.
