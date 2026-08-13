# Eight attempts to make figure selection better

2026-08-12. After the cross-encoder reached 0.269 precision on our textbooks, the
question was whether anything could push it further. Eight experiments, run in
order of expected payoff, plus two corrections to conclusions drawn too early.
Reported together because the failures are as informative as the wins — and
because two of the failures were mine rather than the method's.

**Where it ended.** Best accuracy: 8B VL + strict prompt + every candidate,
F1 0.800 at 6.3 s. Best shippable: 4B VL + strict prompt + top-2 candidates,
F1 0.786 at 1.4 s. Production today scores F1 0.049 on human gold.

**What actually moved the number:** the judge prompt (largest single gain), then
composing VLM recall with cross-encoder precision, then cutting candidates before
the VLM sees them. **What did not:** a smaller model as decider, answer-conditioned
reranking, adding context to figure text, and a newer reranker.

## A. Sight — a VLM selecting instead of a cross-encoder reading descriptions

The asymmetry that motivated it: our *judge* looks at the picture and agrees with
a human at κ = 0.552, while our *selector* reads a 300-character description and
never sees the image at all.

Against the human annotation (the only gold no VLM produced):

| | P | R | F1 |
|---|---|---|---|
| VLM selector | 0.283 | **0.789** | 0.417 |
| cross-encoder | **0.304** | 0.368 | 0.333 |

**The gain is recall, not precision.** The VLM finds four of every five figures a
person chose; the cross-encoder finds about a third. Descriptions do discard
enough to lose half the correct figures — but seeing the image does not make a
model better at rejecting wrong ones.

**A circularity warning worth generalising.** Scored against the Sonnet-built
gold the same VLM reached 0.521 precision, nearly double its honest 0.283. A gold
built by a VLM looking at pictures flatters a selector that is a VLM looking at
pictures. Any pipeline containing a VLM stage must be scored on gold no VLM
produced. `IKAT_GOLD` exists to make that check one environment variable.

## B. Rerank against the generated answer rather than the question

A question is short and often names nothing concrete; the generated answer names
the concept, the units, the steps. The gold answers could not test this — median
17 characters, 96 of 165 under 40 — so 165 answers were generated the way the
system generates them, from retrieved text only, with no figure in the prompt
(one there would let the answer echo the figure we are about to select). Median
505 characters.

| rerank query | P | R | F1 |
|---|---|---|---|
| question (current) | **0.269** | 0.309 | **0.288** |
| generated answer | 0.165 | 0.206 | 0.183 |
| question + answer | 0.145 | 0.353 | 0.205 |

Substantially worse, not marginally. The likely mechanism is **dilution**: five
sentences of tutor prose carry connective and general material matching almost
any figure, while the question is short and discriminating. Concatenating the two
also hurts, which fits — the cost is the extra text, not the substitution.

## C. What text represents a figure

| mode | P | R | F1 |
|---|---|---|---|
| description (current) | 0.232 | 0.324 | **0.270** |
| description + context | 0.224 | 0.279 | 0.248 |
| caption only | 0.232 | 0.235 | 0.234 |
| context only | 0.121 | 0.118 | 0.119 |

Description alone wins. The description+context blend that looked like +3 F1
points on 14 links is *worse* on 68 — it was noise, and declining to claim it at
the time was right.

**This also kills a hypothesis carried for most of the session.** The gap between
MRAMG (67% precision) and our textbooks (30%) was attributed to representation:
there the figure is represented by surrounding document prose, here by a VLM
description. Had that been the cause, context-mode would have closed the gap. It
is the worst of the four.

The corpus is genuinely harder, for a concrete reason: an MRAMG recipe carries
step-specific prose beside each image; a curriculum textbook carries general
narrative beside a decorative one.

## The combination neither method could reach alone

A and C together say: sight decides what is *possible*, discrimination decides
what is *best*. Neither does both. So compose them — VLM filters, cross-encoder
ranks the survivors, take the top one.

Against the human gold:

| | P | R | F1 |
|---|---|---|---|
| **VLM → rerank → top-1** | **0.542** | 0.684 | **0.605** |
| VLM alone | 0.283 | 0.789 | 0.417 |
| cross-encoder alone | 0.304 | 0.368 | 0.333 |
| production | 0.028 | 0.211 | 0.049 |

Precision nearly doubles over the best single method and is **19x production**,
while recall stays close to the VLM's own. More than half the figures shown are
now correct — the first configuration for which that is true.

### What must travel with that number

- **n = 48, 19 positive links.** The 165-question run gives 0.806 precision
  against the Sonnet gold, which is circular for the VLM stage — it confirms the
  pipeline behaves consistently, *not* the size of the effect. That figure
  belongs to the *strict* judge prompt; the loose prompt gives 0.690 on the same
  set (see "The prompt was the missing variable" below).
- **The VLM stage costs one call per candidate at serving time**, which the
  cross-encoder alone did not. Latency on the partner GPU is unmeasured.
- Enlarging the evidence needs **more human annotation**. No amount of compute
  substitutes: the Sonnet gold is circular against this pipeline by construction.

## D. One call, every candidate image — and the same design at 4B

The obvious serving design was never tested: hand the chat VLM all candidate
images at once and let it pick. Two variables had been changed together and
never crossed — picking from a numbered list of *descriptions* scored 0.114,
judging *images* one at a time scored 0.283. This is the missing cell, and it is
the cheap one, since it costs a single call rather than one per candidate.

Wall-clock is reported per question, measured on the partner GPU while nothing
else was generating. Both models were warmed before the run so load time is not
counted as latency.

| selector | P | R | F1 | fig/q | silent | latency |
|---|---|---|---|---|---|---|
| VLM → rerank → top-1 (8B) | **0.542** | 0.684 | **0.605** | — | — | ~14 s, 6 calls |
| one call, all images (8B) | 0.448 | 0.684 | 0.542 | 0.60 | 40% | **8.9 s**, 1 call |
| one call, all images (4B) | 0.231 | 0.316 | 0.267 | 0.54 | 46% | 5.2 s, 1 call |
| cross-encoder alone | 0.304 | 0.368 | 0.333 | — | — | < 1 s |

At 8B the design holds: **recall is identical to the two-stage pipeline** and only
precision gives way, by ten points. The failure mode named in advance — a list
inviting a pick when nothing fits — appears but stays small: silence falls from
50% to 40%, which is exactly where the lost precision goes.

The latency saving is smaller than the call count suggests. Six calls to one cut
14 s to 8.9 s, not to 2 s, because the cost is the **visual prefill of six
images**, not per-call overhead. Fewer calls does not mean less work.

Halving the model does not rescue that. The 4B loses more than half its recall
(0.684 → 0.316) to save 3.7 s, landing **below the cross-encoder on every metric
while being five times slower than it**. It is dominated, so there is no operating
point at which it is the right choice.

What this closes: neither VLM configuration is fast enough to sit in the reply
path. The route to shipping is not a smaller model but **not making the reader
wait** — stream the answer, attach the figure when it lands — or cutting the
candidate set, since prefill scales with images and not with calls.


## E. Show the VLM two candidates instead of six

Experiment D ended by naming the only lever left: prefill scales with images, so
cut images. The cross-encoder already scores every candidate in ~0.02 s and is
the more precise of the two methods — let it discard the obvious losers before
the VLM opens its eyes.

    cross-encoder ranks all candidates -> VLM judges only the top N
                                       -> cross-encoder ranks survivors -> top-1

Each pair is judged once and its wall-clock recorded; the N-curve is then
computed over subsets of those judgements, so latency below N = all is
**projected** as N x (measured pair cost) + measured rerank cost rather than
observed end-to-end. The projection is sound only because pairs are judged
independently, which is a property of the design, not an assumption.

165 questions, Sonnet gold:

| N the VLM sees | 4B F1 | 4B lat | 8B F1 | 8B lat |
|---|---|---|---|---|
| 1 | 0.721 | 0.7 s | 0.735 | 1.0 s |
| **2** | 0.736 | **1.3 s** | **0.753** | **2.0 s** |
| 3 | 0.736 | 2.0 s | 0.757 | 3.0 s |
| all (6) | 0.735 | 3.9 s | 0.763 | 5.9 s |

**Two candidates is the whole pipeline.** Going from 6 images to 2 costs 0.010 F1
at 8B and *gains* 0.001 at 4B, for a two-thirds cut in latency. The four
candidates the VLM never sees were ones it would have rejected anyway — it was
spending 80% of its time confirming the cross-encoder's bottom of the list.

The prefilter's own ceiling explains why this is safe and where it stops being
safe: the correct figure is still present in the top 2 for 59 of 68 links
(0.868), and the full pipeline recovers 0.809 of them. At N = 2 the binding
constraint is the VLM, not the cut. At N = 1 the ceiling falls to 0.765 and
recall follows it down — that is the point where the prefilter starts destroying
answers rather than noise.

**On model size.** At n = 48 the 4B and 8B returned identical figures and that was
written up here as "size stops mattering". It was wrong, and the correction is
instructive: re-running the 8B on the same 48 questions returned F1 0.591 where
it had returned 0.605, so run-to-run noise at that n is larger than the effect
being claimed. At 165 questions, matched prompt and matched N, they separate —
and in the opposite direction to the first guess:

| 165 q, loose prompt | P | R | F1 |
|---|---|---|---|
| 4B, N = 2 | 0.697 | 0.779 | 0.736 |
| 8B, N = 2 | 0.705 | 0.809 | 0.753 |
| 4B, all | 0.684 | 0.794 | 0.735 |
| 8B, all | 0.690 | 0.853 | 0.763 |

**Precision is nearly the same; the 8B's advantage is recall** — 0.853 against
0.794 when both see everything. The extra parameters buy figures the 4B fails to
recognise, not better rejection of wrong ones. The 4B costs 0.017–0.028 F1 and
runs about 35% faster, which is a real operating point rather than a dominated
one.

## F. The prompt was the missing variable

A 165-question run recorded early in this work gave P 0.806 / R 0.794 / F1 0.800,
and every later run of what looked like the same configuration returned P 0.690 /
R 0.853 / F1 0.763. Two independent implementations agreed on 0.690, so the
0.800 was written up here as an unreproducible number and withdrawn, with a
paragraph about contended GPUs and single measurements.

That was the wrong conclusion, and it was wrong in the more embarrassing
direction: nothing was flaky. The original run used the **strict** judge prompt.
Re-running with `IKAT_JUDGE_PROMPT=strict` returns 0.806 / 0.794 / 0.800 exactly,
survivors per question 0.71 against 0.87 — every digit reproduces. The withdrawal
is itself withdrawn.

What actually held constant across all the confusion was the *other* variable, and
that is the finding:

| 165 questions, 8B, all candidates | P | R | F1 | survivors/q |
|---|---|---|---|---|
| loose prompt | 0.690 | **0.853** | 0.763 | 0.87 |
| **strict prompt** | **0.806** | 0.794 | **0.800** | 0.71 |

Twelve points of precision for six points of recall. The strict prompt is the
single largest improvement in this document and it costs nothing at serving time
— it is the same model, the same call, different words. It was written months
earlier to fix the judge's permissiveness against human labels, and never tried
in the selection pipeline.

The real lesson is not about GPUs. It is that **an environment variable is part of
a configuration**, and a number recorded without its full environment is a number
that cannot be defended. Both of my accounts of the discrepancy — "noise" and
then "unreproducible" — were attempts to explain away a disagreement rather than
find the variable that produced it. The variable was in a log line I had already
read.

## G. The whole grid, and what to ship

Model x prompt x how many candidates the VLM sees. 165 questions, Sonnet gold,
latency measured per VLM pair on the partner GPU and projected across N.

| model | prompt | N | P | R | F1 | latency |
|---|---|---|---|---|---|---|
| 4B | loose | 2 | 0.697 | 0.779 | 0.736 | 1.3 s |
| 4B | loose | all | 0.684 | 0.794 | 0.735 | 3.9 s |
| **4B** | **strict** | **2** | 0.764 | 0.809 | **0.786** | **1.4 s** |
| 4B | strict | all | 0.757 | 0.824 | 0.789 | 4.0 s |
| 8B | loose | 2 | 0.705 | 0.809 | 0.753 | 2.0 s |
| 8B | loose | all | 0.690 | 0.853 | 0.763 | 5.9 s |
| 8B | strict | 2 | 0.810 | 0.750 | 0.779 | 2.1 s |
| **8B** | **strict** | **all** | 0.806 | 0.794 | **0.800** | 6.3 s |

Two rows matter. **8B + strict + every candidate is the most accurate thing we
have (F1 0.800).** **4B + strict + two candidates reaches F1 0.786 in 1.4 s** —
0.014 F1 behind, four and a half times faster, and comfortably inside a chat
reply. Against the production selector's F1 0.049 on human gold, either is a
different category of system.

Three things in this table were not obvious beforehand:

- **The strict prompt helps the small model more** (+0.050 F1 at N = 2) **than the
  large one** (+0.026). Most of what the 8B's extra parameters were buying was
  restraint, and the prompt supplies restraint for free.
- **Prefiltering interacts with the prompt.** Under the loose prompt, cutting six
  candidates to two was nearly free (−0.010 F1) because it removed false
  positives the prompt was letting through. Under the strict prompt those false
  positives are already gone, so the cut only costs recall (−0.021). Two
  improvements that each looked additive are partly substitutes.
- **At N = 2 the 4B beats the 8B** (0.786 vs 0.779), reversing at N = all. Well
  inside noise, but it does mean there is no configuration where paying for the
  8B at two candidates is justified.

### The caveat that decides how much of this to believe

Every number in that table is scored against the **Sonnet-built gold, which is
circular for a pipeline containing a VLM** — the same warning this document
raises in experiment A, applied to its own results.

The check that matters is the human gold, and it does not confirm the prompt
effect. At n = 48: strict F1 0.605, loose F1 0.591 — a gap of 0.014 against
measured run-to-run noise of 0.014 at that sample size. **On the only
non-circular gold available, strict and loose are indistinguishable.**

So the honest reading is narrower than the table suggests: the strict prompt
reliably makes our selector agree *with Sonnet* more often. Whether it makes the
selector agree with a *person* more often is untested, because 48 items and 19
positive links cannot resolve an effect this size. The prefilter result does not
depend on this — it reproduces on both golds and both models — but the headline
0.800 does.

**What that implies for the next step, concretely:** more human annotation is now
the binding constraint on every remaining decision, not compute and not model
choice. Roughly 200 annotated items would resolve a 0.05 F1 difference; we have
48.

## H. A newer reranker, and why we kept the old one

The cross-encoder now carries more weight than any other component: it ranks the
candidates the VLM is allowed to see, and it breaks the tie among survivors. So
the obvious remaining lever was to replace it. `bge-reranker-v2-m3` dates from
2024; `Qwen3-Reranker` (2025) reports better multilingual retrieval across the
board.

**`jina-reranker-v2-base-multilingual` cannot run on our TEI at all.** Its
`config.json` omits `model_type` because the architecture ships as
`trust_remote_code`, and TEI refuses to parse it. Dead in five seconds — the
right cost for finding out.

**`Qwen3-Reranker` runs and is clearly worse here.** Served through vLLM, scored
on the 66 questions that have a gold figure, with the VLM removed so only ranking
is under test:

| reranker | gold ranked first | MRR |
|---|---|---|
| **bge-reranker-v2-m3** | **0.788** | **0.859** |
| Qwen3-Reranker-0.6B | 0.288 | 0.481 |
| chance (~6 candidates) | ~0.17 | ~0.41 |

bge stays. Being newer and stronger on public multilingual benchmarks did not
transfer to ranking 300-character Indonesian figure descriptions against student
questions.

### The first version of this section overstated the gap, and the fault was ours

It reported F1 0.034 for Qwen3 against 0.260 for bge and called the ordering
"close to random". Both claims are withdrawn.

Those F1s came from `rerank-sweep.ts`, which admits candidates above an absolute
score threshold — 0.001 — applied identically to both models. But the two score
scales are not comparable. On these questions bge outputs 0.0000–0.1174 while
Qwen3 outputs 0.03–0.999. At 0.001 the threshold makes bge abstain on 20% of
questions and Qwen3 on **none**. Since many questions have no correct figure at
all, abstention is most of what precision measures here — so the comparison was
scoring calibration under a threshold tuned for one model, not ranking quality.
Threshold-free, the honest gap is 0.288 against 0.788: a large real deficit, not
the near-total failure first reported.

**And the reasoning that produced it was weaker than it looked.** Three checks
were run and described as independent — instruction prefix, the 4B, and a
`-seq-cls` conversion that removes the yes/no wiring. They all returned F1 0.034
to three decimals, which was read as convergent evidence. Three implementations
agreeing to three decimals is not convergence; it is a shared cause. All three
went through the same vLLM `/rerank` path and the same threshold, so all three
were measuring the same artefact.

What actually settled it was looking at raw scores instead of summary metrics:

- **Not degenerate.** Qwen3's spread within a question is 0.63–0.97; it is
  discriminating, just wrongly.
- **Not passthrough.** Its ordering matched input order in 0 of 66 questions.
- **Not the adapter's batching.** One-shot scoring and the batched-by-4 path
  disagree on 5 of 66, far too few to explain anything.
- **Not inverted labels.** The scores looked like a flipped `classifier_from_token`
  — gold repeatedly scoring low while irrelevant figures scored 0.99. Sorting
  ascending instead gives 0.121, worse than descending's 0.288. The head is wired
  correctly; the model simply prefers the wrong figures.

The conclusion survives. The evidence for it did not, and had to be rebuilt.

The untested route remains a reranker that reads the **image** rather than a
description of it (ColPali/ColQwen2-style late interaction, image embeddings
precomputed at ingest). Every text reranker inherits the same ceiling — a
description written before the question was known — which experiment C measured
at 0.53 recall. Changing the reranker cannot lift it; changing what the reranker
reads can.
