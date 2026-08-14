# BhashaBridge V4 — Project Write-Up

**Offline, on-device English ⇄ Hindi translation and speech for Arm client devices.**

|  |  |
|----|----|
| **Hackathon** | Arm AI Optimization Challenge — Mobile AI (Track 1: optimization output) |
| **Platform** | Android (Arm64 / `arm64-v8a`), `minSdk 24`, `targetSdk 36` |
| **Runtime & APIs** | ONNX Runtime 1.27.0 (CPU EP / MLAS) · Arm KleidiAI micro-kernels · XNNPACK (probed) · Vosk 0.3.47 |
| **Model** | IndicTrans2 distilled 200M, re-exported with KV-cache and quantized to INT8 |
| **Repository** | Public, open-source — **MIT License** (© 2026 V Paresh Kumar) |
| **Package** | `com.bhashabridge.app` |

------------------------------------------------------------------------

## 1. Project Overview

**What it is.** BhashaBridge is a translator that works when the network doesn’t. It runs a full neural machine-translation model **and** a speech pipeline (speech-to-text and text-to-speech) entirely on the phone’s Arm CPU. Speak or type in English and get Hindi; speak or type in Hindi and get English — with the radio switched off and no data ever leaving the device. The offline guarantee is not a promise in a settings screen; the app declares **no** `INTERNET` **permission at all**, so it is enforced by the Android manifest.

**Why it exists.** The situations where a translator matters most — a clinic counter, a roadside emergency, a border crossing, a low-cost phone on a weak network — are exactly the ones where round-tripping to a cloud service fails: latency, cost, dead zones, and the privacy cost of sending someone’s medical or legal words to a third party. So the design brief was uncompromising: the model must fit and run on an ordinary phone’s CPU, answer fast enough to feel conversational, survive offline, and never make a request it doesn’t have to.

**What makes it interesting, and why it should win.** The story is not “we built a translation app” — it is *how a model that could not run well on-device was made to.* The IndicTrans2 decoder, as originally exported, shipped **with no key/value cache**: every generated token re-attended the entire growing prefix, making decoding **O(n²)** in sentence length — the worst possible shape for the long sentences that matter. BhashaBridge re-exports the model as a **three-graph, KV-cached pipeline** (O(n)), quantizes it to INT8 with a **verified numeric parity gate**, and makes the runtime **Arm-capability-aware** — reading the CPU’s features at launch and dispatching the right integer kernel, deriving its thread count from the core topology, and toggling Arm’s KleidiAI/SME kernel by *measurement*. Every one of those decisions is backed by on-device numbers across **nine devices, four silicon vendors, spanning Armv8.0 to ARMv9**, published as raw data with a strict provenance rule. It is a complete, runnable app whose real deliverable is an **optimization method that transfers** — the exact spirit of an *optimization* challenge.

------------------------------------------------------------------------

## 2. Functionality / Output

**What the user can do:**

- **Type or speak, both directions** — EN→HI and HI→EN, each running from verified cached INT8 graphs.
- **Live speech** — Vosk on-device recognition with a running waveform, streaming translation of partial results, then spoken output through the system TTS voice.
- **Emergency phrases** — 32 human-translated pairs across four categories, served instantly with no model on the path, for the case where a translation must be exactly right and immediate.
- **Audio import** — transcribe and translate a recorded file.
- **History**, **app-language toggle** (English / हिंदी), and a **Model & device panel** that reports the detected CPU capabilities and the ONNX Runtime policy derived from them.

**The tangible outputs of the project (the “artifacts”):**

1.  **An optimized, on-device model** — IndicTrans2 200M re-exported as `encoder` / `decoder_init` / `decoder_step` INT8 graphs with a shared de-duplicated weight blob per direction (**1869 MB fp32 → 472 MB INT8**, 3.96×; artifact footprint cut a further 31% by weight de-duplication).
2.  **A reproducible optimization + validation method** — a scripted export→quantize→verify pipeline with a two-mode numeric gate, and a benchmark that ships inside the app and emits a machine-readable record per run.
3.  **A nine-device measurement campaign** — raw JSON/CSV per device plus written reports, showing the *same APK* scale from **50.3 → 412.8 tokens/sec** across Armv8.0 → ARMv9 with **no recompile**.

------------------------------------------------------------------------

## 3. How it works (technical implementation)

**The engine.** A distilled 200M IndicTrans2 model, INT8, executed by ONNX Runtime on the CPU execution provider. The core change is the export:

- `encoder` runs once per sentence and produces the hidden states.
- `decoder_init` performs the first decode step and emits the initial key/value cache.
- `decoder_step` performs every later step, consuming the previous cache and appending one row.

The full multi-tensor attention cache is flattened to *named* ONNX inputs/outputs so it survives the graph boundary, turning O(n²) decoding into **O(n)** — a flat per-step cost. This is the backbone of the project and the source of its largest win.

**Arm capability-awareness.** The same INT8 model dispatches a different hand-optimized integer kernel depending on what the CPU exposes at runtime — plain NEON, then MLAS’s dot-product (SDOT/UDOT) kernel, then MLAS’s i8mm matmul kernel, and on SME silicon **Arm KleidiAI’s SME int8 kernel**. That last path was confirmed *at the instruction level*: profiling a live run with `simpleperf` showed the single hottest span of code in the whole app disassembling to KleidiAI’s `smopa` (signed-8-bit outer-product) SME instruction — proof the acceleration executes, not merely compiles in.

**Thread policy derived, not hard-coded.** These INT8 GEMMs are latency-bound, so over-threading *regresses* latency. The runtime reads the performance/efficiency core split from `/proc/cpuinfo` and cpufreq, computes an intra-op thread count, and clamps it to a small measured bound — validated to select correctly from 4-core big.LITTLE parts up to an 8-core uniform-IP flagship.

**Execution providers probed, not assumed.** On device, XNNPACK claimed **zero graph nodes** (silently folding back to MLAS), and the NPU-backed provider ran **2.3× slower** than tuned CPU — so the app ships the CPU/MLAS path deliberately, backed by node-placement counts.

**Quantization proven.** The INT8 model produces **greedy token sequences identical to the fp32 reference** (max logit delta 0.448), verified before any device integration by a seven-check gate with an absolute stop rule.

**Architecture.** `MainActivity`/`WelcomeActivity` → `TranslateViewModel` → `BhashaBridgeApp` (process-scoped owner of every native resource) → `MtEngine` (the three INT8 graphs). No Activity can reach the runtime; native resources are owned once, at process scope, with a single release trigger — rules written down and enforced in `docs/ARCHITECTURE_RULES.md`.

------------------------------------------------------------------------

## 4. Measured results (optimization output)

Every optimization was measured on a benchmark that ships in the app; the earlier version’s own graphs were re-run through the same harness so the comparison is apples-to-apples.

| What changed | Before (v3.4.1) | After (V4) | Effect |
|----|----|----|----|
| Decode complexity | O(n²), cache dropped | **O(n)**, three-graph cache | Per-token throughput goes from *falling* (13.9→9.5 tok/s) to *rising* (14.9→21.6); up to **2.12× faster** at 12 tokens on identical hardware, widening with length |
| Model size | 1869 MB fp32 | **472 MB INT8** | 3.96× smaller, greedy output identical |
| Shipped assets | 909 MB (weights duplicated across graphs) | **619 MB** | −31% by pointing both decoders at one shared blob, output bit-identical |
| Cold start to first translation | ~27,000 ms | **~5,100 ms** | 5.3×; ~49% of the old startup was a character-at-a-time JSON parser, not the model |
| Latency stability (stdev, 12 tok) | 93 ms | **18–23 ms** | ~−80% jitter |
| Process memory | ~983 MB (arena on + a per-rotation native leak) | **~605–670 MB** (arena off, leak fixed) | lower, and the leaked native heap now returns 557.8→13.2 MB on release |
| Long-input correctness | 5/16 sentences truncated (31%) | **0/16** | fixed a hard 18-step cap |
| Portability | 1 device, ORT 1.17.1, 2 tests | **9 devices, four vendors, ORT 1.27.0, 94 tests** | same APK: **50.3 → 412.8 tok/s**, Armv8.0 → ARMv9, no recompile |

------------------------------------------------------------------------

## 5. What was significantly updated during the submission period

The project began the period as a v3.4.1 prototype whose translation engine did not use a KV cache and had never been benchmarked, tuned, or validated beyond a single device. The following were built during the submission period and constitute the substance of this entry:

- **Re-architected the model export** from a cache-less O(n²) decoder to a three-graph KV-cached O(n) pipeline, with a flattened attention cache as named ONNX I/O.
- **Built the export→quantize→verify pipeline** (`model_pipeline/`), including the two-mode numeric parity gate and shared-weight de-duplication.
- **Made the runtime Arm-capability-aware** — CPU feature detection, topology-derived thread policy, and a measured KleidiAI/SME toggle.
- **Ran a nine-device measurement campaign** with a shipped benchmark, a strict provenance rule, and a published negative-results ledger.
- **Fixed a class of correctness/lifecycle defects** carried from v3.4.1 (silent truncation, a native use-after-free, a per-rotation memory leak) and added a 94-method test suite.

------------------------------------------------------------------------

## 6. Setup Instructions — build, run, and validate on Arm

**Prerequisites** - JDK 17 (e.g. the JBR bundled with Android Studio) - Android SDK **36** - The Gradle wrapper in the repo (Gradle 9.5 — no separate install needed) - An Arm device (recommended, `arm64-v8a`, API 24+) or an emulator; a physical device is required to reproduce the performance numbers

**Step 1 — Clone**

    git clone https://github.com/pareshvpk/BhashaBridge---ARM-AI-Optimization.git
    cd BhashaBridge-V4

**Step 2 — Stage the model assets** (they are **not** in git — ~619 MB) Place the following into `app/src/main/assets/`. Each graph carries structure only; its weights live in a shared blob that must sit beside it, or the graph will not load:

| Asset | Produced by |
|----|----|
| `encoder_int8.onnx`, `decoder_init_int8.onnx`, `decoder_step_int8.onnx` (EN→HI) | `model_pipeline/cached_export.py` + `quantize_cached.py` |
| `hi_en_encoder_int8.onnx`, `hi_en_decoder_init_int8.onnx`, `hi_en_decoder_step_int8.onnx` (HI→EN) | same pipeline |
| `weights.bin`, `hi_en_weights.bin` | `model_pipeline/dedup_weights.py` |
| `dict.SRC.json`, `dict.TGT.json`, `dict.SRC_HI.json`, `dict.TGT_EN.json` | IndicTrans2 vocabularies |
| `model/`, `model-hi/` | Vosk small English (Indian) and Hindi models |

To regenerate the ONNX graphs from scratch, follow `model_pipeline/EXPORT_WITH_CACHE.md` (requires an authenticated `huggingface-cli login` with access to the gated AI4Bharat checkpoints).

**Step 3 — Build / install**

    export JAVA_HOME="/path/to/Android Studio/jbr"
    ./gradlew assembleDebug           # build the APK
    ./gradlew installDebug            # or install to an attached device

**Step 4 — Run** Launch the app, pick a direction, and type or tap the mic. Try an emergency phrase for an instant, model-free path. The **Model & device** panel shows the detected CPU and the ORT policy chosen for it.

**Step 5 — Validate / reproduce the numbers** - **Unit + instrumented tests:** `./gradlew test` and `./gradlew connectedAndroidTest`. - **On-device benchmark:** the `:benchapp` module is a standalone Arm CPU smoke test that runs the inference workload and prints a machine-readable `REPORT_JSON` record (schema `bb-bench/1`) to logcat — startup, per-sentence latency, per-graph stage timings, thread sweep, kernel A/B, and memory. Cross-device raw results are checked in under `bench/results/cross-device/`.

------------------------------------------------------------------------

## 7. Third-party integrations, licensing, and authorization

All third-party components are used within their licenses, and the project builds on top of them rather than merely repackaging them:

| Component | Version | License | Use |
|----|----|----|----|
| ONNX Runtime (Android) | 1.27.0 | MIT | Executes the three graphs; MLAS supplies INT8 kernels |
| Arm KleidiAI | vendored in ORT 1.27.0 | Apache-2.0 | Arm micro-kernels compiled into MLAS; reached on SME cores |
| Vosk | 0.3.47 | Apache-2.0 | On-device speech recognition |
| kotlinx.coroutines / AndroidX / Material | see `gradle/libs.versions.toml` | Apache-2.0 | Platform + concurrency |
| IndicTrans2 distilled 200M | AI4Bharat checkpoint | MIT | Re-exported + quantized; shipped graphs are derivatives |
| Vosk small EN-Indian & Hindi models | Alpha Cephei | Apache-2.0 | Bundled recognition models |

Model binaries are not committed to the repository; they are staged locally at build time. Redistribution of an APK carries the model licenses above. Full details in `THIRD_PARTY_NOTICES.md`. The project’s own source is **MIT**-licensed, detectable at the repository root.

------------------------------------------------------------------------

## 8. Reusable artifacts (what other developers can take away)

- A **model-agnostic optimization & validation template**: the two-mode export gate, the benchmark schema with its provenance rules, and an ordered optimization checklist (cache the decode state, quantize with a parity gate, derive threads from topology, turn the arena off, toggle the accelerated kernel by measurement, probe execution providers, cache the optimized graph, de-dup shared weights).
- A **reproducible benchmark** any ONNX-Runtime mobile project can adopt to make its numbers defensible.
- A **worked example of KleidiAI/SME and XNNPACK on real Arm devices**, including how to confirm at the instruction level that SME actually executes.

------------------------------------------------------------------------

## 9. Known limitations (stated plainly)

- **~619 MB of assets** — side-loading works today; Play would need asset delivery or a first-run download.
- **~5 s from launch to first translation** — down from 27 s, still the worst user-facing number.
- **Release builds not yet shippable** — debug-signed, R8 disabled, `versionCode` not incremented.
- **Portrait-only** by deliberate choice (the landscape layout was unusable and is locked, not left broken).
- **Speech WER and TTS latency unmeasured** — the bundled Vosk Hindi model publishes 14.96–39.08% WER by test set.

None of these touch the translation engine that is the core of the work; all are recorded rather than papered over.

------------------------------------------------------------------------

## 10. Why it should win

BhashaBridge V4 takes a model that *could not* run well on a phone and makes it run well — on the Arm CPU, offline, in under a tenth of a second on modern silicon — and it proves every step on real hardware across twelve devices with a benchmark that ships in the box. It leverages Arm precisely (runtime capability detection, KleidiAI/SME dispatch confirmed at the instruction level, topology-aware threading), it is honest about what it measured and what it did not, and it leaves behind a transferable optimization method rather than a one-off app. For an *optimization* challenge, that is the assignment, executed and evidenced.
