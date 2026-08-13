# Shipping the VLM figure gate

The benchmark's best selector is now in the product code path
(`src/lib/rag/figure-gate.ts`, wired at the end of `fetchMatchingFigures`).
This is what an operator needs to turn it on, and what to watch afterwards.

## What changes for a student

| | figures shown | of those, correct |
|---|---|---|
| today (`anchor_hybrid`) | many | **2.8%** |
| with the gate | about half as often | **54.2%** |

Measured on the 48 human-annotated questions — the only gold no VLM produced.
Per 100 questions: roughly 50 get no figure, 27 get a correct one, 23 get a
wrong one. The system declining half the time is deliberate: a wrong diagram
costs a student more than a missing one.

## Switching it on

```bash
KB_FIGURE_VLM_ENABLED=1
KB_FIGURE_VLM_BASE=http://ollama:11434/v1          # any OpenAI-compatible vision endpoint
KB_FIGURE_VLM_MODEL=hf.co/mradermacher/Qwen-SEA-LION-v4-4B-VL-GGUF:Q8_0
# optional, defaults are the measured operating point:
KB_FIGURE_VLM_TOPN=2          # candidates the model is allowed to see
KB_FIGURE_VLM_MAX=1           # figures it may actually show
KB_FIGURE_VLM_TIMEOUT_MS=8000 # per-call budget — see "cold starts" below
```

Unset any of the first three and the gate does not run at all.

## Which model

Both were measured on 165 questions at the same operating point:

| | F1 | added latency |
|---|---|---|
| SEA-LION 4B VL | 0.786 | ~1.4 s |
| SEA-LION 8B VL | 0.800 | ~2.1 s |

**Start with the 4B.** It costs 0.014 F1 and runs 35% faster, and on the
partner's GPU the gate shares hardware with generation — a slower gate does not
just delay the figure, it competes with the answer. Move to the 8B only if
figure quality turns out to matter more than reply latency in real use.

## Cold starts decide the timeout

Measured from inside the UGM app container against its own ollama:

| | per call |
|---|---|
| warm | **240 ms** |
| cold (model load) | **4499 ms** |

ollama is not given `OLLAMA_KEEP_ALIVE`, so it unloads a model after five idle
minutes. At the 4000 ms budget the *first question after any quiet period* would
exceed it, fail closed, and silently lose its figure — on a pilot box, that is
most questions. Hence **8000 ms**. Since the warm path is 240 ms, the larger
budget is only ever spent when something is actually wrong.

The alternative fix is `OLLAMA_KEEP_ALIVE=24h` on the ollama container, which
trades permanent VRAM for a never-cold gate. Worth doing if the box is ever put
under real student load; not worth it while it is idle most of the day.

## Cost and failure behaviour

The gate adds `topN` vision calls per question that has any figure candidate —
about 0.25 s wall-clock warm at 4B, since the two calls run concurrently. Questions
with no candidate cost nothing.

It is placed last in the pipeline and can only *remove* figures. If the endpoint
is down, slow, misconfigured, or the crop cannot be fetched from storage, the
affected figure is dropped and the answer proceeds. There is no configuration in
which the gate makes the output worse than today's — the worst case is today's
output minus some figures.

## What to watch

Every gated question logs one line:

```
[RAG] figure gate (model, topN=2): YA:Gambar 2.1 siklus air | no:Foto anak … -> 1/4
```

Two things are worth alerting on:

- **Everything gated to 0.** If survivors are near-zero across many questions the
  endpoint is probably failing closed rather than judging; check for the fetch
  and non-2xx warnings beside it.
- **Nothing ever gated.** If `-> n/n` always holds, the prompt is not reaching
  the model, or the model is not reading the image. Expect roughly half of
  candidates to be rejected.

## Deployed

2026-08-13, UGM stack 28, image `shirologic/rantai-cloud:figgate` (arm64).
The `app` service also had to join `rantai-llmops_llmops`: ollama lives on that
network and does not publish 11434 to the host, so without it every gate call
would fail closed and *all* figures would disappear. Connectivity was verified
from inside the container before the flag was turned on, in that order and for
that reason.

Rollback is one variable: `KB_FIGURE_VLM_ENABLED=0` and update the stack. The
image does not need changing — the code returns to its previous behaviour
exactly.

## The gate ran zero times until a real chat was watched

Deployed, verified, and dead. Every check passed — the env was set, the app
could reach ollama, the model answered, the container was healthy — and the gate
still never executed, because a floor upstream of it discarded every candidate
before it could look at anything.

On a live question about the human digestive system, the cross-encoder ranked the
candidates *correctly*:

```
0.016  Gambar 1.15 Posisi usus…
0.005  Gambar 1.14 Struktur d…
0.002  Gambar 1.13 Lokasi Empedu…
```

The admission floor was 0.2. Nothing survived, so `final` was empty, so the gate
was skipped, so no `[RAG] figure gate` line was ever written. The absence of the
line was the only symptom, and absence is exactly what nobody notices.

Two things made this invisible:

- **The default was tuned on the wrong text.** 0.2 came from descriptions, where
  bge-reranker-v2-m3 scores relevant figures 0.5–0.8. Production ranks the printed
  caption — far shorter, often truncated — and the whole distribution collapses.
- **The log lied.** It printed the raw environment variable, so an unset
  `KB_FIGURE_MIN_RERANK` read as `floor=off` while a 0.2 floor was silently
  dropping everything.

Both are fixed in code (default 0.001; the log now prints the effective floor and
the survivor count). After the change, the first real gate line:

```
[RAG] figure gate (…4B-VL…, topN=2): YA:Gambar 1.15 Posisi usu | YA:Gambar 1.14 Struktur d -> 1/5
```

Five candidates admitted, the cross-encoder cut to two, the VLM approved both,
one shown. The right one.

**The lesson is about verification, not about the number.** Every component was
checked in isolation and every check passed. What was never checked was whether
the stage ran at all on a real question — and that is the only check that would
have caught it.

## The honest limits of the 54.2%

- **n = 48, 19 positive links.** Run-to-run noise at this sample size is ±0.014
  F1, so treat differences smaller than that as unresolved.
- Those 48 questions were drawn from questions that *have* figure candidates.
  Real traffic carries many questions with no figure dimension at all, where the
  gate will simply stay silent — so field precision may differ in either
  direction.
- **There is still zero online signal.** No measurement of this pipeline exists
  from real use. The log line above is the first one.
