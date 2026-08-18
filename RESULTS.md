# Reproducing every number in the paper

One row per reported result: what it is, which script produces it, and what that
script needs. Nothing in the paper is computed anywhere else.

Configuration is entirely environment variables — see [`.env.example`](.env.example).
The columns below name the variables that *change* the result; everything else
can stay at its default.

## Start here — no GPU, no API key, no model

| paper | what | command |
|---|---|---|
| Table I | caption coverage, anchor coverage, sibling ambiguity, page sharing, on two independent extraction paths | `bun bench/structural-analysis.ts` |

This one reproduces from a cached extraction. It is the claim the rest of the
paper rests on — four figures in five have no caption to match against — and it
is the cheapest to check, so check it first.

## Placement

| paper | what | command |
|---|---|---|
| §IV, PD = 2.08 / 4.69 sentences; exact-placement rate | displacement under a perfect retriever and a perfect answer | `bun bench/run-bench.ts` |
| §IV significance | paired bootstrap (10k) + McNemar exact over placement decisions | `bun bench/placement-significance.ts` |

`placement-metrics.ts` is the library the metrics live in (displacement, PA@k,
Grounded Figure F1), not an entry point — `run-bench.ts` and the significance
scripts import it.

Placement gold comes from the source typesetting, not from annotation, so these
need no human labels and no judge model.

## Selection

| paper | what | command | needs |
|---|---|---|---|
| Table III | VLM filter → cross-encoder rank vs. each stage alone, against human annotation | `bun bench/select-eval.ts [system…]` | `IKAT_GOLD` pointing at human gold; rerank + VLM endpoints |
| Table IV | candidate-cut curve and projected latency | `bun bench/prefilter-pipeline.ts` | `IKAT_PREFILTER_NS`, `IKAT_JUDGE_PROMPT` |
| §V κ validation | judge-vs-human agreement, used to establish the gold is not self-serving | `bun bench/judge-figures.ts validate` | annotation set |
| §V, VLM selector / composed pipeline rows | the same judge in its other two roles | `bun bench/judge-figures.ts select` / `… pipeline` | as above |
| §V significance | McNemar exact over paired selection decisions | `bun bench/selection-significance.ts` | a scored run |
| docs/12 | judge κ on the scale-up protocol, representativeness gate, rectified estimator | `bun bench/ppi-eval.ts` | both annotation exports |
| docs/12 | the scale-up judge pass itself | `bun bench/judge-scale.ts` | figure crops, VLM endpoint |
| docs/12 | rebuilding deleted figure crops from the PDFs | `IKAT_PDF_DIRS=/path bun run crops -- --verify` | source PDFs |

**`IKAT_JUDGE_PROMPT` is part of the configuration, not a detail.** `strict` and
`loose` differ by roughly 12 precision points on the same model and the same
data. A result recorded without it is not reproducible; we lost a day to exactly
that.

**Do not score a VLM pipeline against VLM-authored gold.** It inflates by about
1.9× (0.521 against 0.283 on the same system). `IKAT_GOLD` must point at human
annotation for any headline number.

## External validation

| paper | what | command | needs |
|---|---|---|---|
| Table II | MRAMG-Bench Academic, Image Precision, against published comparators | `MRAMG_SUBSET=arxiv bun bench/mramg-eval.ts 200` | `MRAMG_DIR` |
| Table II, forced-emission row | same, emitting exactly one image on every question | `MRAMG_SUBSET=arxiv IKAT_RERANK_TOP_K=1 IKAT_RERANK_MIN=-1 bun bench/mramg-eval.ts 200` | `MRAMG_DIR` |
| all six subsets | every subset, pooled into the three domains the published table uses | `MRAMG_SUBSET=all bun bench/mramg-eval.ts 99999` | `MRAMG_DIR` |
| §External, MMDocRAG | image-quote selection over the benchmark's own descriptions, per domain | `bun bench/mmdocrag-eval.ts` | `MMDOCRAG_DIR` (the 25 MB `dev_15.jsonl` only) |
| §External, BM25 tie | the no-model floor the ranker must clear (it does not, there) | `python3 bench/mmdocrag-baselines.py` | same |
| §External, vision gate | composed pipeline vs. cross-encoder alone, identical 200-question stride sample | `bun bench/mmdocrag-composed.ts --manifest 200`, judge the manifest, then `--score verdicts.json` | images archive + a VLM judge |
| §External, anchor-window sweep | precision decay as the window grows past the anchor | `MRAMG_SUBSET=arxiv MRAMG_CTX=… bun bench/mramg-eval.ts 200` for CTX ∈ {100,200,400,800,1600,6000} | `MRAMG_DIR` |
| §Admission | the full admission-rule surface over one persisted score file | `bun bench/dump-scores.ts` once, then `python3 bench/admission-rules.py` | rerank endpoint once; afterwards nothing |
| §External, the margin bounded | question-level bootstrap of the Academic margin | `python3 bench/bootstrap-margin.py` | the score dump |

The score dump this repo ships ([`docs/score-dump.jsonl`](docs/score-dump.jsonl),
6,819 questions, ids and scores only — no benchmark text) lets the admission
study and the bootstrap reproduce **with no model at all**.

Only the `.jsonl` files are needed — 18 MB. `IMAGE.zip` (1.5 GB) is not used by
this path, which scores the text standing in for each image.

`MRAMG_SUBSET=all` is the honest form. Reporting one subset out of six reads as
selection whether or not it was, and the only answer is to run the rest and
publish them together. Note that the published comparators are reported **per
domain**, not per subset: Web pools wit+wiki+web and Lifestyle pools
recipe+manual, so the script sums raw tp/fp before computing pooled precision.
Averaging per-subset precision is a different quantity and would be wrong.

The positional argument is the question limit. Forced emission is not a flag: it
is `TOP_K=1` with the admission floor pushed below any attainable score, so the
filter admits everything and exactly one image is emitted per question. The run
prints its own silent-rate, which is how you confirm it reached 0%.

The configuration was frozen before the dataset was downloaded. If you tune
against MRAMG and then report it, the comparison means nothing — that is the
whole reason the freeze is stated in the paper.

### Scaling it without more annotation

`ppi-eval.ts` implements a prediction-powered estimator: a judge supplies cheap
labels, and its bias is measured on the human-labelled subset and subtracted, so
a useless judge degrades the estimate to human-only rather than corrupting it.

Its representativeness gate now **passes** (0.95x after judging the annotator's
own pools verbatim), and the estimator still buys nothing: at N/n = 3.7 the
rectifier's own sampling error dominates, and four of five intervals fail to
narrow. The machinery works; the ratio is the constraint. See
[`docs/12-prediction-powered-selection.md`](docs/12-prediction-powered-selection.md)
for both the earlier refusal and the current negative result.

## What limits the headline number

The composed selector scores **P 0.542 / R 0.684 / F1 0.605** on `n = 48` items
carrying 19 positive links. That is a small sample, and it is the binding
constraint on the strongest claim in the paper. Scaling the annotation is the
single highest-value contribution anyone could make here; see
[`CONTRIBUTING.md`](CONTRIBUTING.md).

Everything else — Tables I, II and IV, and all of §IV — is either model-free or
external, and does not depend on that sample.

## Results that argue against the method

They are in the paper and in [`docs/10-improvement-experiments.md`](docs/10-improvement-experiments.md),
reported because they were run. Two of them killed hypotheses of our own:
answer-conditioned reranking is worse (F1 0.183 vs 0.288), and adding context to
figure text is worse than the description alone (0.248 vs 0.270).
