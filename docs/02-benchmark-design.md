# Benchmark Design

Status: draft for review. Last updated 2026-08-06.
Companion to `01-related-work-positioning.md`.

Working name: **IKAT-Bench** — *Indonesian Curriculum Adaptive Tutoring benchmark*.
("ikat" = to bind/tie — the figure bound to its explanation. Rename freely.)

---

## 1. What the benchmark has to prove

Three claims, each with its own experiment. Written as falsifiable statements so
that a failed experiment produces a reportable negative result rather than a
temptation to re-tune.

| # | Claim | Killed by |
|---|---|---|
| C1 | Figures in curriculum textbooks carry content absent from the prose, so text-only RAG has a hard ceiling | text-only RAG scoring within noise of multimodal on figure-dependent questions |
| C2 | Placement is a failure mode distinct from retrieval — systems that retrieve the right figure still place it wrong | placement accuracy tracking retrieval accuracy almost perfectly (high correlation, low residual) |
| C3 | Anchor-preserving ingest resolves placement without fine-tuning or a serving-path VLM | anchor pipeline losing decisively to caption-matching, or requiring a VLM anyway |

C2 is the load-bearing one. If placement turns out to be trivially implied by
retrieval, the paper collapses to a dataset contribution and we should reframe
early rather than late.

---

## 2. Corpus

**Source.** Indonesian K-12 curriculum textbooks. Currently in hand: 31 books from
the *Buku Referensi* deployment, plus the Hindu SD kelas III book used in
development. Development book is held out of all reported numbers — it has been
used for debugging and is contaminated.

**Split discipline.**
- *dev* — 3 books, all iteration and prompt tuning happens here
- *test* — remaining books, run **once per reported configuration**, no tuning against it

Every book is ingested through the same pipeline; the ingest path used
(MinerU-API vs Mistral OCR vs MinerU sidecar) is recorded per document and
reported, because the sidecar path cannot supply anchors (see §7).

**Licensing.** Blocking for release, not for internal results. Kemendikbud /
Kemenag curriculum titles are generally freely redistributable, but this must be
cleared per title. Until cleared, the paper promises the *benchmark construction
code* plus per-book identifiers, not the PDFs.

---

## 3. Structural ground truth: the document as its own annotator

A layout parser emits a document as an ordered block sequence

    B = [b₁, b₂, …, b_n],  each bᵢ either a text block or a figure block

For a figure `f` at index `i`, the surrounding blocks are exactly the prose the
book's authors chose to place it inside. That is a **human-authored placement
label**, free, at corpus scale, with no annotator disagreement.

Define:
- `anchor(f) = i` — reading-order index of the figure
- `ctx(f) = concat(b_{i-1}, b_{i+1})` — the immediately adjacent prose
- `chunk(f)` — the retrieval chunk whose source span covers index `i`

This is what replaces ImageRef-VL's human 0–3 scoring of every candidate position.

**Validity threat, stated up front.** Layout-gold measures fidelity to the book's
own sequencing. The system's answer is *generated* prose, not the book's prose, so
the correspondence is a proxy. §6 defines the human study that must establish how
strong that proxy is. If the correlation is weak, the metric contribution is
downgraded and we fall back to hand-scored positions on a subsample. This is
declared before we see the numbers.

---

## 4. Question construction

Generated from source spans, then human-verified — following UniDoc-Bench's
protocol so our numbers are comparable in kind.

For a sampled span `s` of a book, generate questions of four types:
1. **factual** — answer stated in the prose
2. **figure-dependent** — answer obtainable *only* from a figure/table in `s`
3. **explanatory** — "why/how", the tutoring case, where a figure aids explanation
4. **comparison** — spans two spans or a table

**The figure-dependent split is the instrument for C1** and must be constructed
adversarially: a question only enters this class if a strong text-only model,
given the full page prose *with the figure removed*, cannot answer it. Verified by
human check, not by the generator's self-report.

Target ≈ 800 questions on test. Each carries:
- gold answer
- gold evidence span(s)
- gold figure set `F*(q)` = figures whose anchor falls inside the gold span
- class label (1–4)

Two annotators on a ≥ 20% subsample, report Cohen's κ. Reviewers will ask.

---

## 5. Metrics

### 5.1 Figure selection (comparable to prior work)

Standard, over `F*(q)`:

    FigPrecision = |F_pred ∩ F*| / |F_pred|
    FigRecall    = |F_pred ∩ F*| / |F*|
    FigF1        = harmonic mean

### 5.2 Placement — the new part

Let the generated answer be sentences `A = [a₁ … a_m]`. A system inserts figure
`f` after sentence index `p(f)`.

**Ideal position.** The answer sentence that actually conveys the figure's
surrounding content:

    ideal(f) = argmax_j  sim( a_j , ctx(f) )

where `sim` is cosine similarity under the same embedding model used for
retrieval (reported explicitly; sensitivity to this choice is an ablation).

**Placement Displacement** — signed distance in sentences, reported as mean |PD|:

    PD(f) = p(f) − ideal(f)

**Placement Accuracy at tolerance k:**

    PA@k = | { f ∈ F_pred ∩ F* : |PD(f)| ≤ k } | / | F_pred ∩ F* |

Note the denominator: PA@k is conditioned on the figure being *correctly selected*.
This is deliberate — it isolates placement from retrieval, which is what C2 needs.
Report PA@0 and PA@1.

### 5.3 Headline metric: Grounded Figure F1

A figure counts as a hit only if it is the right figure **and** in the right place:

    GF-F1 = FigF1 computed with the hit predicate  (f ∈ F*) ∧ (|PD(f)| ≤ 1)

This is the number that distinguishes this benchmark from every retrieval-only
multimodal RAG benchmark, and the one C3 is argued on.

### 5.4 Answer quality

Completeness and faithfulness, LLM-as-judge with a published rubric and a
human-scored subsample to report judge–human agreement. Naming these the same as
UniDoc-Bench is intentional and should be flagged as *comparable in kind, not
directly comparable in value* (different corpus, different language).

### 5.5 Cost

Not a footnote — it is half of C3. Per query: wall-clock latency, generator
tokens, VLM invocations, and whether any fine-tuned model is required. A method
that wins GF-F1 at 30× cost loses the deployment argument, and we should say so.

---

## 6. Validation: LLM-as-judge

**Decision (2026-08-06): the judge is an LLM, not human raters.** Implemented in
`tests/bench-kb/src/judge.ts`, unit-tested in `tests/unit/judge.test.ts`.

Two studies, as before, now judge-run:

**(a) Layout-gold validity.** ~150 (question, answer, figure) triples. The judge is
shown the answer split into numbered insertion slots and asked, **forced-choice**,
which slot the figure belongs in. We report Pearson correlation between the judge's
slot and `ideal(f)` from layout. This licenses the metric and runs *before* the main
results. Forced choice rather than a rating because it reproduces far better across
repeats.

**(b) Pedagogical quality.** Judge scores completeness / faithfulness / helpfulness
1–5 against the textbook reference answer.

### 6.1 Bias controls — enforced in code, not promised in prose

LLM-as-judge is the most attacked component of a paper like this. Each control is a
guard in the implementation so it cannot quietly lapse:

| Risk | Control | Where |
|---|---|---|
| Self-preference (judge grading its own family) | `assertJudgeIndependence` **throws** if any generator shares the judge's vendor | `judge.ts` |
| System identity leaking to the judge | answers relabelled A/B/C… under a seeded permutation; names never enter the prompt | `shuffle` |
| Position bias in pairwise comparison | every pair judged in **both orders**; only order-consistent verdicts count as decisions; flip rate reported | `judgePairwise` |
| Unstated run-to-run variance | every judgement repeated `repeats` times, majority taken, agreement carried in the result | `selfConsistent` |
| Parse failures silently favouring compliant models | tolerant JSON extraction, unparseable rate reported | `parseJsonLoose` |
| Rating drift | forced choice preferred over Likert wherever the question allows it | placement prompt |

Consequence of the independence guard worth stating plainly: **with an Anthropic
judge, no Anthropic model may appear among the systems under test.** The guard makes
that configuration impossible rather than disclosing it as a caveat.

Reported per run, as first-class results and not an appendix footnote: judge model
and version, `meanAgreement`, `unanimousRate`, `positionFlipRate`, `unparseableRate`.
`summarizeDiagnostics` emits all of them and no results table ships without them.

### 6.2 Residual risk — stated, not hidden

The core placement metric is **structural**: `ideal(f)` comes from the source
document's typesetting and involves no model at all. So the headline numbers do not
depend on the judge. The judge is used to show layout-gold *agrees with a competent
reader* — which is an external check only insofar as the judge is a competent reader.
An LLM judge validated only against itself is circular.

`HUMAN_SPOTCHECK_NOTE` in `judge.ts` carries this verbatim into every diagnostics
file, and a unit test asserts it is present, so it cannot be dropped on the way to
the manuscript:

> A human spot-check of ≥150 items remains the correct validation and is left as
> future work.

If a reviewer insists on human validation, study (a) is the one to run with people —
150 forced-choice items is roughly a day of annotator time, and the harness already
produces the exact item format.

---

## 7. Systems compared

All share the same corpus, chunker, embedder, generator, and top-k. Only the
figure mechanism varies. Configs published.

| ID | System | Figure mechanism | Placement mechanism |
|---|---|---|---|
| S0 | Text-only RAG | none | n/a — floor for C1 |
| S1 | Caption-match co-retrieval | caption keyword + rerank gate | post-hoc keyword placement |
| S2 | Joint co-embedding | CLIP/SigLIP dual index | post-hoc similarity |
| S3 | VLM-over-page | full page image to a VLM | model-decided |
| S4 | **Anchor-preserving (ours)** | reading-order anchor from parser | citation of anchor chunk |
| S5 | S4 + VLM figure descriptions | anchor + ingest-time description | citation of anchor chunk |
| S6 | ImageRef-VL-style | fine-tuned VLM emits image IDs | model-decided |

**S1 is our own current production system.** Using it as a baseline is honest and
convenient — it is the strongest realistic "reasonable engineering" comparator and
we cannot be accused of strawmanning it.

**S2 is expected to lose** — UniDoc-Bench reports joint multimodal embedding at
64.1% completeness vs 68.4% for text–image fusion. We implement it anyway; a
predicted-and-confirmed negative result is evidence of a faithful setup.

**S6 is the hard one.** Full reproduction requires fine-tuning an InternVL2-class
model. If we cannot run it, we say so explicitly and compare against their
published protocol on our data with a non-fine-tuned VLM, labelled as a weakened
variant — never as ImageRef-VL itself.

**Baseline fairness protocol.** Each baseline gets the same tuning budget as S4,
tuned on *dev* only, with the tuning procedure logged. State this in the paper.

**Generator constraint imposed by §6.1.** With an Anthropic judge, no system under
test may use an Anthropic generator. This rules out the `anthropic/claude-*` entries
in the existing `bench-e2e.ts` sweep for any judged run; the generator pool becomes
Gemini / Qwen / our house models. Either that, or the judge moves to another vendor —
but the two cannot overlap, and `assertJudgeIndependence` enforces the choice at
runtime rather than leaving it to reviewer trust.

---

## 8. Ablations (these come free from the build order)

The engineering roadmap and the ablation table are the same list, which is why the
harness must exist before any of it is built:

| Ablation | Corresponds to |
|---|---|
| current production system | S1, baseline snapshot |
| + assetKey dedup, context-aware rerank | Phase 0 |
| + reading-order anchors | Phase 1, isolates the anchor contribution |
| + VLM figure descriptions at ingest | Phase 2 |
| + VLM-at-answer for visual questions | Phase 3 |
| anchor without citation-based placement | isolates placement mechanism from anchor |
| `sim` embedder swap | metric sensitivity |

---

## 9. Build order

1. **Harness first** — runner, metric implementations, per-config result store
2. **Freeze S1 baseline numbers** before touching any retrieval code
3. Phase 0 → measure. Phase 1 → measure. Phase 2 → measure.
4. Baselines S0, S2, S3 in parallel with phases
5. Human study (a) as soon as any system produces answers
6. S6 last, scope decided by available compute

Rule: **no configuration is reported that was not produced by an actual run of the
harness.** No hand-copied numbers, no "expected" values in tables.

---

## 10. Target venue

Assumption in force: education-AI framing — *Computers & Education: Artificial
Intelligence* or *IEEE Access*, both Scopus Q1. Consequence: human study (b) is
mandatory and the on-prem/low-resource deployment argument is a feature, not an
aside.

If the framing moves to IR/NLP (SIGIR, EMNLP), the dataset and metric lead and the
teacher study becomes optional. The benchmark design does not change either way —
only the emphasis — so this decision can be deferred until results exist.
