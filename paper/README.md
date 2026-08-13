# Manuscript

IEEEtran, single source file. Every bibliography entry was checked against its
arXiv or publisher listing page on 2026-08-13; none are marked unverified.

```bash
make          # → main.pdf
```

Requires [Tectonic](https://tectonic-typesetting.github.io). No local TeX
distribution is needed — it downloads the packages this document actually uses
and caches them.

## What the paper claims, and what it does not

The method is a join over data the layout parser already produced. There is no
model in it, nothing is trained, and nothing is tuned per corpus, so the claim
is not scoped to a language or a document genre. The headline comparison is on
**MRAMG-Bench's Academic subset** — English arXiv documents whose gold was
authored by that benchmark's authors, not by us.

The Indonesian textbook corpus is the *stress case*, not the scope: a fifth of
its figures carry a printed caption, so a caption-matching pipeline fails there
in a way that caption-rich corpora hide.

## Before camera-ready

- The author block in `main.tex` and `../CITATION.cff` is unfilled.
- The human-annotated selection result rests on `n = 48`. That sample size is
  the binding constraint on the headline number; see `../RESULTS.md`.
