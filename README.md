# 🌍 BabelLocal

A universal language translator that runs entirely inside your browser — no server, no API keys, no data leaving your device.

**[→ Try it live](https://nakliTechie.github.io/BabelLocal)**

## What it does

Translates text between 55 languages using Meta's NLLB-200 model, loaded directly into the browser via WebAssembly. The model downloads once (~600 MB), is cached locally, and works offline from that point on.

**Supported language families**

| Region | Languages |
|---|---|
| European | English, French, German, Spanish, Italian, Dutch, Polish, Czech, Greek… |
| Eastern European | Russian, Ukrainian, Serbian, Bulgarian, Belarusian, Croatian |
| Middle Eastern | Arabic, Hebrew, Persian, Turkish |
| Indian | Hindi, Tamil, Telugu, Bengali, Kannada, Gujarati, Marathi, Punjabi, Urdu, Odia… |
| East & SE Asian | Chinese (×2), Japanese, Korean, Thai, Vietnamese, Indonesian, Malay… |
| African | Swahili, Hausa, Yoruba, Amharic, Zulu, Somali |

## How to run

Serve the file from any static server:

```bash
python3 -m http.server 8080
# open http://localhost:8080/universal-translator.html
```

No build step. No dependencies. No backend.

---

## Why would you need this?

This is the more interesting question.

**Privacy.** When you paste text into Google Translate, DeepL, or any cloud service, that text travels to a third-party server and may be logged, stored, or used for model training. For casual use, that's usually fine. But what about a contract clause? A medical note? A confidential business document? A personal letter in another language? The moment you translate it via a cloud service, you've handed it to someone else. This tool keeps everything on your device — nothing is transmitted anywhere.

**Offline.** Download the model once. After that, it works on a plane, in a remote area, or anywhere without a reliable connection. Cloud translators stop working entirely when connectivity drops.

**No API costs or rate limits.** Production translation APIs charge per character — at scale, this adds up quickly. Local inference has no marginal cost and no usage cap.

**Sensitive environments.** Air-gapped systems, corporate intranets, and regulated industries (legal, medical, finance) often cannot send data to external APIs at all. A self-contained HTML file that runs locally is deployable anywhere, including places that have never seen the public internet.

**Auditability.** You can read every line of code in this project. There is no black box, no telemetry, no network calls beyond the one-time model download from Hugging Face.

The trade-off is honest: a ~600 MB one-time download, and quality that is good but not quite at the level of the top commercial services. For most real-world use cases — especially ones where privacy matters — it's a worthwhile exchange.

**The download is genuinely one-time.** The model is cached in your browser's storage — close the tab, shut down your laptop, come back a week later and it's still there, ready in a few seconds. No re-download. The only way it disappears is if you explicitly clear your browser's site data.

---

## Tech

- **[Transformers.js](https://huggingface.co/docs/transformers.js)** — runs Hugging Face models in the browser over WebAssembly
- **[NLLB-200-distilled-600M](https://huggingface.co/facebook/nllb-200-distilled-600M)** — Meta's open multilingual model, 200-language coverage, q8-quantised for browser use
- Zero build tooling. One HTML file.

## Palette

Coloured with **`china-02 · 米紙 MIZHI`** — hand-scroll mulberry paper, calligraphy. Warm cream body, indigo translation accent, brick-red brand mark.

Palette pulled from [**Rangrez**](https://github.com/NakliTechie/rangrez), the global colour-palette library that backs all NakliTechie projects.

---

## Part of the NakliTechie series

| Tool | What it does |
|------|--------------|
| **BabelLocal** | Offline translation — 200 languages, NLLB model |
| [**StripLocal**](https://github.com/NakliTechie/StripLocal) | EXIF metadata stripper — nothing leaves the browser |
| [**GambitLocal**](https://github.com/NakliTechie/GambitLocal) | Chess vs Stockfish — correspondence mode via URL |
| [**VoiceVault**](https://github.com/NakliTechie/VoiceVault) | Audio transcription — Whisper, offline-first |
| [**KingMe**](https://github.com/NakliTechie/KingMe) | English draughts — custom minimax AI, zero deps |
| [**SnipLocal**](https://github.com/NakliTechie/SnipLocal) | Background remover — RMBG-1.4, passport mode |
| [**Clacker**](https://github.com/NakliTechie/Clacker) | Split-flap display — browser-native, offline |
| [**Strait Command**](https://github.com/NakliTechie/strait-command) | Mine-clearing game in strategic waterways |
| [**Chokepoint**](https://github.com/NakliTechie/chokepoint) | Maritime tower defense — hold the strait |
| [**PredictionMarket**](https://github.com/NakliTechie/PredictionMarket) | Prediction market simulator — Parimutuel & LMSR |
| **PDFLOcal** | PDF editor — merge, split, rotate, annotate |
| **RangeLocal** | Missile range simulator — interactive globe & map |
| [**3D Tic-Tac-Toe**](https://github.com/NakliTechie/3d-tic-tac-toe) | 3D tic-tac-toe — 3×3×3 to 5×5×5, minimax AI |

---

**Built by [Chirag Patnaik](https://github.com/NakliTechie)**

---

*Built in a few minutes with [Claude](https://claude.ai).*
