# BhashaBridge — the development story, v1 → v4

A framing note first: this repo's own commit log and docs only ever name two eras —
**v3.4.1** (the legacy app, kept out-of-repo as a measurement control) and **V4** (everything here,
~140 commits, 2026-07-20 → 2026-08-13). There was never an official "v1/v2/v3" tag. What follows
groups V4's real phases into four chronological stages and narrates them as v1 → v4, because the
work genuinely has four distinct centers of gravity even though nobody labelled them that way at the
time. Every number below cites the committed file it came from — mainly `docs/OPTIMIZATION_SUMMARY.md`
(the ledger), `docs/FINAL_REPORT.md` (the narrative pass), and `docs/LESSONS_FROM_V3.md`.

---

## v0 — what v1 inherited

v3.4.1 worked: it translated, offline, on a phone. It is not being mocked, but its defects are the
reason V4 exists, and they were structural, not careless:

- `MainActivity` was 961 lines — it owned both translators, both loading states, direction state,
  the streaming-partial gate, three executors, the drawer, history, and audio import.
- `OnnxSessionManager.release()` was correct code with **zero call sites**. Every screen rotation
  leaked ~639 MB of native heap and re-paid an 8.6 s model load.
- The exported decoder had no KV-cache ports — every generated token re-attended the entire prefix
  (O(n²) decode).
- Threads were hard-coded `intraOp=4, interOp=2` with a comment claiming they were "tuned against
  real device measurements" — they weren't; the same values later measured 719 ms / σ 88.8 against
  V4's derived 667 ms / σ 18.4 on the hardware they were supposedly tuned for.
- 1 device validated, 2 `@Test` methods, no export script for HI→EN, `INTERNET` permission present
  despite the offline claim.

This is the starting line every v1–v4 number below is measured against.

---

## v1 — foundation (Phases 1–5, 2026-07-20)

**Goal: make the codebase safe to build on before making it fast.** No runtime benchmark exists yet
in this stage; the work is structural.

- **Phase 1 — clean reconstruction.** Native resources moved to a single process-scoped owner
  (`BhashaBridgeApp`): one engine per direction, created lazily, released on `onTrimMemory`.
  Activities borrow, never own. The v3.4.1 leak becomes structurally impossible rather than a
  discipline to remember. (An audit later found `TRIM_MEMORY_COMPLETE` stopped being delivered on
  API 34+, so even the *fix* needed a second correction — the same defect class, caught the same way:
  check that the trigger fires, not that the code exists.)
- **Phase 2 — benchmark infrastructure.** `Metrics` emits one structured JSON line per timed
  operation, `inline`-guarded by `BuildConfig.DEBUG` so it compiles out of release builds. Every
  performance claim in every later phase traces back to this one type.
- **Phase 4 — decoder abstraction.** A `Decoder` interface plus a `LogitsSource` seam
  (`nextLogits(prefix)`) separates decode strategy from model execution. `MtEngine` depends only on
  the interface. This single seam is what lets the entire uncached→cached rewrite in v2 land without
  touching the decode loop.
- **Phase 3 — KV-cache feasibility (no code).** Read the IndicTrans2 source directly: the model
  *does* implement the mBART caching contract end to end; the ONNX export wrapper was what dropped
  `past_key_values`/`use_cache`. This finding is what makes v2 possible — the model was never the
  problem.
- **Phase 5 — minimum translation runtime.** Tokenizer, ORT sessions, `MtEngine` wired together for
  the first time in V4, mirroring v3.4.1 semantics as a control.

**v1's contribution:** zero speedups, one architectural precondition (the `LogitsSource` seam) and
one leak that can no longer happen by construction.

---

## v2 — the cache rewrite (Phases 6A–8, 2026-07-21)

**Goal: fix the O(n²) decode, since v1's Phase 3 proved it was an export defect, not a model
limitation.**

- **6A — KV-cache export.** Optimum has no config for IndicTrans' custom architecture, so the cached
  decoder was hand-exported as three graphs — `encoder` / `decoder_init` / `decoder_step` — with the
  72-tensor cache (18 layers × 4) flattened to named ONNX I/O and verified numerically (7/7 gate,
  max_abs_diff 9.06e-06).
- **6B — cached runtime.** The three graphs wired in behind the existing `Decoder`/`LogitsSource`
  interface from v1 — the decode loop did not change.
- **6C — INT8 quantization** of the cached graphs, matching v3.4.1's own quantization approach for a
  fair comparison (verified against v3's shipped file sizes: `encoder_int8.onnx` 74.9 MB both ways).
- **6D — the benchmark that mattered most.** v3's own INT8 graphs re-run on V4's harness — same
  tokenizer, same sentences, 30 runs — the only true apples-to-apples measurement of the v3 lineage
  that exists anywhere:

  | Sentence | Tokens | v3 lineage | V4 (cached) | Speedup |
  |---|---|---|---|---|
  | "Water." | 2 | 184.5 ms | 166.4 ms | 1.11× |
  | "Hello, how are you?" | 6 | 526.4 ms | 350.0 ms | 1.50× |
  | 12-token sentence | 12 | 1353.6 ms | **640.1 ms** | **2.11×** |

  Throughput at 12 tokens: v3 **falls** to 9.5 tok/s as length grows; V4 **rises** to 21.6 tok/s. That
  divergence, not the 2.11×, is the real argument — v3's decode cost grows with output length, V4's
  doesn't.
- **7 — ORT session tuning.** `intra_op=2`, CPU arena off, sequential execution — reached by testing
  and rejecting the naive "use all four performance cores" rule (719.0 ms / σ 88.8 vs. 667.2 ms /
  σ 18.4).
- **8 — capability-aware Arm policy.** `CpuCapabilities.detect()` reads `/proc/cpuinfo` HWCAP and
  cpufreq topology; `ExecutionPolicy` derives thread count and arena setting from it. No device list
  anywhere in the code. (This classifier broke twice on real silicon and was corrected twice — see
  v3/v4 below.)

Same day, the UI layer was rebuilt from scratch on top of the new engine: branding, translation
screen, speech interaction, emergency phrases, drawer/history/settings, onboarding, audio import —
all restored against the v1/v2 architecture rather than ported from v3.4.1's 961-line `MainActivity`.

**v2's contribution:** the 2.11× decode number and the architectural proof that KV-caching plus a
derived thread policy beats a hard-coded one on the hardware it was supposedly tuned for.

---

## v3 — startup, footprint, and the ecosystem (2026-07-21 → 2026-08-06)

**Goal: v2 made translation fast; v3 makes the *app* fast to open, small enough to ship, and correct
across languages and devices — not just on the one phone in the room.**

- **Startup, instrumented first.** Of the original ~27 s cold start, measurement (not guessing) found
  49% was a JSON tokenizer parser reading one character at a time across 3.4M characters, and 46% was
  ORT session build. Unpacking assets — the part everyone assumes is expensive — was 1.8 s, once.
  Four sequential fixes:

  | Step | Engine ready | Change |
  |---|---|---|
  | Baseline | 27,000 ms | — |
  | Buffered + block-wise dictionary parse | 16,584 ms | −32.8% |
  | Parallel ONNX session load | 10,502 ms | −36.7% |
  | Optimized-graph cache + `.ort` mmap | ~5,134 ms | −51% |
  | Packed binary vocabulary cache | **2,736 ms** | −43% |

  One experiment reverted here: running the tokenizer parse concurrently with session load made cold
  start *worse* (5,134 → 5,475 ms) — three ORT sessions at `intra=2` already saturate four big cores,
  so a fourth CPU-bound thread only adds contention.
- **Footprint.** Splitting the decoder into `decoder_init` + `decoder_step` had silently doubled every
  weight tensor (torch.onnx.export writes a full copy into each graph) — 283 MB was being blamed on
  the KV-cache for months before hashing the tensor bytes showed `decoder_step` holds no unique data
  at all. One content-addressed shared blob per direction took assets 893.97 → 617.23 MB.
- **ONNX Runtime 1.17.1 → 1.27.0**, gated on the full benchmark suite so the upgrade itself is a
  measured decision, not a version bump.
- **HI→EN pipeline (Phase 12).** v3.4.1 had no HI→EN export script and no traceable checkpoint at
  all. This phase closes that provenance gap and makes the app genuinely bidirectional.
- **The cross-device campaign begins (entries #1–#9).** The Arm capability classifier from v2's
  Phase 8 broke on real silicon twice: it filed a Snapdragon 8 Gen 1's three A710 mid cores as
  "little" and ran single-threaded on the most capable CPU in the database; corrected, it then broke
  on an 8-core Oryon part where every core reports the same `CPU part` and DVFS splits 6@3629 +
  2@4742 MHz — misreading six full-size cores as "efficiency" by frequency alone. The fix gates the
  split on core IP, not frequency ratio, because a genuine A55/A78 split (0.91 ratio) can be *higher*
  than the misleading Oryon split (0.77).
- **Correctness bugs surfaced by real sentences.** Every benchmark sentence in both v3.4.1 and early
  V4 was 2, 6, or 12 tokens — convenient lengths that hid a truncation bug: `maxSteps=18` against a
  cap that should have scaled with source length, silently cutting off 5/16 long sentences (31%).
  Fixed once a 20–25-token test set was actually tried. Resource-lifecycle bugs (use-after-free on the
  Vosk model, jobs that kept running after being stopped, a ten-second lock parking the main thread)
  were found and fixed in the same window by testing the app under real, sustained use rather than one
  clean run.

**v3's contribution:** 9.9× faster cold start, a smaller and finally-bidirectional APK, validation
across 9 devices instead of 1, and the first evidence that the Arm policy from v2 needed real silicon
to find its own bugs.

---

## v4 — hardening and the honest ledger (2026-08-06 → 2026-08-13)

**Goal: everything that only shows up under sustained load, real corpora, or a device nobody had
tested yet — and being willing to publish the negative results, not just the wins.**

- **Quality held as a gate, then actually measured.** No optimization that changed output was ever
  accepted through v1–v3. This stage adds the real check: INT8 vs fp32 on WMT14 newstest (500
  sentences, paired bootstrap, 1000 resamples) — BLEU and chrF2++ move in *opposite* directions by
  comparable, marginal amounts (EN→HI: 21.88→21.85, p=0.336; HI→EN: 32.31→32.79, p=0.039). A
  quantization perturbation, not degradation — and exact token parity on real sentences is only
  50.6%/44.4%, so the synthetic "identical greedy tokens" gate had been overstating risk the whole
  time.
- **Energy, measured against what the hardware allows.** 0.9 J per translation on the baseline
  device, built on a harness designed to refuse a lie about power rather than report one.
- **SME: found, then correctly sized.** `simpleperf` plus disassembly of the hottest loop proved the
  runtime dispatches into KleidiAI's SME int8 kernel (`smopa za0.s, ...`) on the S26 Ultra. It was
  then priced by A/B: **+4.9–9.1%**, not the 2× the raw device-to-device gap suggested — most of that
  2× is microarchitecture, thread count, and thermal headroom, not the ISA. `ExecutionPolicy` was
  then updated to disable KleidiAI specifically on SME parts, where it turned out to be 12.7%
  *slower* — a reversal caught only because the ledger kept re-testing a "closed" finding.
  No further SME claim can be made — the only device that has it is no longer obtainable, and the
  report says so rather than extrapolating.
- **Footprint, finished.** Weight blobs compressed in the APK (617.3 → 520.3 MiB), and shipping
  optimized-ONNX over the shared blob took process memory 783 → 460 MB PSS — closing gaps that the
  v3 dedup pass couldn't reach on its own.
- **Sustained-load validation.** 500 direction switches: 0 failures. 1024 consecutive translations
  unplugged: +29% latency drift, attributed to DVFS stepping down (2942→2092 MHz) rather than
  labelled as a regression, because memory, GC, threading, and lifecycle were each excluded by their
  own separate measurement. Release build measured faster than debug on every metric, because
  `Metrics` and debug logging compile out — not because of R8, which is still off.
- **The re-validation discipline.** The S26 Ultra's numbers were re-measured against the artifact
  that actually ships (not the one that shipped when it was first tested) rather than trusting a
  months-old result — "nothing changed" is itself a finding, published as one.
- **`docs/FINAL_REPORT.md`.** The narrative pass this document draws from — 29 prior documents
  reduced to the one a reader should see first.

**v4's contribution:** the numbers stop being single-device optimism and become a corpus-scored,
energy-priced, load-tested, cross-silicon-honest ledger — including every REVERT and every retracted
claim, because deleting a row that "turned out not to work" is how a ledger starts lying.

---

## The arc, v1 → v4, in one table

| | v0 (v3.4.1) | v1–v2 (Phases 1–8) | v3 (startup/footprint) | v4 (hardened) |
|---|---|---|---|---|
| 12-token decode | 1353.6 ms | **640.1 ms** (2.11×) | 640.1 ms | 640.1 ms, corpus-verified quality-neutral |
| Cold engine ready | 27,000 ms* | 27,000 ms | **2,736 ms** (9.9×) | 2,736 ms, release build ~5% faster still |
| Process memory (PSS) | 981 MB | 783 MB (arena off) | 617 MB assets | **460 MB PSS** |
| Devices validated | 1 | 1 | 9, four vendors | 9, re-validated against shipping artifact |
| Long-sentence truncation | 5/16 (31%) | 5/16 | 5/16 | **0/16**, fixed with test coverage |
| `@Test` methods | 2 | growing | growing | **94** |
| Energy | unmeasured | unmeasured | unmeasured | **0.9 J/translation, measured** |

\*v1–v2 did not touch startup; that work starts in v3.

---

## What the shape of the story actually says

Four stages, four different failure modes fixed in order: v1 removed a structural hazard (unowned
native memory) that no amount of later optimization could have worked around. v2 fixed an
algorithmic-complexity bug hiding behind a working demo. v3 found that the *visible* slow part
(cold start) was not the *actual* slow part (a naive JSON parser), and that the whole thing had never
been checked on hardware besides the one phone in the room. v4 found that most of the interesting bugs
only exist under conditions nobody had tried yet — sustained load, real corpora, a device that hadn't
shipped when the classifier was written — and that a ledger which never publishes a negative result
cannot be trusted the moment it publishes a positive one.

Full source material: `docs/OPTIMIZATION_SUMMARY.md` (phase-by-phase ledger with evidence grades),
`docs/FINAL_REPORT.md` (single narrative pass), `docs/LESSONS_FROM_V3.md` (defect → rule mapping),
`docs/V3_VS_V4_COMPARISON.md` (side-by-side numbers), `bench/results/cross-device/` (raw per-device
data).
