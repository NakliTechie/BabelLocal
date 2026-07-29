# BabelLocal — IT2 engine upgrade

> **Goal**: when the user's source/target pair is `eng_Latn → {22 Indic languages}`,
> dispatch translation through AI4Bharat's IndicTrans2 200M (en→indic) instead of
> NLLB-200. Substantial quality lift on Indic, especially low-resource (Bodo,
> Dogri, Manipuri-Meitei, Santali, Sanskrit, Maithili). Everything else stays
> on the existing NLLB pipeline. Once both engines land, Anuvaad is redundant
> and can be retired.

Open this in a fresh Claude Code session at
**`/Users/chiragpatnaik/Code/Browser/BabelLocal/`**.

---

## Status of preceding work

A full IT2 browser pipeline has already been built and validated end-to-end —
that's the entire stack you need to lift. It currently lives at
[`/Users/chiragpatnaik/Code/Browser/Anuvaad/index.html`](../Anuvaad/index.html)
and runs at https://naklitechie.github.io/Anuvaad/.

**What's already validated and reusable verbatim**:

- ONNX export of IT2 200M en→indic (3 graphs: encoder, decoder, decoder_with_past) — bit-exact vs PyTorch on 528 fixtures.
  Published at [`naklitechie/indictrans2-en-indic-dist-200M-ONNX`](https://huggingface.co/naklitechie/indictrans2-en-indic-dist-200M-ONNX).
- Two BPE tokenizer.json files (src + tgt) + tokenizer_meta.json with the `>= dict_size → unk` remap. Same repo.
- Vanilla-JS port of `IndicTransToolkit.IndicProcessor` — 100% parity vs Python on 528 fixtures.
- Vanilla-JS BPE tokenizer for the dual-vocab IT2 setup.
- Greedy decode loop (encoder.run + decoder.run loop with full past_kv plumbing).
- IndexedDB-backed model caching (HF Hub returns fresh signed S3 URLs per request, so HTTP cache misses every visit; Cache Storage API rejects 280MB+ entries with `Unexpected internal error`; **IDB stores 500MB+ Blobs reliably**).

The full inlined pipeline is in [`../Anuvaad/index.html`](../Anuvaad/index.html)
— ~1500 lines, 53 KB. **Lift the relevant chunks verbatim into BabelLocal.**

---

## Target dispatch logic

```
if src == "eng_Latn" AND tgt ∈ {22 IT2 languages}:
    route → IT2 pipeline (Anuvaad's engine)
else:
    route → NLLB pipeline (BabelLocal's existing engine)
```

The 22 IT2 target languages (with script variants split = 25 entries) are
defined in `Anuvaad/index.html`'s `TARGET_LANGS` array.

**Out-of-scope for this session**:
- Indic → English (handled by a separate handoff at
  `prashnam-voice/browser-prep/HANDOFF-INVERSE-IT2.md`. After that lands, BabelLocal
  can also dispatch `{indic} → eng_Latn` to IT2-indic-en for additional quality.)
- Indic → Indic (not natively supported by IT2; could be done as
  `indic → eng_Latn → indic'` chained, but defer that).

---

## Architectural choices to make up front

### 1. One worker or two?

BabelLocal currently uses one Web Worker that loads NLLB via Transformers.js
`pipeline()` (see `index.html:646-708`). For IT2 we use raw onnxruntime-web,
not Transformers.js, because the IT2 architecture isn't in the Transformers.js
model registry and our hand-rolled decoder loop is parity-validated.

Options:

- **A: two workers**, one per engine. Routing happens on the main thread.
  Cleaner separation, easier to reason about. Slight overhead from two WASM
  contexts.
- **B: one worker** that loads both engines on demand and dispatches internally.
  Single state machine. More plumbing but probably the right long-term shape.

**Recommendation**: start with **A (two workers)** because it lets you copy
Anuvaad's worker pattern verbatim. Refactor to B later if the dual-context
overhead is real.

### 2. Lazy loading

Each engine (~600 MB NLLB q8, ~1.3 GB IT2 fp32) loads only on first use of its
route. A user who only ever does Spanish↔French never downloads IT2. A user
who only ever does English→Hindi never downloads NLLB.

**Show progress** for whichever engine is loading at the moment the user clicks
Translate. Don't pre-warm both at boot — the cumulative ~1.9 GB upfront would be
hostile to first-time users.

### 3. UI changes

The existing language picker in BabelLocal lists all 200 NLLB languages. Decision
points:

- **Keep one unified picker**, dispatch internally. User experience unchanged.
  Cleanest. Recommended.
- **Add a visible engine indicator** (e.g. a small badge: "powered by IT2 / NLLB")
  so power users know which model they're getting. Minor polish.
- **Optionally surface the quality lift**: when user picks an Indic target with
  English source, show a one-time tooltip "Using AI4Bharat IndicTrans2 for
  better Indic quality." Skip for v1.

---

## Lifting the IT2 pipeline from Anuvaad

The pieces, in priority order:

| What | Where in Anuvaad/index.html | Notes |
|---|---|---|
| `IT2_REPO`, `IT2_BASE` constants | line ~1212 | Update to point at your worker's structure |
| 25-language `TARGET_LANGS` array | line ~1215 | Filter BabelLocal's existing language picker against this set |
| `IndicProcessor` class (~470 LOC) | inlined script ~line 350-820 | Copy verbatim. Validated 100% parity. |
| `IT2Tokenizer` class (~225 LOC) | inlined script ~line 825-1200 | Copy verbatim. |
| `fetchWithProgress` + IDB cache helpers (~70 LOC) | line ~1287-1356 | Copy verbatim. |
| `loadModel()` for IT2 | line ~1362-1430 | Adapt for worker context |
| `greedyTranslate()` | line ~1435-1490 | The decoder loop. Copy verbatim. |
| `collectPresents()`, `greedyArgmax()` | line ~1492-1515 | Copy verbatim. |

The IndicProcessor + IT2Tokenizer + decoder loop together are ~1000 LOC of
parity-validated JS. Don't rewrite — copy.

---

## Step-by-step plan

1. **Recon**: read [`../Anuvaad/index.html`](../Anuvaad/index.html) end-to-end,
   identify the chunks above, understand the worker boundary in BabelLocal's
   current `index.html:646-708`.

2. **Add a second worker** (or expand the existing one) that loads the IT2 stack
   on first call. Worker exports a `translate(text, srcLang, tgtLang) → string`
   message contract identical to the existing NLLB worker.

3. **Add a router** on the main thread:
   ```js
   function pickEngine(srcLang, tgtLang) {
     if (srcLang === "eng_Latn" && IT2_LANGS.has(tgtLang)) return "it2";
     return "nllb";
   }
   ```
   `IT2_LANGS` is the 25-entry set from Anuvaad's `TARGET_LANGS`.

4. **Wire UI events through the router**. Existing `doTranslate()` calls
   `worker.postMessage({ type: 'translate', ... })`; replace with
   `routerPostMessage(...)` that dispatches to the right worker.

5. **Test parity**:
   - For an Indic target: result should match standalone Anuvaad output. Use
     the existing fixtures at
     `/Users/chiragpatnaik/Code/prashnam-voice/browser-prep/fixtures/expected_processed.json` —
     12 of those sentences translated to 11 Indic languages = 132 quick checks.
   - For a non-Indic target: result should match current BabelLocal NLLB output
     (no regression).

6. **Polish**:
   - Status badge differentiates "Loading IT2 model…" vs "Loading NLLB model…".
   - Progress bar shows the right total size for whichever route is active.
   - Maybe a subtle "(IT2)" / "(NLLB)" tag near the output language label.

7. **Deploy**: existing GitHub Pages auto-rebuilds. Verify the live site at
   https://naklitechie.github.io/BabelLocal/ after push.

---

## Open questions to raise at the start of the next session

1. **One worker or two?** (See architectural choice 1.) Recommendation: two.
2. **Visible engine indicator?** (Power-user-friendly. Or keep transparent.)
3. **Anuvaad retirement timing**: once this lands and is verified, archive
   `Browser/Anuvaad/` and update its README to point to BabelLocal? Or leave
   Anuvaad standing as the simpler en→indic-only demo for users who don't want
   the 600 MB NLLB extra?

---

## Suggested first-message handoff prompt

```
Read HANDOFF-IT2.md in this directory first — it has full context.

Goal: add the IT2 (AI4Bharat IndicTrans2) engine to BabelLocal so en→Indic
dispatches through IT2 (better Indic quality) while everything else stays
on NLLB. The full IT2 browser pipeline already exists at
/Users/chiragpatnaik/Code/Browser/Anuvaad/index.html — lift the validated
chunks verbatim (IndicProcessor port, IT2Tokenizer, decoder loop, IDB
caching). Both engines lazy-load on first use of their respective routes.

Plan: two workers (one per engine) + a router on the main thread that
dispatches by (srcLang, tgtLang). Test parity against
/Users/chiragpatnaik/Code/prashnam-voice/browser-prep/fixtures/expected_processed.json
for the Indic side.

Confirm the architectural choices in HANDOFF-IT2.md before writing code.
```

---

## Reference paths

```
BabelLocal (this folder):
  /Users/chiragpatnaik/Code/Browser/BabelLocal/
    index.html             — current single-file (917 lines, NLLB via Transformers.js)
    README.md              — model card / palette notes

Anuvaad (the IT2 pipeline to lift):
  /Users/chiragpatnaik/Code/Browser/Anuvaad/
    index.html             — full validated single-file IT2 pipeline (~1500 lines)
  Live: https://naklitechie.github.io/Anuvaad/

Source-of-truth JS port libraries (already validated 100% parity vs Python):
  /Users/chiragpatnaik/Code/prashnam-voice/browser-prep/js/
    indic_processor.js     — preprocess + postprocess + transliterate
    tokenizer.js           — BPE encoder/decoder for tokenizer.json
    indic_processor.test.js + tokenizer.test.js — Node-runnable parity tests

Test fixtures (use for parity validation of the IT2 route):
  /Users/chiragpatnaik/Code/prashnam-voice/browser-prep/fixtures/
    expected_processed.json   — 528 PyTorch baseline outputs, native script
    test_sentences.json       — 48 EN inputs

Published ONNX bundle (this is what BabelLocal will fetch):
  https://huggingface.co/naklitechie/indictrans2-en-indic-dist-200M-ONNX
```

---

## Memory entries to carry forward (from prior sessions)

- **Quality > size for browser bundles** — up to 5 GB fine, do not quantize at
  quality cost without asking.
- **GitHub repo path is `prashnam/prashnam-voice`** (not `chiragpatnaik/...`),
  for any links in READMEs or commits.
