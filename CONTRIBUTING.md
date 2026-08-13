# Contributing

## The contribution that would help most

**Annotate more figure–question pairs.** The headline selection result rests on
48 items. Every other number in the paper is model-free or external; that one is
not, and it is the number a reviewer will push hardest on.

```bash
bun bench/export-annotation.ts    # writes items to $IKAT_ANNDIR
# ... label them ...
bun bench/score-annotation.ts     # agreement + scored output
```

A second annotator on the *existing* 48 is nearly as valuable as new items,
because it puts an inter-annotator agreement figure on the gold itself.

## Ground rules for anything numeric

**Record the configuration with the result.** `IKAT_JUDGE_PROMPT=strict` versus
`loose` moves precision by ~12 points on identical inputs. A number without its
environment is not a result. We withdrew a finding once, then withdrew the
withdrawal, over precisely this.

**Never score a VLM pipeline against VLM-authored gold.** Measured inflation is
about 1.9×. If a change makes the number go up, check `IKAT_GOLD` first.

**Compare rankers by ranking, not by threshold.** Two models can have entirely
incomparable score scales; applying one absolute cutoff to both measures
calibration and calls it quality. Use gold-first rate or MRR. We published a
wrong conclusion about a competing reranker this way before catching it.

**Report the runs that went against you.** The repo has a section for those and
it is not decoration — two of the entries there are our own hypotheses failing.

## Code

- Bun + TypeScript, no build step. `bun bench/<script>.ts`.
- Configuration is environment variables only. Nothing endpoint-specific, and
  no credential, belongs in a source file — see `.env.example` for the surface.
- No corpus content in git. The books are Indonesian government curriculum
  materials distributed under their own terms; this repo holds code and derived
  measurements.

## Pull requests

Say what you measured, on what, with which configuration. A PR that changes a
reported number must update [`RESULTS.md`](RESULTS.md) and the paper together.
