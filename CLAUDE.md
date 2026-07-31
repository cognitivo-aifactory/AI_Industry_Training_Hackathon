# CLAUDE.md

Guidance for Claude Code when working in this repository.

## Communication style

Default to caveman style: minimal words, direct, no fluff, no long explanations, no
pleasantries, no hedging. Short blunt sentences. Get to point fast. This apply to all
replies in this repo unless user say otherwise for a specific request.

## What this repo is

Team submission for the **Cognitivo Hackathon**: build and fine-tune an evidence-grounded
financial-market Q&A agent over RBA, ASX, and AFR data. The repo root doubles as the
`TeamSubmission/` structure the organizers expect — do not restructure it.

Current repo state (2026-07-31): `src/`, `training/`, `logs/` contain only `.gitkeep`
placeholders. `submission.json` at root still has placeholder values (`mock-team`,
example GitHub URL, `172.20.x.x` IPs, fake commit SHA). All of this must be filled in
before submission — see [Before submitting](#before-submitting).

## Objective

Answer unseen financial-market questions (easy/medium/hard, single- and cross-dataset)
through a working HTTP agent. Must show:
- Measurable Nemotron fine-tuning quality vs the supplied base model.
- A reproducible, well-engineered agent architecture.
- Grounded, component-accurate answers on hidden questions.

## Required architecture (non-negotiable model roles)

```
question
  -> Qwen3.6-35B-A3B-FP8 ("agent-brain" via LiteLLM) plans + emits tool calls
  -> agent runtime (application code) validates + executes query_data/retrieve calls
  -> results loop back to Qwen until reasoning is complete
  -> fine-tuned Nemotron ("domain-ft") synthesizes the final `answer`
  -> POST /query returns {"answer": "...", "steps": N, "tool_trace": [...]}
```

- **Qwen3.6-35B-A3B-FP8** (`BRAIN_MODEL=agent-brain`): planning, tool selection, tool-call args. Never
  fine-tuned. Never replaced by Nemotron.
- **Agent runtime**: the only thing that actually touches data. Executes Qwen's requested
  tool calls, returns structured results.
- **Fine-tuned Nemotron** (`DOMAIN_FT_MODEL=domain-ft`, base model
  `Llama-3.1-Nemotron-Nano-8B-v1`): receives question + verified tool results, writes the
  final answer. This is the ONLY model participants fine-tune.

Anti-patterns explicitly called out as scoring zero on architecture: Nemotron as
tool-caller/planner, RAG-only single-model pipelines, base (non-fine-tuned) Nemotron doing
synthesis at submission time.

`DOMAIN_PREDICT_MODE` starts as `mock` (bootstrap default) — **must be switched to `llm`**
before evaluation or the fine-tuned model isn't actually used and both model-quality and
architecture credit are lost.

## Datasets (do not alter source data)

| Dataset | Location | Fields | Notes |
|---|---|---|---|
| RBA | `data set/RBA Rates/RBA-rates.{csv,jsonl}` | `Effective Date`, `Change % points`, `Cash rate target%` | UTF-8 BOM. 175 decision records, 2010-2026. Use deterministic parsing, not retrieval. |
| ASX | `data set/ASX/*.jsonl` (18 tickers) | `ticker, date, open, high, low, close, volume` | 2015-01-02 to 2021-12-30, 1,774 rows/ticker. |
| AFR | `data set/AFR/*.jsonl` (monthly files) | `HEADLINE, SUBHEAD, INTRO, TEXT, NEWSPAPER, PUBLICATIONDATE` | News corpus 2015-2021. |

Tool interface (single entry point): `query_data(dataset, metric, ...)`.

| Dataset | Key metrics |
|---|---|
| `rba` | `count`, `count_changes`, `count_increases`, `count_decreases`, `extremes`, `max_hold_streak`, `lookup_rate`, `list` |
| `asx` | `annual_return`, `rank_annual_returns`, `full_sample_return`, `volatility`, `correlation`, `max_drawdown` |
| `afr` | `count`, `count_by_month`, `share` (all require `pattern=`, a Python regex) |

### Hard rules that break reproducibility if ignored

- **AFR search**: match case-insensitively across `HEADLINE + SUBHEAD + INTRO + TEXT`
  combined, once per record (a hit in multiple fields still counts once). Always use
  word-boundary regex (`\bNAB\b`), never bare substrings — bare acronyms wildly overcount.
- **RBA date lookups**: `lookup_rate(date_from=<date>)` must return the rate in force
  **on or before** the given date, never the nearest date (which could be in the future).
  Date precision matters for grading — e.g. the correct "first effective date" for a rate
  can differ by one day from what looks obvious.
- **ASX basket questions**: unless told otherwise, **exclude Tabcorp** (`TAH.AX`) — most
  "non-Tabcorp basket" questions require `exclude_tickers=["TAH.AX"]`; forgetting it
  changes rankings and returns.
- **Cross-dataset questions**: respect actual date overlap. RBA covers 2010-2026; AFR and
  ASX both end in 2021. If asked about 2022-2023 market reaction, the correct answer is to
  say the evidence doesn't support it — not to invent numbers (see MHQ090 in
  `public_questions.jsonl`).
- Use structured parsing/deterministic calculation for RBA and ASX. Use local
  search/retrieval/RAG for AFR. Never use retrieval to answer a counting/calculation
  question — `retrieve()` over news prose cannot produce exact counts.

## Agent API contract

- `GET /health` -> `200 {"status": "ok"}`. Hard gate — fails this, team gets skipped
  entirely for the run.
- `POST /query` with `{"question": "..."}` -> must return JSON with required `answer`
  (string). `steps` (int) and `tool_trace` (list of `{tool, args, result}`) are optional,
  private-diagnostics only, not scored.
- Must safely handle **3 concurrent `/query` requests** without mixing state — this
  includes the model-serving stack, not just the web layer.
- Target **≤60s** response time (>60s and ≤300s = 20% point penalty on that question;
  >300s = timeout = zero). Aim for ≤3 tool calls per question; avoid `list` on large
  datasets.
- Malformed/missing `answer` = zero for that question.

## Official scoring

```
final_score = fine_tuned_model_score*0.30 + architecture_repository_score*0.30 + hidden_question_score*0.40
```

- **Fine-tuned model quality (30%)**: training data prep, config/hyperparams, base-vs-FT
  comparison on held-out data, robustness, evidence the agent actually routes through
  Qwen→tools→Nemotron.
- **Architecture & repo quality (30%)**: correct role separation, code quality,
  reproducibility, docs, training artifacts, logs, no secrets.
- **Hidden-question evaluation (40%)**: component-based partial credit, graded only on
  `answer` by an independent Qwen-based LLM judge checking each `expected_fact` YES/NO.
  Equivalent formatting accepted (comma/no-comma numbers, equivalent date formats,
  sentiment synonyms preserving meaning); hedging ("approximately", "roughly") and
  correct-answer-buried-in-noise are NOT accepted.

`public_questions.jsonl` (15 calibration cases) shows the exact grading shape —
`grading.components[].expected_fact` is literally what the judge checks. Use it to test
the pipeline; never hardcode answers by question ID.

## Fine-tuning (Nemotron)

- Base model: `Llama-3.1-Nemotron-Nano-8B-v1`, served via vLLM on the fine-tuning/model
  node's port 8001, LoRA adapter loaded at runtime (no merge needed).
- Reference baseline config: LoRA rank 32, batch 2 / grad-accum 4 (effective 8), LR `5e-5`
  (NOT `1e-4`, causes loss spike at step 50), max seq len 512 (OOMs above this on one
  node), warmup 50 steps, NeMo container `nvcr.io/nvidia/nemo:25.09+` (25.04 crashes on
  GB10), checkpoint every 20 steps.
- Step-20 checkpoint already shows meaningful improvement (confirmed: +110% composite
  improvement vs base on 50 test samples in organizer's run) — evaluate early, don't
  assume you need the full 100 steps.
- Full 100-step run ≈ 2-3 hours on one GB10 node. Always run inside `tmux` (earlyoom will
  kill untracked training). Nothing checkpoints before step 20 — a crash before that means
  restart from scratch.
- Training data prep script expects the raw dataset dirs and emits
  `data/{train,val,test}.jsonl` (48k/6k/6k) plus a 500-sample smoke subset.

## Infrastructure

Two-node GIGABYTE Atom cluster, one NVIDIA GB10 each (organizer-assigned hostnames/IPs —
never hardcode example IPs):

| Node | Role |
|---|---|
| Brain/agent node | LiteLLM proxy :4000, Qwen `agent-brain` serving, agent web server :5000 |
| Fine-tuning/model node | Nemotron training, fine-tuned vLLM serving :8001 |

Key env vars (`agent/config.py`): `LITELLM_BASE_URL`, `LITELLM_KEY`, `BRAIN_MODEL`,
`DOMAIN_FT_MODEL`, `DOMAIN_PREDICT_MODE` (`mock`→`llm`), `EMBED_MODEL`, `QDRANT_URL`,
`QDRANT_COLLECTION`, `MAX_AGENT_STEPS`. Keep all endpoints/credentials in env vars, never
hardcoded, never `localhost` in `submission.json` (organizer harness runs on a different
machine — use the assigned IP).

## Before submitting

- [ ] Fill in real `submission.json` (team id/name, actual GitHub URL, real 40-char commit
      SHA, real agent IP:port, real model endpoint) — root file, not the `_template.json`.
- [ ] Populate `src/` (agent+tools), `training/` (fine-tuning evidence), `logs/`
      (non-sensitive run logs); remove the `.gitkeep` placeholders once real files land.
- [ ] `DOMAIN_PREDICT_MODE=llm` (not `mock`) before evaluation.
- [ ] `GET /health` returns 200 from a machine other than the one running the agent.
- [ ] `POST /query` handles 3 concurrent requests correctly.
- [ ] README documents the full Qwen→runtime→Nemotron architecture, run command, training
      summary, base-vs-FT comparison, and known limitations.
- [ ] No credentials, secrets, or hidden evaluation data committed anywhere.
- [ ] Repo stays fully public (no private repo / collaborator-only access supported).

## Key reference docs (read before implementing)

- [Participant_Package/Challenge_Brief.md](Participant_Package/Challenge_Brief.md) — objective, model roles, scoring, worked good/bad answer examples.
- [Participant_Package/Setup_Instructions.md](Participant_Package/Setup_Instructions.md) — dataset schemas, AFR search rules, model serving endpoints.
- [Participant_Package/submission-guide.md](Participant_Package/submission-guide.md) — exact repo layout, `submission.json` schema, README requirements.
- [Participant_Package/handout/01_training_guide.md](Participant_Package/handout/01_training_guide.md) — fine-tuning timings, baseline config, step-by-step workflow.
- [Participant_Package/handout/02_execution_guide.md](Participant_Package/handout/02_execution_guide.md) — agent architecture, LiteLLM config, eval harness usage.
- [Participant_Package/handout/03_scoring_and_examples.md](Participant_Package/handout/03_scoring_and_examples.md) — real score progression, good vs bad architecture diagrams, implementation checklist.
- [Participant_Package/public_questions.jsonl](Participant_Package/public_questions.jsonl) — 15 calibration Q&A pairs with full grading rubric, use for testing.
