When preparing your fine-tuning strategy for Llama-3.1-Nemotron-Nano-8B-v1, the short answer is yes, you should use data from all three datasets, but you should not fine-tune the model to memorize them.

Because Nemotron's role in your LangGraph agent is grounded answer synthesis (taking retrieved context and writing the final answer), the objective of fine-tuning is to train the model how to reason over and synthesize the specific structures found in those three datasets.

Here is a breakdown of why this distinction matters and how to structure your fine-tuning data across the three sources.

## Why You Mix All Three (But Don't Memorize)

If you only fine-tuned on one dataset (e.g., AFR news), the model might struggle to read structured ASX price tables. If you only fine-tuned on ASX tables, it might lose its ability to synthesize unstructured news paragraphs.

Rather than trying to bake the raw data into the model's weights, you want to fine-tune Nemotron to process the outputs of your tools. Your training data should consist of variations of:

\text{Instruction} + \text{Retrieved Context (from RBA/ASX/AFR)} + \text{Question} \rightarrow \text{Grounded Answer}

## How to Prepare Fine-Tuning Data for Each Dataset

To ensure the model synthesizes answers accurately, your training examples should reflect the specific characteristics of each dataset:

### 1. ASX Company Prices (Structured / Tabular Synthesis)

ASX data is highly structured (dates, ticker symbols, open/close/volume numbers).

- **The Challenge**: Smaller models often struggle to compute differences between numbers or accurately pull data points from raw text tables.
- **Fine-Tuning Focus**: Teach the model to read raw CSV or JSON-formatted stock prices and extract the exact requested figure without hallucinating nearby numbers.
- **Example Training Pair**:
  - Input (Context): `[ASX Tool Output]: Ticker: CBA, Date: 2024-10-15, Open: 120.50, Close: 119.20, Volume: 1,200,000`
  - Question: "What was the closing price of CBA on October 15, 2024?"
  - Target Output: "On October 15, 2024, CBA closed at $119.20."

### 2. RBA Cash-Rate Decisions (Macroeconomic / Temporal Logic)

RBA datasets often contain dates, basis point changes, and macroeconomic statements.

- **The Challenge**: Questions might ask for comparisons over time (e.g., "How many times did the RBA raise rates in 2023?").
- **Fine-Tuning Focus**: Teach the model to parse lists of historical decisions chronologically and synthesize rates accurately.
- **Example Training Pair**:
  - Input (Context): `[RBA Tool Output]: Nov 2023: Raised to 4.35%. Dec 2023: Held at 4.35%. Feb 2024: Held at 4.35%.`
  - Question: "Did the RBA change the cash rate in the first meeting of 2024?"
  - Target Output: "No, the RBA held the cash rate at 4.35% during its February 2024 meeting."

### 3. AFR News Corpus (Unstructured Text / Noise Filtering)

The AFR news corpus is unstructured and may contain noise, opinions, or irrelevant market commentary.

- **The Challenge**: The model needs to ignore irrelevant information and extract only facts relevant to the prompt.
- **Fine-Tuning Focus**: Teach the model to synthesize long paragraphs of text into concise, factual summaries without introducing external knowledge (hallucinating).
- **Example Training Pair**:
  - Input (Context): `[AFR News Tool Output]: ...Despite concerns over inflation, banking stocks rallied today. Analysts at Goldman Sachs suggested that CBA's recent margin performance was resilient, though some remain cautious about credit growth...`
  - Question: "What did analysts say about CBA's margin performance?"
  - Target Output: "Analysts from Goldman Sachs characterized CBA's margin performance as resilient."

## Suggested Fine-Tuning Dataset Mix

A balanced training set of 1,000 to 5,000 examples is typically sufficient for a synthesis fine-tuning task of this scale. You might structure the mix as follows:

- **35% Mixed Tasks**: Questions that require combining data from two or more sources (e.g., synthesizing an AFR news reaction with an ASX price movement).
- **25% ASX-only Tasks**: Extracting specific values, high/low comparisons, or temporal movements from price logs.
- **20% RBA-only Tasks**: Summarizing rate changes, extracting dates, or parsing policy text.
- **20% AFR-only Tasks**: Summarizing news snippets and performing entity extraction (identifying key figures, companies, or dates).

By exposing Llama-3.1-Nemotron-Nano-8B-v1 to the varied structures of all three datasets during training, you help ensure that it can handle any combination of inputs that the Qwen routing engine provides during the evaluation phase.

---

Here is the consolidated list of preprocessing and cleaning tasks your scripts should execute across the three datasets. These changes are designed to align the data formats, reduce token waste, and improve search index retrieval.

---

### 1. Global / Cross-Dataset Changes (All Datasets)

- **Standardize Date Format**: Convert all dates to ISO-8601 format (`YYYY-MM-DD`). This is the single most important change for enabling chronological reasoning across datasets.
  - *ASX*: Already `YYYY-MM-DD` (keep).
  - *RBA*: Convert `"3 Feb 2010"` to `"2010-02-03"`.
  - *AFR*: Convert `"20150131"` to `"2015-01-31"`.
- **Standardize Schema Keys**: Convert all keys to lowercase `snake_case` to ensure clean parsing in Python and tool execution.

---

### 2. ASX Prices Dataset Script

- **Reduce Price Precision**: Round all floating-point price fields (`open`, `high`, `low`, `close`) to **2 decimal places** (standard currency format).
  - *Example*: `7.627206538008646` → `7.63`.
- **Extract Clean Tickers**: Store both the original Yahoo-style ticker and a cleaned, standard ticker to help map to names in the news corpus.
  - *Example*: Add a `"clean_ticker"` field (e.g., `"AGL.AX"` → `"AGL"`).
- **Verify Numerical Types**: Ensure price values are floats and `volume` is an integer, rather than strings.

---

### 3. RBA Cash Rate Decisions Script

- **Parse Stringified Numbers**: Convert string representations of percentages to floats.
  - *Example*: `"3.75"` → `3.75` (float).
- **Calculate Basis Points (Bps)**: Add a derived integer field representing the rate change in basis points. This maps directly to how the AFR writes about rate hikes/cuts.
  - *Formula*: `change_pct × 10000` (e.g., `0.25` change → `25` bps, `0.00` change → `0` bps).
- **Normalize Keys**: Remove special characters and spaces from the keys.
  - *Before*: `"Cash rate target%"` → *After*: `"cash_rate_target"`.
  - *Before*: `"Change % points"` → *After*: `"change_pct"`.

---

### 4. AFR News Corpus Script

- **Unicode Normalization**: Apply `unicodedata.normalize("NFKD", text)` to resolve inconsistent spacing formats and accents.
- **Remove Soft Hyphens**: Strip out the invisible soft hyphen characters (`\xad`) which break exact keyword searches (BM25) and tokenizer parsing.
- **Normalize Whitespace**: Replace non-breaking spaces (`\xa0`) with standard spaces (`" "`), and collapse double spaces or weird newline sequences (`\n\n\n`) into standard double newlines (`\n\n`).
- **Format Dates**: Convert `"PUBLICATIONDATE"` strings into standard hyphenated dates.
  - *Example*: `"20150131"` → `"2015-01-31"`.

---

### Summary Checklist for Script Verification

| Dataset | Field | Source Format | Target Format | Reason |
| :--- | :--- | :--- | :--- | :--- |
| **Global** | Keys | Mixed (`HEADLINE`, `Effective Date`) | `snake_case` | Code standardization |
| **ASX** | Prices | 15-decimal floats | 2-decimal floats | Token efficiency, easier comparisons |
| **ASX** | Ticker | `"AGL.AX"` | Add `"clean_ticker": "AGL"` | Alignment with news mentions |
| **RBA** | Date | `"3 Feb 2010"` | `"2010-02-03"` | Date matching |
| **RBA** | Rates | `"3.75"` (string) | `3.75` (float) | Mathematical alignment |
| **RBA** | Change | `"0.25"` (string) | Add `"change_bps": 25` | Matching AFR terminology |
| **AFR** | Date | `"20150131"` (string) | `"2015-01-31"` | Date matching |
| **AFR** | Text | raw HTML/newspaper crawls | Normalized Unicode (no `\xad`, `\xa0`) | Search index precision |

---

> **Claude's review (see TEAM_PLAN.md, G1 section, for the full writeup):** solid overall strategy, 3 things to fix before implementing —
> 1. Bps formula above is wrong: `change_pct × 10000` gives 2500 for a 0.25 move, not 25. Use `× 100` (matches her own worked example).
> 2. ASX 2-decimal rounding is fine for the training corpus only — never apply it in the runtime `query_data` tool, grading tolerance on closes is ±0.0001.
> 3. AFR unicode/soft-hyphen cleanup can shift exact pattern-match counts. Validate cleaned counts against `public_questions.jsonl` before trusting it for `count`/`count_by_month`/`share` — already checked in `src/data_prep/clean_datasets.py` (QBE 2021 count: raw=369, clean=369, matches reference).
