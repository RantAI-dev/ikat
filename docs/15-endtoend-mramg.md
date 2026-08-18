# 15 — End-to-end on MRAMG: the caveat closed, and a loss found inside it

**Question.** Every external comparison so far placed a *selection* result
beside *end-to-end* systems, stated as a caveat each time. What happens when we
run the benchmark's own generation protocol?

**Answer.** Two things, one against us. With the same 8B generator on both
arms, letting the model insert images while writing (the benchmark's LLM-based
framework) beats our post-hoc selection on F1 — 61.71 vs our best 55.91. Our
pipeline keeps a ten-point precision lead and slightly better prose. And the
admission-rule study's prediction replicated out of sample: its pre-registered
scale-free rule lifted our arm from 48.30 to 55.91 by ending a 50% silence.

---

## Protocol

MRAMG's published generation setup, reproduced: bge-m3 retrieval over
~256-token sentence chunks, top-10 chunks per question, Image P/R/F1 counted
per emission, ROUGE-L against `ground_truth`. Academic subset, all 200
questions. Retrieval surfaces 90.4% of gold images — the shared ceiling.

Two arms, differing in exactly one mechanism:

- **A — generator inserts.** Chunks keep inline `<imgN>` placeholders;
  the model answers and places images itself (prompt adapted from the
  benchmark's own, published in their repo).
- **B — IKAT selects.** Same chunks, placeholders stripped; the model writes
  text only; the frozen pipeline (anchor context → cross-encoder → admission)
  picks the images.

Same retriever, chunks, generator, questions.

## Results (n=200)

| arm | IP | IR | IF1 | ROUGE-L | silent |
|---|---:|---:|---:|---:|---:|
| A: generator inserts | 53.38 | **73.13** | **61.71** | 0.261 | 7% |
| B: IKAT, frozen | **63.91** | 38.81 | 48.30 | **0.277** | 50% |
| B: IKAT, scale-free admission | 47.45 | 68.04 | 55.91 | **0.277** | 0% |

The scale-free row is **pre-registered**: `rel α=0.2 K=3` was fixed by docs/14
on the score dump before this experiment existed, and applied unchanged.

## Reading it honestly

1. **The loss.** The generator places images with the full answer in view; our
   selector ranks against the question alone. Inline insertion wins F1. This
   qualifies the selection-only Academic win (69.12): that number holds under
   its own protocol, but does not automatically transfer to end-to-end F1.
2. **What we keep.** Precision (63.91, ten points above arm A) and marginally
   better text (ROUGE-L 0.277 vs 0.261 — figure duty and writing duty
   interfere when one model does both). For a system that shows figures to
   schoolchildren, precision is the expensive metric.
3. **The replication.** docs/14 predicted the frozen floor's failure shape
   (silence, recall collapse) and its fix, on independent data. Applied here:
   +7.6 F1, silence 50% → 0%. A pre-registered prediction that survives a
   fresh experiment is worth more than either number alone.
4. **What is not measured.** Placement/provenance — the axis the paper is
   about. Arm A's insertions carry no provenance. The benchmark scores
   position only via LLM-judged metrics (GPT-4o judge); deferred, not proxied.

## The judged half: placement ties, selection loses

The benchmark's four LLM-judged metrics, run blind: 400 answers (arm B in its
pre-registered variant, each figure placed after the answer sentence matching
its anchor context best, via cross-encoder), interleaved under neutral shuffled
keys `J000..J399`, scored by a single judge model, key unblinded only at
scoring. Paired per-question differences, bootstrap CIs (B=5000):

| metric (scale) | mean A−B | 95% CI | verdict |
|---|---:|---|---|
| relevance (1–5) | +0.306 | [+0.145, +0.478] | A wins |
| effectiveness (1–5) | +0.242 | [+0.081, +0.403] | A wins |
| **position (0–1)** | **−0.012** | **[−0.085, +0.061]** | **tie, 48 vs 49** |
| comprehensive (1–5) | +0.085 | [−0.035, +0.210] | tie |

The split is the finding. The generator picks better images — it selects with
its own answer in view. But on *position*, the axis this paper exists for, a
mechanical join ties the generator's own inline placement exactly, while
remaining traceable to a source location. Overall answer quality is
indistinguishable. Placement does not need the generator; selection is where
the generator's edge lives — which is exactly the stage docs/11 showed a
vision gate can supply.

Caveats: one judge model (Claude, via blind subagent batches); text stand-ins
for images, not pixels; the judge shares a vendor with the evaluation
tooling's author (not with any system under test — generator is SEA-LION).
Blinding is the mitigation; per-item verdicts are shipped.

## Deviations from their protocol, stated

- Generator: `Llama-SEA-LION-v3.5-8B-R` fp8 via the box's vLLM, thinking off.
  (Intended v4-8B-VL via ollama, but ollama on this arm64 Blackwell host loads
  models on CPU — `size_vram: 0`, 75 s to prefill 2.5k tokens — despite a GPU
  device reservation.)
- 4096-token window: whole retrieved chunks are dropped from the bottom of the
  ranking when the prompt would not fit, never truncated mid-sentence.
- Captions for arm A are anchor tails (the caption field is not in these
  dumps; the text before the placeholder is what their placeholder-based
  baseline reads).
- LLM-judged metrics (image relevance/effectiveness/position, comprehensive)
  not run.

## Infra notes

The box's LLMOps platform recycles the vLLM container mid-run (observed:
`rantai-vllm` "Up 54 seconds", a dead `vllm-noadapter-*` beside it). The
runner resumes by id and retries through ~6-minute outages. All four socat
bridges (rerank 8095, embed 8096, ollama 8097, vllm 8098) were removed after
the run.

## Files

- `src/ikat/mramg-endtoend.ts` — index / run / score / rescore.
- `corpus/results/endtoend/answers-arxiv.jsonl` — both arms' answers, emitted
  ids, retrieved-candidate lists (resume-safe by question id).
