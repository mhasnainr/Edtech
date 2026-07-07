# AI Admissions Eligibility Platform (Pre-MVP)

**Status:** Actively in development · Solo-built · Phase 1 of a 6-phase build

A trust-first admissions-intelligence system for international (Pakistan-outbound) students applying to UK and German master's programs. Two things it does, and one thing it deliberately refuses to do:

- Answers questions about universities, cited to official sources (free tier).
- Gives a **deterministic eligibility verdict** — the candidate's grades, checked against curated, sourced admission rules.
- **Never lets an LLM decide a verdict.** Every "eligible" / "not yet" / "can't tell" answer comes from auditable logic, not model inference.

> This repo documents architecture and engineering decisions for evaluation purposes. Application code, admission-rule data, and business strategy are kept private during pre-MVP development.

---

## The core engineering problem

Most eligibility tools either guess (an LLM improvises an answer) or oversimplify (a single "you need X%" rule that doesn't match how universities actually admit). Real admission rules are messier — some universities publish no grade cutoff at all and use a holistic committee review; some score applicants on a points scale with an interview band; some publish a clean class threshold. The system has to represent all three honestly, and it has to be **impossible to accidentally tell a student they're eligible when the system can't actually confirm that.**

## Architecture

Two data paths on one Postgres (Supabase) database, deliberately kept independent:

```
Path A — Eligibility Engine (trust-critical)          Path B — Free Q&A (RAG)
Curated, sourced rules → deterministic verdict         Scraped content → embeddings (pgvector)
NO LLM in this path                                    → retrieval → cited LLM answer
```

Path A never depends on Path B's freshness — a stale scrape can degrade the chatbot, but it can never produce a wrong eligibility verdict.

## Engineering decisions worth highlighting

**The gate-vs-criteria firewall is enforced by the database, not application code.** Every admission rule is classified as either a `deterministic_gate` (a real, checkable threshold — e.g. a minimum degree classification) or `criteria_only` (qualitative — e.g. "a related field," an aptitude assessment). A database check constraint physically prevents a qualitative rule from being inserted with a threshold, and vice versa. The engine can only ever compute a verdict on a gate; a criterion is always surfaced, cited, and never blocks or passes anyone.

**The output has four honest states, with a strict precedence.**

| State | Meaning |
|---|---|
| `eligible_to_apply` | All gates evaluated and passed. Explicitly *not* an admission guarantee. |
| `not_yet_eligible` | A gate failed — with the specific, fixable next step. |
| `indeterminate` | A gate exists but can't be evaluated yet (e.g. a grade-conversion mapping isn't sourced). |
| `no_deterministic_gates` | The program has no fixed cutoff — a cited summary and referral, never a fabricated verdict. |

**Precedence: fail beats indeterminate beats pass.** If any part of the evaluation is uncertain, the result can never be reported as "eligible." This is enforced structurally — an unrecognized input, a missing grade-conversion mapping, or any edge case resolves to *indeterminate*, never to a silent pass.

**Regression testing is weighted toward the worst failure mode.** The eval harness runs every test case against the engine and explicitly flags a `false_eligible_breach` — the scenario where the system would have told someone they're eligible when they're not. Every rule change must pass the harness at zero breaches before being trusted.

**Every stored rule carries its source.** `source_url` and `source_date` are non-nullable on every admission-rule table — a rule without a citation cannot be inserted. Unsourced figures (e.g. a grade-conversion cutoff not yet published anywhere official) are tracked in a running backlog and deliberately left out of the system rather than estimated.

**Row-level security is enabled on every table**, deny-all by default, before any public access is designed.

## Tech stack

| Layer | Tool | Role |
|---|---|---|
| Database | Supabase (Postgres + pgvector) | Structured rules + vector corpus, one DB, two isolated paths |
| Eligibility engine | Postgres functions (PL/pgSQL) | Deterministic, stateless, read-only, no network calls |
| Orchestration (planned) | n8n | Scraping and pipeline automation (Phase 2) |
| Frontend (planned) | Lovable | Free-tier UI (Phase 4) |
| Build environment | Claude Code | Agentic development — every schema change and function is proposed as a plan, reviewed, then executed with explicit approval |

## Development workflow

Built solo using an AI pair-programming workflow with hard guardrails rather than autonomous execution:

- Every risky change (schema migration, engine logic) is proposed as a plan and reviewed before any write happens — no unsupervised auto-apply on trust-critical work.
- Domain knowledge is captured as reusable, versioned skill modules (e.g. an "eligibility curation" procedure) rather than re-derived per task, so later work compounds on earlier decisions instead of repeating them.
- A running `needs_review.md` backlog tracks every deliberately-withheld value (e.g. an unsourced grade conversion) so nothing silently becomes a guess.

## Current progress

- ✅ Phase 0 — Schema live: 11 tables, RLS enabled, both data paths represented.
- 🟡 Phase 1 (in progress) — Deterministic eligibility engine built and passing its regression harness (0 false-eligible breaches). Rules curated across all three real-world admission archetypes (no-cutoff, points-based, and clean-threshold). Expert sign-off on the rule set is the remaining exit item.
- ⬜ Phase 2 — Automated content ingestion for the free Q&A tier.
- ⬜ Phase 3 — Guardrails and evaluation for the RAG path.
- ⬜ Phase 4 — Public frontend.
- ⬜ Phase 5 — Launch.

## Demo Video

https://github.com/user-attachments/assets/83cb0186-602c-4250-bef7-323bda4b7053
