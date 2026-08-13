# The human audit: 80% of our gold does not track human judgement

Run 2026-08-10, one annotator, 48 items x 8 candidates = 384 binary decisions.
The annotator saw only the question and the crops — no anchor, no position, no
page, no description, no caption.

## Agreement between our gold and a person

| subset | Cohen's kappa |
|---|---|
| clean gold (`figure_dependent`) | **0.631** |
| **anchor-derived gold** (80% of links) | **0.092** |
| overall | 0.373 |

A kappa of 0.09 is agreement at chance. On the four-fifths of our gold standard
defined as "figures whose anchor falls in the gold span", the standard is close
to unrelated to what a person calls a correct figure.

This is stronger than the confound recorded in `06`. There the finding was that
the gold favoured one mechanism. Here it is that the same portion of the gold
barely measures the task at all.

Volume disagrees too: our gold marks 48 correct figures across these items, the
human marks 19. The human answered "no figure fits" on 30 of 48 items; our gold
does so on 14.

## Systems scored against human gold

| system | P | R | F1 |
|---|---|---|---|
| `co_embed` (descriptions) | 0.076 | **0.579** | **0.135** |
| `anchor_hybrid` | 0.064 | 0.368 | 0.109 |
| `anchor` | 0.052 | 0.263 | 0.087 |
| `caption_match` (production) | 0.028 | 0.211 | 0.049 |

The ordering inverts. The anchor, reported all week as the contribution, places
second from last among real systems; description similarity, which this project
declared the wrong instrument only hours earlier, leads.

## What this does and does not establish

**Does:** our aggregate selection conclusions are not supported by human
judgement, and the direction of the anchor's advantage reverses under it. That
is enough to stop those claims going into a paper.

**Does not:** establish the new ordering. One annotator, 48 items, 19 positive
links. Data this small can falsify; it cannot confirm. `co_embed` must not be
reported as the winner on this basis.

Every absolute number here is poor — best precision 0.076 — which is its own
finding: measured against a person rather than against our own construction, no
system we have selects figures well.

## Why the questions themselves are also wrong

Raised by the annotator, and correct. A sample from our set:

> *"Berapa banyak angka yang tertera pada dial timbangan duduk tersebut?"*

No pupil asks that. Real questions are conceptual — *"apa itu gaya gesek?"*,
*"bagaimana Yesus memberi makan 5000 orang?"*

The cause is structural. `figure_dependent` questions were produced by showing a
model one figure's context and asking for something answerable *only* from that
figure. That procedure manufactures "name what is visible" questions, which
flatter description-matching for the trivial reason that both describe picture
content.

So the two defects hide each other:

- span questions — gold near-arbitrary (kappa 0.09), questions reasonable
- figure-dependent questions — gold sound (kappa 0.63), questions unrealistic

No split of the current benchmark is simultaneously well-grounded and
representative.

## What has to happen

1. **Regenerate questions from the curriculum, not from figures.** Ask what a
   pupil studying this chapter would ask, with no figure in view. Only then
   decide, separately, which figure belongs.
2. **Human gold at usable scale.** 48 items with one annotator was enough to
   invalidate; several hundred with 2-3 annotators is needed to establish
   anything, and inter-annotator kappa must be reported first.
3. **Re-examine every selection claim** in the manuscript. Placement claims are
   unaffected: they hold selection fixed, so this bias cancels between the
   systems compared.
