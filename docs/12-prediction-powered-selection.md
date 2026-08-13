# 12 — Scaling the selection number without scaling the annotation

**Question.** The headline selection result rests on 48 human-annotated items.
Can a judge supply the rest without importing its own bias into the number?

**Answer.** The instrument is good enough (κ = 0.662, newly measured). The
estimator is implemented. The *data* is not eligible, for a reason that is
measurable and fixable without any further human work.

---

## What was already on disk

Two labelling passes existed and had never been compared to each other:

| set | items | candidates each | labels |
|---|---:|---:|---|
| `annotation/` | 48 | 8 | one human annotator, plus a Sonnet judge |
| `annotation-big/` | 165 | 6 | a Sonnet judge only |

The 48-item pass had a recorded judge agreement of κ = 0.552. The 165-item pass —
by far the larger, and the one that would have to carry any scale-up — had never
been scored against a person at all.

## The instrument is better than recorded

Measuring the 165-item judge run against the human labels, on the pairs both
passes covered:

```
questions 42, pairs 107
kappa = 0.662   both-yes=11  judge-only=7  human-only=2  both-no=87
judge says yes 18x, human 13x — permissiveness 1.38x
```

κ = 0.662, above the 0.6 bar usually asked of a judge before its labels are used
for anything, and above the 0.552 previously recorded for the same model on the
8-candidate protocol. The likely cause is prosaic: six candidates is an easier
discrimination than eight. Permissiveness is 1.38×, against 2.9× for the on-prem
8B VLM on the same task.

So the judge is usable. That was the open question, and it is now closed.

## Why the estimate is still refused

Prediction-powered inference corrects a cheap, biased estimator using the subset
that carries both labels:

```
theta_ppi = theta(judge, all) - [ theta(judge, L) - theta(human, L) ]
```

The bracket is unbiased only if `L` is a **random subsample** of the population.
It is not:

```
positive rate, full human protocol : 19/384 = 4.9%
positive rate, pairs used here     : 13/107 = 12.1%
enrichment                         : 2.46x
```

The cause is structural, not accidental. The two passes drew different candidate
pools, so `L` is their intersection — and **every harness-gold figure is in both
pools by construction, 38 of 38**, because both draws start from the gold and pad
with distractors. The intersection is therefore enriched in exactly the pairs a
person is likely to mark positive.

Running the estimator anyway produces intervals that do not narrow (−4% to +7%
against human-only) and point estimates displaced downward. Both are symptoms of
the same violation, and neither should be reported as a result.

`ppi-eval.ts` computes the enrichment and **refuses to print estimates** when it
exceeds 1.25×. The numbers are reachable behind `IKAT_PPI_TOL`, which exists so
the failure can be inspected, not so it can be worked around.

## What unblocks it

Judge labels on pools drawn the *same way the annotator's were* — a re-export at
8 candidates over the ~165 scored questions, then one judge pass. Cost is a few
dollars of inference and no human time at all.

The blocker is that `ugm3-built`'s figure crops were deleted during a disk
cleanup. 55% of them (680 of 1230 non-decorative figures) survive inside the two
annotation exports, but rebuilding pools from only the surviving crops would bias
the **distractors** instead — the same disease relocated, and harder to see.

So the sequence is: regenerate the crops, re-export at 8 candidates, judge, then
`ppi-eval.ts` passes its own gate and reports.

## What this changes about the paper

Nothing yet, and that is the point. Two things are now true that were not:

1. The judge we would scale with has a measured κ of 0.662 on the protocol we
   would scale, rather than an unmeasured one.
2. The path to a defensible larger-n selection number is **compute-bound, not
   annotation-bound**. That is a materially better position than "we need weeks
   of human labelling", which is what the limitations section says today.

A second annotator on the existing 48 remains the separate, and still
outstanding, requirement: without it there is no inter-annotator bound, so κ =
0.662 cannot be compared against how much two people would disagree.

## Files

- `src/ikat/judge-scale.ts` — builds pools and runs the judge; reuses the human's
  pool verbatim wherever one exists, so the instrument is identical on both
  halves. Currently blocked on the missing crops; `--dry` reports what it would
  send.
- `src/ikat/ppi-eval.ts` — κ, the representativeness gate, and the rectified
  estimator with bootstrap intervals over questions.
