# AmazonHelp Support Agent

An AI support agent built on the [Customer Support on Twitter](https://www.kaggle.com/datasets/thoughtvector/customer-support-on-twitter) dataset, scoped to the **AmazonHelp** brand. Given an incoming customer tweet, it classifies intent, drafts a reply grounded in how AmazonHelp has historically responded to similar messages, and decides whether to auto-handle or escalate to a human, with a stated reason.

Full pipeline write-up (numbers, findings, caveats): [`pipeline_summary.txt`](pipeline_summary.txt).

## Reproduce this (≈15 minutes, ≈$0.40 in API cost)

**1. Setup**

```bash
python -m venv .venv
.venv\Scripts\activate          # Windows; use `source .venv/bin/activate` on macOS/Linux
pip install -r requirements.txt
```

Create a `.env` file in the project root:

```
OPENAI_API_KEY=sk-...
```

**2. Get the raw data**

Download `twcs.csv` from the [Kaggle dataset](https://www.kaggle.com/datasets/thoughtvector/customer-support-on-twitter) and place it at `Data/twcs.csv` (not included in this repo — 493MB).

**3. Run the notebooks in order**

| notebook | what it does | approx. time | approx. cost |
|---|---|---|---|
| `dataprep.ipynb` | reconstructs conversations from raw tweets, filters to clean AmazonHelp threads | ~2 min | $0 |
| `intent_pipeline.ipynb` | classifies the first 5,000 conversations into 12 intents via `gpt-4o-mini` | ~3 min | ~$0.10 |
| `reply_and_escalation.ipynb` | embeds messages, retrieves similar historical cases, drafts replies, decides escalation | ~5 min | ~$0.25 |
| `eval_harness.ipynb` | samples a 150-row golden set, computes accuracy/escalation metrics, runs an LLM judge against 40 hand-rated drafts | ~2 min | ~$0.02 |

Each notebook reads the previous stage's output from `Data/` and writes its own; run top-to-bottom in Jupyter or via `jupyter nbconvert --to notebook --execute --inplace <notebook>.ipynb`.

## Headline results

- **Intent classification**: 89.3% accuracy against a 150-row hand-labeled golden set (94.3% on confidently-classified messages, 82.3% on the ones below the confidence threshold — see caveats).
- **Escalation policy**: 70.7% agreement with human judgment; errs toward over-escalation (28%) rather than under-escalation (1.3%).
- **Reply drafting**: LLM-judge quality scores agree with a human rater within 1 point 92.5% of the time, but the actual correlation is weak (Spearman ρ = 0.11, not significant) — see `pipeline_summary.txt` for why that matters.

## What's *not* in this repo

- `Data/twcs.csv` and the large intermediate files it produces (`amazonhelp_threads.csv`, the four `*_discard.csv` files) — all regenerable by running `dataprep.ipynb`, excluded to keep the repo small.
- The formal written report (problem framing, baselines, failure analysis, decision log) — this repo is the runnable pipeline and its evaluation artifacts; the report is a separate document.

## Key design decisions (see `pipeline_summary.txt` for full detail)

- Only the customer's **first turn** is classified/acted on — that's the only information available at the moment a real system has to decide what to do, before AmazonHelp has replied.
- No outcome/resolution label exists in the raw data. "How the brand historically resolved similar issues" is approximated by **AmazonHelp's own first reply** to the most similar past message with the same intent — a stated proxy, not verified ground truth.
- Escalation is a **deterministic rule**, not another LLM call, so every auto-handle/escalate decision is cheap, auditable, and comes with an explicit reason.
- Intent classification scope is the first 5,000 conversations (chronologically oldest by `conv_id`) — a deliberate subsample per the assignment's own guidance, not a claim of full-corpus coverage.
