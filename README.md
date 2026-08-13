<div align="center">

<img src="assets/wordmark.svg" alt="IKAT" width="300">

**Figures belong somewhere. Bind them there at ingestion, not at ranking.**

[![License](https://img.shields.io/badge/license-Apache--2.0-blue.svg)](LICENSE)
[![Paper](https://img.shields.io/badge/paper-IEEEtran-B31B1B.svg)](paper/)
[![MRAMG Academic](https://img.shields.io/badge/MRAMG%20Academic-69.63%20IP-E4572E.svg)](#external-validation)

</div>

---

*Ikat* is Indonesian for **to bind**, and the name of a weaving technique in which
threads are tied before the loom so the pattern lands where it should. That is
what this does: it ties each figure to its place in the reading flow, at the one
moment the information still exists.

**The method is not language- or domain-specific.** Anchoring is a join over data
the layout parser already produced — no model, no training, nothing tuned per
corpus. It is evaluated here on two very different bodies of text:

| | what it is | whose gold |
|---|---|---|
| **MRAMG-Bench, Academic** | English arXiv documents | the benchmark authors' |
| **IKAT-Bench** | Indonesian K–12 textbooks | ours (+ human annotation) |

The state-of-the-art result below is on the **English** one. The Indonesian corpus
is not the scope of the claim — it is the hard case that made the problem visible:
only a fifth of its figures carry a printed caption, so a caption-based pipeline
fails there in a way that caption-rich corpora hide.

## The problem

A layout parser reads a page and emits an ordered sequence of blocks — heading,
paragraph, figure, paragraph. The figure's position in that sequence says exactly
where it belongs in the text.

Conventional ingestion then stores prose as chunks and figures as a separate
array, and **throws the ordering away**. Everything downstream tries to
reconstruct it from captions.

On the textbook corpus, that reconstruction is attempting the impossible:

| | hosted OCR | on-prem parser |
|---|---:|---:|
| figures with a printed caption | 19.3% | 34.3% |
| **figures with a recoverable reading-order anchor** | **100%** | **100%** |
| lexically indistinguishable from a sibling figure | 20.7% | 34.1% |
| shares a page with another figure | 49.9% | 54.2% |

Four figures in five have no caption to match on. A fifth of the corpus has
another figure with effectively identical index text, so no matcher can ever
separate them. And half share a page, so page-level provenance is ambiguous too.

Both columns were measured independently, on two extraction pipelines that share
no code and no models. Agreement between them is replication.

## Placement is a distinct failure mode

Take a figure's own anchor chunk as the answer. Retrieval is now perfect, the
answer is perfect, and the correct slot is known exactly. Under those conditions
caption matching **still** lands 2.08 sentences away and is exactly right 43% of
the time. A design with no positional signal — appending at the end, which is what
a co-embedding index can offer — sits 8.71 sentences away and is never right.

Whatever error remains there is intrinsic to the mechanism. Both the retriever and
the generator were removed.

## What actually moved the number

Anchoring fixes *placement*. The practical win turned out to be *selection*, and
it needed a stage that looks at the image rather than at text about the image.

Against human annotation (`n=48`, 19 positive links — the only gold no model
produced):

| system | P | R | F1 |
|---|---:|---:|---:|
| **VLM filter → cross-encoder rank → top-1** | **0.542** | 0.684 | **0.605** |
| VLM selector alone | 0.283 | **0.789** | 0.417 |
| cross-encoder alone | 0.304 | 0.368 | 0.333 |
| deployed production system | 0.028 | 0.211 | 0.049 |

Sight buys **recall**, not precision — the VLM finds four of five figures a person
chose; the cross-encoder finds a third. But seeing the image does not make a model
better at *rejecting*. So compose them: sight decides what is possible,
discrimination decides what is best.

### Showing the model fewer images

Prefill cost scales with images, not calls, so shrinking the model is the wrong
lever. Cutting images is the right one.

| candidates the VLM sees | F1 (4B) | latency | F1 (8B) | latency |
|---|---:|---:|---:|---:|
| 1 | 0.721 | 0.7s | 0.735 | 1.0s |
| **2** | **0.736** | **1.3s** | **0.753** | **2.0s** |
| 3 | 0.736 | 2.0s | 0.757 | 3.0s |
| all (6) | 0.735 | 3.9s | 0.763 | 5.9s |

Two candidates is the whole pipeline. The four it never sees were ones it would
have rejected anyway.

## External validation

Everything above is our corpus and our gold. 80% of our gold links derive from the
anchor itself, and on those our harness gold agrees with a human at chance
(κ = 0.092). Internal care cannot fix that — only data we did not author can.

**MRAMG-Bench, Academic subset** (`n=200`, published comparator figures). The
configuration was frozen before the dataset was downloaded; nothing was tuned on
it.

| system | Image Precision |
|---|---:|
| **IKAT (cross-encoder, 568M params)** | **69.63** |
| **IKAT, forced emission (0% silent)** | **67.50** |
| GPT-4o, LLM-based | 65.28 |
| Claude-3.5-Sonnet, LLM-based | 62.17 |
| GPT-4o, MLLM-based | 60.39 |
| Gemini-1.5-Pro, LLM-based | 59.85 |
| DeepSeek-V3, rule-based | 56.12 |
| Llama-3.3-70B (best open-weight) | 38.78 |

Image Precision counts only emitted images, so abstention could buy the score —
ours abstains on 49%. Forcing exactly one image on **every** question still leads,
with recall rising 41.59 → 61.64. The margin is not selective answering.

## Quickstart

Requires [Bun](https://bun.sh). All configuration is environment variables; nothing
is baked in.

```bash
cp .env.example .env        # point at your embedding / rerank / VLM endpoints
bun bench/structural-analysis.ts     # model-free corpus measures, no GPU needed
bun bench/run-bench.ts --limit 40    # scored run
bun bench/mramg-eval.ts              # external benchmark
```

`bench/structural-analysis.ts` is the one to run first: it needs no model, no API
key and no GPU, and it reproduces the caption/anchor table above from a cached
extraction.

## What's here

| path | |
|---|---|
| `bench/systems.ts` | system definitions — the selection ladder, rerank plumbing, figure text modes |
| `bench/structural-analysis.ts` | model-free corpus measures (A1–A5) |
| `bench/placement-metrics.ts` | displacement, PA@k, Grounded Figure F1 |
| `bench/judge-figures.ts` | VLM judge: κ validation, selector, pipeline, one-call variants |
| `bench/prefilter-pipeline.ts` | the candidate-cut curve and its ceiling |
| `bench/mramg-eval.ts` | MRAMG-Bench evaluation |
| `bench/*-significance.ts` | McNemar exact + paired bootstrap |
| `paper/` | IEEEtran manuscript |
| `docs/` | method notes, metric definitions, deployment write-up |

## Results that argue against the method

Reported because they were run.

- **Anchoring alone does not beat similarity selection.** The win needs a
  competent placement rule, and the best one in our comparison is MRAMG's, not
  ours.
- **Retrieval bounds everything.** The chunk holding the relevant figure is
  retrieved for only 51.8% of figure-bearing questions.
- **Answer-conditioned reranking is worse** (F1 0.183 vs 0.288). Tutor prose
  carries connective material matching almost any figure.
- **Adding context to figure text is worse** than the description alone (0.248 vs
  0.270). This killed our own hypothesis that the gap between MRAMG (67%) and our
  textbooks (30%) was about text representation.
- **A newer reranker is worse here.** Qwen3-Reranker (2025) ranks our figures at
  gold-first 0.288 against 0.788 for the 2024 bge-reranker-v2-m3.
- **On figure-only questions, nothing recovers the content.** Best completeness
  1.93/5, and the one system that beat text-only did so by losing faithfulness.

## The deployment lesson

The composed selector was installed on an air-gapped 8B-class deployment. Every
component check passed — env present, endpoint reachable, model answering, container
healthy — and the stage **ran zero times**.

An admission threshold upstream discarded every candidate. It was 0.2, tuned when
the reranker scored *descriptions*; production scores the *caption*, where the
correct figure scored 0.016. Nothing survived, the stage was skipped, and no log
line was ever written.

**The absence of a line was the only symptom.**

A stage that fails by not executing raises no exception and produces no degraded
output. Component-level verification cannot catch it. If you build one, make it
log a survivor count unconditionally, so `kept 0/108` shows up on the first query
instead of after weeks.

## Citing

See [`paper/`](paper/). Please cite the MRAMG-Bench authors separately for the
external benchmark.

## License

Apache 2.0 — see [LICENSE](LICENSE).

The corpus consists of Indonesian government curriculum textbooks, distributed
under their own terms; this repository contains code and derived measurements, not
the books.
