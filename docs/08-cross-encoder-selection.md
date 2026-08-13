# The cross-encoder: the biggest single gain, from a service already running

2026-08-10. Every selector in this benchmark — ours and the published baselines —
ranked candidates by cosine similarity between two independently built vectors.
The partner box has been running `BAAI/bge-reranker-v2-m3` the entire time,
serving production as a figure gate, and the benchmark never called it once.

A cross-encoder reads the question and the candidate **together**. That is the
distinction that matters in a primary-school textbook, where a dozen figures are
all legitimately "children learning" and no independently built vector can
separate them.

## Result against human gold

The only gold not derived from a mechanism under test (48 items, 1 annotator):

| system | P | R | F1 | fig/q | silent | wasted |
|---|---|---|---|---|---|---|
| **`sel_rerank`** | **0.250** | 0.263 | **0.256** | 0.42 | 77% | **8** |
| `co_embed` | 0.076 | 0.579 | 0.135 | 3.00 | 0% | 90 |
| `anchor_hybrid` | 0.064 | 0.368 | 0.109 | 2.29 | 15% | 62 |
| `anchor` | 0.052 | 0.263 | 0.087 | 2.00 | 17% | 53 |
| `caption_match` (production) | 0.028 | 0.211 | 0.049 | 3.00 | 0% | 90 |

Precision is **3.3x** the previous best and **8.9x** production. Wasted figures —
emitted on questions with no correct figure at all — fall from 90 to 8.

It is also the first system that behaves like the annotator: silent on 77% of
questions against the human's 62%. Every other system answers with three figures
almost every time.

## Operating point, swept rather than assumed

| threshold | human gold P / R / F1 | harness gold P / R / F1 |
|---|---|---|
| 0 | .069 / .526 / .123 | .061 / .213 / .095 |
| **0.01** | .175 / **.526** / .263 | .122 / .118 / **.120** |
| 0.1 | .259 / .368 / .304 | .190 / .072 / .104 |
| **0.2** | **.316** / .316 / **.316** | .206 / .053 / .084 |
| 0.6 | **.500** / .211 / .296 | .277 / .031 / .056 |

**0.01 is free.** Recall is identical to no threshold at all while precision
multiplies 2.5x — the scores it discards are noise, and removing them costs no
correct figure. It is also the reranker's best F1 on the harness gold, making it
the one point both standards endorse. That is the default.

Higher floors are the right *product* choice for a tutor, where a wrong diagram
costs more than a missing one, but choosing between 0.2 and 0.6 on 48 items would
be fitting to noise.

## Tuned on a grid, taking the plateau rather than the peak

| rule | P | R | F1 |
|---|---|---|---|
| top-2 @0.2 | 0.375 | 0.316 | **0.343** |
| top-1 @0.1 | 0.353 | 0.316 | 0.333 |
| top-2 @0.1 | 0.304 | 0.368 | 0.333 |
| top-1 @0.2 | **0.417** | 0.263 | 0.323 |
| top-1 @0.01 | 0.258 | **0.421** | 0.320 |

The best six cells sit between .320 and .343 across k in {1,2} and thresholds
.01–.2. With 19 positive links those gaps are inside the noise, so the shipped
default is **top-2 @0.1** — the middle of the plateau. Taking the grid maximum
would repeat a mistake this project has already made twice.

**Emitting fewer figures than asked is the finding, not a compromise.** Every
three-figure rule ranked below every one- and two-figure rule: the 2nd and
especially 3rd pick are almost always wrong.

## The anchor adds nothing once a cross-encoder is present

| rule | F1 |
|---|---|
| rerank@0.01 alone | 0.263 |
| rerank@0.01, anchored sorted first | 0.263 (identical) |
| rerank@0.2 + anchored@0.001 (union) | 0.200 (worse) |
| rerank@0.01 AND anchored (intersection) | 0.194 (worse) |
| anchored only | 0.087 |

Reordering by anchor changes nothing; unioning with it adds noise. This is
uncomfortable, because `anchor_hybrid` was shipped to production hours earlier on
the strength of the anchor's selection contribution.

Two things keep that from being a wasted change, and both should be stated
rather than assumed:

1. **Placement is untouched.** The anchor's placement result is significant and
   independent — it holds selection fixed, so this says nothing about it.
2. **Reach.** The production port also fixed figures being appended with no
   positional link at all, which is what made caption matching the only route.

But the honest reading is that for SELECTION specifically, the cross-encoder
supersedes the anchor on this data, and production should move to it once the
larger annotation confirms the effect.

## Where the two golds disagree, and why

On harness gold the cross-encoder's best F1 is 0.120 against `anchor_hybrid`'s
0.219 — it loses. On human gold it wins by 2.9x. Precision favours it on both.

This is explicable rather than awkward: the harness gold marks **2.5x more**
figures correct than a person does (48 vs 19 across the audited items), so it
rewards emitting three figures every time and penalises abstention. A selector
calibrated to a person will always look recall-poor against it.

We should not resolve this by picking the flattering standard. What is known is
that the harness gold agrees with human judgement **at chance** on 80% of its
links (kappa 0.092, `07-human-gold-audit.md`), so optimising against it is
optimising against noise.

## Tested, not merely observed

Everything above was reported as an observation with "n=48, too small" attached.
That was an evasion: a paired test can establish a large effect at that size, and
not running one was a gap in the analysis rather than a limit of the data.

Paired over the 48 human-gold questions:

| comparison | precision difference | 95% CI | p |
|---|---|---|---|
| vs `caption_match` (production) | **+0.265** | [0.078, 0.471] | **0.0028** |
| vs `co_embed` (previous best) | **+0.167** | [0.039, 0.314] | **0.0086** |
| vs `anchor_hybrid` (just shipped) | **+0.196** | [0.020, 0.392] | **0.027** |

**The precision advantage is statistically established against every
alternative**, including the configuration shipped to production hours earlier.
No confidence interval touches zero.

**Coverage is not established either way.** hit@q differences are not significant
in any pairing (discordant counts of 4–7 — underpowered, not null). Whether the
cross-encoder finds the right figure more or less often than the alternatives
remains an open question at this sample size.

So the defensible claim is precise about which axis it holds on: *significantly
more precise than every method tested; coverage undetermined.*

## What this does NOT establish

**Not SOTA, and not claimable as such — but for a narrower reason than sample
size.** The precision effect IS statistically established (above). What is
missing is comparability: a SOTA claim requires measurement against published
systems on a shared benchmark, and ours is a private corpus with a gold standard
we built and then found to be partly broken. No test on our own data can close
that gap. Published comparators report image-selection
F1 in the 0.35–0.50 range for open-weight models on English benchmarks with
curated candidate pools; our 0.316 is not measured on the same task and the
numbers must not be set side by side.

The honest claim is narrower and still worth having: *on the only gold standard
we have that was not built from a mechanism under test, a cross-encoder over
existing figure descriptions is decisively more precise than every method
previously tried, including our own, and is the first to abstain like a person.*

Confirming it needs the larger annotation — several hundred items, 2–3
annotators, inter-annotator kappa reported first.

## Cost

No new model, no new service, no per-query VLM. The reranker is already deployed
and already serving production traffic.

One operational caveat found the hard way: it shares a GPU with generation and
**wedges under sustained load**, returning `CUBLAS_STATUS_INTERNAL_ERROR` for
every request until the container is restarted. An earlier probe produced a
confident and entirely fictitious "capacity envelope" because it was measuring a
dying service. `rerank()` now backs off adaptively instead of trusting a fitted
constant, but a production deployment needs a health check that detects the
wedged state — a container that is `running` and answering every request with an
error is exactly the silent failure this project keeps finding.
