# 16 — A second vendor's judge: one exam failed, one conclusion strengthened

**Question.** The judged half of the end-to-end result rested on one judge
model, and the selection judge's κ = 0.556 is a per-model credential. What does
an independent vendor's frontier model (MiniMax-M3) change?

**Answer.** Two things, in opposite directions. On the *selection-judging*
exam it fails: κ = 0.362 at 2.76× permissiveness on the identical 360
human-labelled pairs — so it was **not** licensed as a gate or selection judge,
and the paper's "κ does not transfer" claim now has cross-vendor evidence. On
the *end-to-end judged metrics* it strengthens everything: the generator's
selection advantage replicates, and placement moves from a tie to a win for
the anchor join. Under neither judge does the generator place better than the
join.

---

## The licence exam (κ, images inline)

Same manifest, same 165 pools, same crops, the annotator's Indonesian
instruction verbatim. MiniMax-M3 reads images natively on the
`api.minimax.io/v1` endpoint (the M-series is multimodal; there is no separate
VL model on this key).

```
kappa=0.362  both-yes=13 judge-only=34 human-only=4 both-no=309
judge says yes 47x, human 17x — permissiveness 2.76x
(incumbent, same exam: kappa=0.556, permissiveness 1.24x)
```

The representativeness gate still passes (0.95×) — the pools are the
annotator's own — so this is a clean instrument comparison: **same exam, same
answers sheet, different model, κ drops by 0.19**. Consequence enforced: no
MiniMax gate run on MMDocRAG, per the pre-stated rule that the gate is
licensed by κ. The refusal is the result.

## The second judge on the end-to-end manifest (text-only rubric)

All 400 blind items re-scored, same rubric, same neutral keys.

| metric | judge 1 (A−B) | judge 2 / MiniMax (A−B) | agree? |
|---|---|---|---|
| relevance | +0.306 [+0.145,+0.478] | +0.290 [+0.113,+0.462] | yes — A wins twice |
| effectiveness | +0.242 [+0.081,+0.403] | +0.199 [+0.027,+0.376] | yes — A wins twice |
| comprehensive | +0.085 [−0.035,+0.210] | +0.115 [−0.005,+0.235] | yes — tie twice |
| **position** | −0.012 [−0.085,+0.061] tie | **−0.087 [−0.162,−0.012] B wins** | B never loses |

Inter-judge reliability: Pearson r = 0.79 / 0.73 / 0.70 / 0.64 per metric,
MiniMax uniformly ~0.4 points stricter — a threshold offset, not a different
ordering, which is exactly what its permissive-failure on the κ exam predicts.

**Combined claim for the paper (conservative form):** under neither judge does
the generator place better than the mechanical join; under the second judge
the join is strictly better (56 questions to 36).

## Files

- `src/ikat/minimax-judge.py` — the κ exam; writes `judge-labels-minimax.json`
  in the incumbent's schema; `ppi-eval.ts` consumes it via `IKAT_JUDGE_LABELS`.
- `src/ikat/minimax-judge-endtoend.py` — second judge over the blind manifest;
  resume-safe; writes `judge-verdicts-minimax.json`.
- Gotcha recorded: the `.env` line carries a trailing comment after the key —
  strip at whitespace or every request 401s with MiniMax error 1004.
