# 13 — Every MRAMG subset, including the three that measure nothing

**Question.** The paper reported one MRAMG subset out of six. Reporting one reads
as selection whether or not it was. What do the other five say?

**Answer.** One win, one loss, and three subsets on which no claim is possible —
because under our candidate construction they contain no selection decision at
all. Running them turned a weak defence ("we ran them all") into a demonstrable
one ("only Academic tests the task, here is the number").

---

## Results

Per domain, which is the unit the published comparison table uses. Web pools
wit+wiki+web; Lifestyle pools recipe+manual. Pooling sums raw tp/fp — averaging
per-subset precision would weight a 200-question subset like a 2360-question one.

| domain | n | ours IP | best published | |
|---|---:|---:|---:|---|
| **Academic** | 200 | **69.12** | 65.28 (GPT-4o) | win, +3.84 |
| Lifestyle | 2714 | 55.00 | 62.23 (Gemini-1.5-Pro) | **loss, −7.23** |
| Web | 1850 | *100.00* | 93.63 (Gemini-1.5-Pro) | **no claim possible** |

Per subset, with the diagnostic that matters:

| subset | n | cands/q | all-gold | IP | IR |
|---|---:|---:|---:|---:|---:|
| arxiv | 200 | 3.4 | **0%** | 69.12 | 41.59 |
| wit | 600 | 1.0 | 100% | *100.00* | 79.50 |
| wiki | 500 | 1.0 | 100% | *100.00* | 65.00 |
| web | 750 | 2.0 | 100% | *100.00* | 45.27 |
| recipe | 2324 | 4.9 | 47.9% | 54.71 | 31.64 |
| manual | 390 | **91.5** | **0%** | 56.69 | 34.34 |

## Why three subsets measure nothing

`all-gold` is the share of questions where *every* candidate is already a gold
image. Where it is 100%, there is no wrong answer available: precision is 1 by
arithmetic the moment anything is emitted, and the only decision left is whether
to emit at all. That is what the recall column is measuring there, and nothing
else.

This is a property of **our** candidate construction — the images of the
question's provenance document — not a defect in MRAMG. Their comparator systems
generate interleaved answers from a different pool. The honest statement is
narrow: under this construction, the Web domain leaves no selection decision.

Reporting 100.00 against a published 93.63 would therefore be a false claim, and
an obvious one. `mramg-eval.ts` computes `all-gold%` itself and prints a warning
above 50%, so the number cannot be quoted without its context.

## The result that was not expected

`manual` carries **91.5 candidates per question** — 27× the arxiv pool — with 0%
degenerate, and still reaches 56.69 precision. `recipe` has 4.9 candidates and
half its questions degenerate, and scores *lower* at 54.71.

So distractor count is not what breaks this method. Something else in `recipe` is
harder. The likely cause is that recipe averages 2.2 gold images per question
while the pipeline emits at most two and abstains on 19%, capping recall at 31.64
and dragging F1 with it. That is a hypothesis, not a measurement.

## What this does to the paper's claim

Nothing, and that is the point: the manuscript already scoped its claim to
"MRAMG-Bench's Academic subset". What changes is the strength of the defence.
Before, one subset out of six with no account of the others. Now, the subset that
is the only one posing the task, with the arithmetic shown for the three that do
not and the loss on Lifestyle stated in the same table.

**Still outstanding:** the +3.84 margin on Academic carries no confidence
interval. On 200 questions and 136 emissions that gap may or may not survive
resampling, and until it is bounded the correct phrasing is "higher", not
"significantly better".

## Reproducing

```bash
MRAMG_SUBSET=all bun bench/mramg-eval.ts 99999   # ~5 h on a 12-core CPU reranker
```

Only the `.jsonl` files are needed — 18 MB. `IMAGE.zip` (1.5 GB) is unused: this
path scores the text standing in for each image.

The arxiv figure reproduced at 69.12 on a CPU backend against 69.63 from the
original GPU run — identical true positives, identical recall (41.59), identical
silent rate, differing by exactly one emitted image where a borderline candidate
falls either side of the 0.1 admission threshold.
