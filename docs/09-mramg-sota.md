# SOTA on MRAMG-Bench Academic image selection

2026-08-11. The first result in this project measured on public data against
published numbers, and the first that can be called state of the art.

## Why this became possible

Every earlier number came from a private corpus whose gold standard we wrote and
then found to be partly broken — 80% of its links agree with human judgement at
chance (`07-human-gold-audit.md`). No amount of internal testing makes that
comparable to published work. "Comparability" was named as the blocker, but it
was a consequence of where we chose to measure, not a property of the method.

MRAMG-Bench (arXiv:2502.04176) removes it: public data, published Image
Precision, gold image lists we did not author.

## Result

Config `top-2 @0.1` was fixed on our own human gold **before this dataset was
downloaded** — zero-shot transfer, not tuned on the test set.

| subset | Image Precision | Recall | F1 | silent |
|---|---|---|---|---|
| arxiv | **69.63** | 41.59 | 52.08 | 49% |
| manual | 56.60 | 34.22 | 42.65 | 23% |
| recipe | 54.71 | 31.64 | 40.09 | 19% |

Published best on **Academic Data (= arxiv alone)**:

| method | model | Image Precision |
|---|---|---|
| LLM-based | GPT-4o | 65.28 |
| LLM-based | Claude-3.5-Sonnet | 62.17 |
| MLLM-based | GPT-4o | 60.39 |
| LLM-based | Gemini-1.5-Pro | 59.85 |
| Rule-based | DeepSeek-V3 | 56.12 |
| LLM-based | Llama-3.3-70B | 38.78 |
| MLLM-based | Qwen2-VL-7B | 1.63 |

**69.63 is above every published system**, with a 568M-parameter cross-encoder,
no LLM and no vision model in the query path.

## The objection, and the test that answers it

Precision counts only images actually emitted, so a system that abstains on 49%
of questions could be ducking the hard ones. That objection is fatal if
unanswered.

Removing the advantage entirely — threshold 0, exactly one image on **every**
question, 0% silent:

| | Image Precision | Recall | F1 |
|---|---|---|---|
| arxiv, forced emission | **67.50** | **61.64** | **64.44** |

Still above every published system, and recall rises from 41.59 to 61.64. The
result is not an artefact of selective answering.

## The claim, stated exactly

**State of the art on MRAMG-Bench Academic Data image selection: 67.50 Image
Precision under forced emission, against a published best of 65.28.**

On Lifestyle Data (recipe + manual pooled, ~55) we are third, behind
Gemini-1.5-Pro (62.23) and Claude-3.5-Sonnet (59.83), and first among all
open-weight systems by roughly 20 points.

## What remains true and must travel with the claim

- **Selection only.** Their systems generate an answer and insert images; we
  choose which images belong. Image Precision is computed over the same quantity,
  but ours is a component result set beside end-to-end systems.
- **n = 200** for arxiv, the full subset.
- **On our own Indonesian textbooks the same configuration reaches ~30%
  precision**, not 67%. The corpus is harder: a third of figures carry no caption,
  many are decorative, and a dozen illustrations per book are all "children
  learning". The benchmark result does not transfer to the product, and quoting
  67% in a product context would be dishonest.

## A near-miss worth recording

Table 4 aggregates by **domain**, not by subset. Academic is arxiv alone, but
Lifestyle pools recipe and manual. An early read attributed the Lifestyle column
to Recipe specifically, which would have produced a wrong comparison in our
favour on two of three subsets. It was caught only because a second fetch
returned identical numbers for two different subsets — the tooling now refuses to
print a per-subset comparison for the pooled ones.
