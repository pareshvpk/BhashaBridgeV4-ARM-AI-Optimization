# Model assets — how to obtain and stage them

The model binaries are **deliberately not committed to git** (~632 MB of ONNX graphs, shared weight
blobs, SentencePiece vocabularies, and Vosk acoustic models). Committing them once would put them in
history permanently. This file tells a builder exactly what to put where so the app compiles into a
working APK.

> If you only want to *run* the app, install a prebuilt APK from the repository's **Releases** page —
> those already contain everything below. This file is for building from source.

## Where the assets go

Everything below lives under `app/src/main/assets/`:

| Asset | What it is | Produced by |
|---|---|---|
| `encoder_int8.onnx` | EN→HI encoder graph (structure only) | `model_pipeline/cached_export.py` → `quantize_cached.py` |
| `decoder_init_int8.onnx` | EN→HI first-step decoder | same |
| `decoder_step_int8.onnx` | EN→HI cached decode step | same |
| `hi_en_encoder_int8.onnx` | HI→EN encoder | same pipeline, indic-en checkpoint |
| `hi_en_decoder_init_int8.onnx` | HI→EN first-step decoder | same |
| `hi_en_decoder_step_int8.onnx` | HI→EN cached decode step | same |
| `weights.bin` | shared weight blob for EN→HI (must sit beside its graphs) | `model_pipeline/dedup_weights.py` |
| `hi_en_weights.bin` | shared weight blob for HI→EN | same |
| `dict.SRC.json`, `dict.TGT.json` | EN→HI vocabularies | IndicTrans2 checkpoint |
| `dict.SRC_HI.json`, `dict.TGT_EN.json` | HI→EN vocabularies | IndicTrans2 checkpoint |
| `model/` | Vosk small English (Indian) acoustic model | Vosk model zoo |
| `model-hi/` | Vosk small Hindi acoustic model | Vosk model zoo |

**Note on the graphs + blob split:** each `*_int8.onnx` graph carries *structure only*; its weights
live in the shared `.bin` blob beside it. A graph without its blob will not load.

## Two ways to get the assets

### Option A — download the prebuilt asset bundle (fastest)
1. Download the `assets.zip` bundle from the repository's [Releases page](https://github.com/pareshvpk/BhashaBridge---ARM-AI-Optimization/releases/latest).
2. Unzip it so its contents land directly in `app/src/main/assets/`.
3. Build (see `README.md` → Build).

### Option B — regenerate from the source checkpoints (fully reproducible)
Requires an authenticated Hugging Face login with access to the gated AI4Bharat checkpoints
(`ai4bharat/indictrans2-en-indic-dist-200M` and `ai4bharat/indictrans2-indic-en-dist-200M`).

1. `huggingface-cli login`
2. Follow `model_pipeline/EXPORT_WITH_CACHE.md` to run `cached_export.py` → `quantize_cached.py` →
   `dedup_weights.py`.
3. Verify the export before integrating: `python model_pipeline/verify_cache.py --onnx-dir <dir>
   --direction en_hi` — all seven numeric checks must pass (greedy tokens identical to fp32).
4. Copy the outputs into `app/src/main/assets/` as in the table above.
5. Download the two Vosk models from the Vosk model zoo into `model/` and `model-hi/`.

## Licensing reminder
The models keep their own licenses (IndicTrans2: MIT; Vosk models: Apache-2.0). Redistributing an APK
carries those obligations. See `THIRD_PARTY_NOTICES.md`.
