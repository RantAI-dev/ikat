# Research notes

These are the working records the paper was assembled from. They are kept in the
repository rather than summarised away, because several of them document claims
that were **withdrawn** — and a retraction is only useful if the reasoning that
produced the original claim is still visible.

| file | |
|---|---|
| `02-benchmark-design.md` | corpus, question construction, adversarial filtering |
| `06-gold-standard-confound.md` | why a model-built gold inflates a model-containing pipeline |
| `07-human-gold-audit.md` | κ against a human annotator; the harness gold agrees at chance |
| `08-cross-encoder-selection.md` | the cross-encoder as selector rather than gate |
| `09-mramg-sota.md` | external validation, including a near-miss misattribution |
| `10-improvement-experiments.md` | eight experiments, two of which failed because of us |
| `11-vlm-gate-deployment.md` | shipping the gate, and its zero executions |
| `12-prediction-powered-selection.md` | a judge cannot replace the annotator at this scale, and why |
| `13-mramg-all-subsets.md` | all six MRAMG subsets — one win, one loss, three that measure nothing |

Two entries worth reading for the method rather than the result:

**`06`** measures the circularity directly — the same selector scores 0.521 against
a VLM-built gold and 0.283 against human annotation. Any paper that reports the
first number without the second is reporting agreement with a reference model, not
correctness.

**`10`** contains two corrections of our own reasoning. One number was declared
unreproducible and a whole paragraph written about contended GPUs; the actual
cause was an environment variable we had not recorded alongside the result. The
withdrawal is itself withdrawn, in place, with the reasoning left intact.

**`12`** is the newest and the least conclusive on purpose. A rectified estimator
would let judge labels stand in for annotation without inheriting the judge's
bias — but it is valid only if the doubly-labelled pairs are a random subsample,
and ours are enriched 2.46x because every gold figure lands in both candidate
pools by construction. The script measures that and refuses to report. What it
buys is a better problem: the remaining work is compute, not annotation.
