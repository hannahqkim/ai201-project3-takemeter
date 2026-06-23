# Baseline Results — Zero-Shot Groq (`llama-3.3-70b-versatile`)

Milestone 4. Baseline run on the **locked test set** before any fine-tuning analysis.
These numbers are the bar the fine-tuned model must beat.

## Setup
- **Model:** `llama-3.3-70b-versatile` (Groq), zero-shot, `temperature=0`.
- **Prompt:** the TakeMeter label definitions from `planning.md`, instructing the model to
  output only the label name (`analysis` / `hot_take` / `reaction`).
- **Test set:** 34 examples (deterministic 70/15/15 split, `random_state=42`):
  analysis 16, hot_take 10, reaction 8.

## Results
- **Overall accuracy: 0.824** (28 / 34 correct), **macro-F1 0.82**.
- **Unparseable responses: 0 / 34** — the prompt produced clean label names every time.

### Per-class breakdown

| label | precision | recall | f1 | support |
|---|---|---|---|---|
| analysis | 0.75 | 0.94 | 0.83 | 16 |
| hot_take | 0.88 | 0.70 | 0.78 | 10 |
| reaction | 1.00 | 0.75 | 0.86 | 8 |
| **macro avg** | 0.88 | 0.80 | 0.82 | 34 |

## Reflection — where the baseline struggled (and how the hypotheses held up)

The zero-shot 70B model is strong (0.82 macro-F1) and the results largely confirmed the
pre-run hypotheses:

1. **`reaction` was the baseline's cleanest class** — perfect precision (1.00) and high recall
   (0.75). As predicted, emotional, event-triggered posts ("I'm shaking", an MV drop) are
   lexically obvious enough that a general model nails them without training.
2. **`hot_take` was the baseline's weakest class** (recall 0.70) — it missed 3 of 10 hot
   takes, most plausibly reading confident, well-written takes as `analysis` (analysis recall
   is high at 0.94, so misses leak *into* analysis). This is exactly the `analysis` ↔
   `hot_take` load-bearing-evidence boundary flagged as hardest in `planning.md`.
3. **The model leans toward `analysis` when unsure** (analysis precision only 0.75 despite
   0.94 recall → it over-predicts analysis). That's the same directional bias the fine-tuned
   model shows far more severely.

**This is the bar to beat: 0.824 accuracy / 0.82 macro-F1.** See the README evaluation report
for the head-to-head — the fine-tuned DistilBERT does **not** beat it.
