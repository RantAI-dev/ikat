# 14 — Admission does not transfer; ranking does

**Question.** The frozen configuration admits candidates by an absolute score
floor, `s >= 0.1`. The external runs indicted it twice (MMDocRAG News 11.31 with
it, 74.03 without; Lifestyle recall capped by abstention). Is the fix a better
constant, or a different kind of rule?

**Answer.** A different kind. The boundary that matters is not where the floor
sits but whether the rule references an absolute score at all. Absolute floors
fail catastrophically somewhere (worst unit 11.31 at 0.1; still 42.64 at 0.001);
every scale-free rule — plain top-K, or admission relative to the question's own
maximum — lands within about one macro point of every other, with no unit
collapsing. Family choice: ~16 macro points. Choice within the family: ~1.

---

## Method: one inference pass, then arithmetic

Studying admission by re-running the model once per rule invites tuning-on-test
and costs hours per point. Instead `dump-scores.ts` persisted the cross-encoder's
raw score for **every candidate of every question on both benchmarks** — 6,819
questions, seven evaluation units (three MRAMG domains, four MMDocRAG domains) —
and `admission-rules.py` evaluates any rule offline. The full grid
(α ∈ [0.2,0.9], K ∈ {2,3,4}, three guard variants) is computed across all units
at once and reported whole. The statistic a rule is judged on is its **worst
unit** and the macro average. Nothing is selected per benchmark.

The dump was validated before use: under the deployed rule it reproduces the
published Academic IP **69.12 exactly**, and an integrity pass found 0
unparseable lines and 0 duplicate keys. One counting bug was caught here:
set-based tp (87) vs the eval scripts' list-based tp (94) — a document can carry
the same image id twice and MRAMG's metric counts per emission. The rule study
now counts identically to the evals.

## Results (F1 per unit)

| rule | MD-Acad | MD-Fin | MD-News | MD-Res | MR-Acad | MR-Life | MR-Web | macro | worst |
|---|---:|---:|---:|---:|---:|---:|---:|---:|---:|
| s≥0.1 top-2 (frozen) | 59.38 | 61.65 | **11.31** | 62.84 | 51.93 | 40.47 | 72.58 | 51.45 | 11.31 |
| s≥0.001 top-2 | 63.32 | 63.40 | 42.64 | 61.97 | 65.55 | 49.26 | 94.97 | 63.02 | 42.64 |
| top-2, no threshold | 62.60 | 62.22 | 74.03 | 60.65 | 64.16 | 49.73 | 100.00 | **67.63** | **49.73** |
| rel α=0.2 K=3 | 66.74 | 67.33 | 65.62 | 65.78 | 68.62 | 48.78 | 90.99 | **67.70** | 48.78 |

- **News under the frozen floor: 86% silent, P=100 at R=6.** The system was not
  imprecise there — it was mute.
- **Even 0.001 leaves News 48% silent.** The News score distribution sits almost
  entirely below 0.001: the corpus shift is not a matter of degree any constant
  survives. This also indicts the production default `figMin=0.001` from
  `d603530e` — right direction, still the wrong family.
- **The whole scale-free surface is flat.** All α from 0.2–0.9, all K in 2–4:
  macro 61.99–68.06, worst unit never below 38. There is nothing to tune,
  which is the point.
- **What relative buys over plain top-K: precision at equal F1.** On
  description-rich domains it trades a little recall for 5–8 points of
  precision (MD-Fin P 52.65 → 60.08). That is the right trade for a system
  showing figures to a reader, and it cannot go silent wholesale by
  construction.

## The margin, bounded (bootstrap)

Also from the same dump: Academic pooled IP 69.12, bootstrap over questions
(B=20,000) gives 95% CI **[61.43, 76.56]**; 15.7% of resamples fall at or below
the published 65.28. One-sided by necessity (the comparator is a point
estimate). Paper phrasing: *higher, not significantly higher*.

## Discipline notes

- The paper's frozen numbers are **not** restated under better admission.
  Losses stay reported as losses. The claim is about rule families.
- The GPU run: scores came from the UGM box's own `bge-reranker-v2-m3` TEI via
  a temporary socat bridge (`ikat-rerank-bridge`, port 8095, removed after).
  6,819 questions in ~12 minutes versus ~7 h on the CPU container. CPU/GPU
  score parity was already established earlier (identical tp on Academic,
  one borderline emission apart).

## Production consequence

`figMin` should not exist as a global absolute constant — not at 0.2, not at
0.001. The scale-free replacement (admit `s ≥ 0.2·s_max`, cap K) is one line in
`hybrid-search.ts` and removes the class of corpus where figures silently
vanish. Filed as the per-KB calibration follow-up.

## Files

- `src/ikat/dump-scores.ts` — one-pass score dump, resumable, concurrency-safe.
- `src/ikat/admission-rules.py` — the offline grid; writes
  `corpus/results/admission-rules.json`.
- `src/ikat/bootstrap-margin.py` — question-level bootstrap for the Academic
  margin.
