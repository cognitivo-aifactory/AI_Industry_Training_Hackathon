# TEAM PLAN

Caveman plan. No fluff. Follow order top to bottom.

## Groups — who do what

| Group | Name | Job |
|---|---|---|
| **G1** | TRAIN | Fine-tune Nemotron. Prep data, train, export, serve adapter. Prove FT better than base. |
| **G2** | BRAIN | Build agent. Qwen tool-calling loop, runtime, `/health` `/query` API, concurrency. |
| **G3** | DATA | Build `query_data` tools (RBA/ASX), AFR search/retrieval, test vs public questions, package submission. |

One group can't finish without other two. Talk to each other. Share `agent/config.py` env vars early so nobody blocks.

---

## Phase 0 — Setup (all groups, day 1 morning)

```bash
# everyone: get on cluster, source team env
source ~/team.env
cd ~/Cognitivo_Training/finagent-finetune
bash scripts/02_smoke_test.sh   # ~30s. confirms GPU + data path + checkpoint save work
```

No smoke test pass, no proceed. Fix cluster first.

---

## Phase 1 — Parallel build (day 1)

### G3 — data tools first (BRAIN needs this to test)

```bash
# build query_data(dataset, metric, ...) — single entry point, no shortcuts
```
- `rba`: `count`, `count_changes`, `count_increases`, `count_decreases`, `extremes`, `max_hold_streak`, `lookup_rate`, `list`
- `asx`: `annual_return`, `rank_annual_returns`, `full_sample_return`, `volatility`, `correlation`, `max_drawdown`
- `afr`: `count`, `count_by_month`, `share` — all need `pattern=` regex

Rules, no skip:
- AFR pattern match = `HEADLINE+SUBHEAD+INTRO+TEXT` combined, case-insensitive, once per record.
- Word boundary always: `\bNAB\b` not `NAB`. Bare word = overcounts, fail grading.
- RBA `lookup_rate(date_from=X)` = rate on-or-before X. Never nearest date, never future.
- ASX basket questions = exclude `TAH.AX` unless told otherwise.
- Do not touch raw dataset files. Read only.

Also: stand up AFR retrieval (local search / Qdrant) for article-grounded sentiment questions.

Test each metric against `Participant_Package/public_questions.jsonl` — answers there are ground truth.

### G2 — agent skeleton (needs G3 tools to test loop, build in parallel with mocks first)

```bash
# brain/agent node
litellm --config litellm_config.yaml --port 4000
```
```yaml
model_list:
  - model_name: agent-brain
    litellm_params:
      model: openai/Qwen/Qwen3.6-35B-A3B-FP8
      api_base: http://localhost:8000/v1
  - model_name: domain-ft
    litellm_params:
      model: openai/nemotron-8b-finance-lora
      api_base: http://<ft-node-ip>:8001/v1   # fill in real IP, no localhost
```

```bash
# src/ — agent server
uvicorn agent:app --host 0.0.0.0 --port 5000
```
- `GET /health` → `200 {"status":"ok"}`. Hard gate. Must work from OTHER machine.
- `POST /query` → `{"question": "..."}` in, `{"answer": ..., "steps": N, "tool_trace": [...]}` out.
- Loop: Qwen plans → emits tool_calls → runtime executes `query_data`/`retrieve` → result back to Qwen → repeat till no more tool_calls (cap ~10 iterations, aim ≤3).
- Handle 3 concurrent `/query` calls. No shared mutable state across requests. Test it, don't assume.
- Start with `DOMAIN_PREDICT_MODE=mock` — fine for now, G1 not done yet.

### G1 — fine-tune Nemotron (own node, runs in background rest of day)

**Data cleaning — already done, scripts live in `src/data_prep/`:**

```bash
# optional: stage a local raw copy from 'data set/' into data/raw/ (never touches source)
python src/data_prep/copy_raw_data.py

# clean RBA/ASX/AFR into a training-only corpus at training/data/clean/
python src/data_prep/clean_datasets.py
```

Already validated: RBA 175 rows, ASX 31,932 rows/18 files, AFR 219,538 rows/85 files. QBE 2021
whole-word count checked raw vs cleaned vs reference (369/369/369) — cleaning didn't shift counts.

What the script does (per Sonali_plan.md, good ideas, 3 fixes applied):

- Cleans into `training/data/clean/` only. Never touches `data set/` originals. G3's runtime
  `query_data` reads the RAW files, always, full float precision. Cleaned/rounded copy is
  training-only, never wired into the live agent.
- ASX: rounds prices to 2dp for training tokens. **Not applied in the runtime tool** — grading
  tolerance on closes is ±0.0001, rounding to cents blows past that on computed
  returns/drawdowns.
- RBA: `change_bps = change_pct * 100`, not `* 10000` (her original formula's math didn't match
  her own example — 0.25 → 25bps only works with ×100).
- AFR: unicode-normalizes / strips soft hyphens (`\xad`) for retrieval index. Validated this
  doesn't shift exact pattern-match counts before trusting it (see above).
- Output is gitignored (`/data/`, `/training/data/` — 1.5GB+ combined). Regenerate locally,
  don't commit it. Only the scripts are tracked.
- Train on synthetic Q&A pairs shaped like the public examples (her ASX/RBA/AFR pair templates
  are good), not the 15 public questions verbatim — those are for calibration/testing, not
  training data. Same spirit as "no question-ID hardcoding" rule.
- Otherwise her plan is right: mix instruction+context+question→answer, 35/25/20/20 split
  (mixed/ASX/RBA/AFR), teach synthesis not memorization. Good split, keep it.

```bash
tmux new-session -s train8b "bash scripts/07_train_8b_quicktest.sh"
tail -f /tmp/nemo_8b_test.log
```
Baseline config, don't fight it unless you know why:
```env
LR=5e-5          # 1e-4 spikes loss at step 50, don't
MAX_SEQ_LEN=512  # OOM above this
LORA_RANK=32
NEMO_IMAGE=nvcr.io/nvidia/nemo:25.09   # 25.04 crashes on GB10
CHECKPOINT_EVERY=20
```
Always inside `tmux` — earlyoom kills untracked jobs. No checkpoint before step 20 = crash before that means start over.

Check step-20 checkpoint before waiting for full 100 — it already shows big improvement. Don't burn the whole window on steps you don't need.

---

## Phase 2 — Integration (day 1 evening / day 2 morning)

1. **G1** exports adapter, serves it:
   ```bash
   find "$MODELS_DIR/checkpoints" -type d -name hf_adapter
   ADAPTER_CHECKPOINT=".../hf_adapter" bash scripts/04_export_and_serve.sh
   # vLLM up on :8001, fine-tuning node
   ```
2. **G2** points `domain-ft` LiteLLM route at that endpoint, wires synthesis step:
   Qwen loop finishes → send question + verified tool results to Nemotron → Nemotron writes final `answer`.
3. **G2** flips the switch:
   ```bash
   export DOMAIN_PREDICT_MODE=llm   # mock is dead now, no going back
   ```
4. **G3** runs all 15 `public_questions.jsonl` through the live `/query` endpoint. Check each `grading.components[].expected_fact` by hand. Log mismatches, hand back to G2/G1.

Nobody moves to Phase 3 till full pipeline (Qwen → tools → Nemotron) answers a public question correctly end to end.

---

## Phase 3 — Prove the fine-tune (G1, parallel with Phase 2 wrap-up)

```bash
# base vs fine-tuned, same prompts, same tool results, side by side
```
- Run held-out/val examples through base Nemotron and FT Nemotron.
- Record before/after in `training/` — logs, metrics, a short comparison table.
- No cherry picking. Show failures too if honest comparison needs it.

---

## Phase 4 — Harden and time it (G2 + G3, day 2)

- Every question ≤60s or lose 20% of points. >300s = zero. Budget ≤3 tool calls per question.
- Fire 3 concurrent requests at `/query`, confirm no mixed-up answers.
- For unsupported cross-dataset questions (e.g. asking about 2022-2023 when AFR/ASX end 2021): answer "no, evidence doesn't cover this" — don't invent numbers.
- `answer` must state every requested component plainly. No hedge words ("approximately", "roughly") — judge rejects them.

---

## Phase 5 — Package and submit (all groups, final hours)

```text
src/         <- G2's agent + G3's tools + G1's data_prep scripts, .gitkeep gone
training/    <- G1's training scripts, config, logs, model comparison
logs/        <- non-sensitive run logs/screenshots
```

- [ ] `submission.json` at root: real `team_id`, `team_name`, `github_url`, 40-char `commit_sha`, real agent IP:port (no `localhost`), model endpoint + name.
- [ ] `README.md`: architecture (Qwen→runtime→Nemotron), run command, training summary, base-vs-FT results, known limitations.
- [ ] `GET /health` verified 200 from a DIFFERENT machine.
- [ ] No secrets, no API keys, no hidden eval data anywhere in the repo.
- [ ] Repo is public. Push. Confirm URL loads in incognito.
- [ ] Double check `DOMAIN_PREDICT_MODE=llm`, not `mock`. This kills your model-quality score if forgotten.

Done. Submit commit SHA. Stop touching main branch after that.
