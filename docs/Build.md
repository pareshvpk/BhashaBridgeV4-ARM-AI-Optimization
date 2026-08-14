# Build

Project structure, dependency rules, and how the app is built.


---

# BhashaBridge V4 — Project Structure

The complete hierarchy. Four subsystem folders, one root package with a hard three-file cap.
Every folder below is justified against `ARCHITECTURE_RULES.md` §0: *what breaks today if this does
not exist?*

---

## The whole thing

```
BhashaBridgeV4/
├── docs/                          ← these documents
├── bench/                         ← host-side benchmark harness (shell + Python, not app code)
├── model_pipeline/                ← ONNX export + quantisation (Python, not app code)
└── app/src/main/
    ├── assets/                    ← model binaries, never committed
    ├── res/                       ← layouts, strings (en + hi), drawables
    └── java/com/bhashabridge/app/
        ├── BhashaBridgeApp.kt     ← Application. The process-scoped owner.
        ├── Direction.kt           ← EN_TO_HI | HI_TO_EN. The one shared type.
        ├── ui/
        ├── mt/
        ├── speech/
        └── bench/
```

Four folders. One entry point. Compare V3.4.1: eighteen Kotlin files in a single flat package, with
no folder telling you which of them mattered.

---

## Root package — `com.bhashabridge.app`

**Why it exists.** Two things are genuinely shared by every subsystem: the app's entry point, and the
translation direction. `Direction` is used by `mt` (which model), `speech` (which Vosk model, which
TTS voice), and `ui` (which labels). Putting it inside `mt/` would force `speech` to depend on `mt`
for a two-value enum — a dependency edge bought for nothing.

**What belongs inside.**
- `BhashaBridgeApp.kt` — the `Application`. Sole creator and destroyer of all native resources
  (`ARCHITECTURE_RULES.md` R4.4). Exposes lookup, not logic.
- `Direction.kt` — the enum, plus whatever trivially derives from it.

**Hard cap: three files.** This is the load-bearing part of the rule. "Shared" is not a filter —
everything can be argued into it. A number is a filter. When a fourth file wants in, someone has to
justify it out loud, which is exactly the conversation V3.4.1 never had before `FileUtils.kt`
appeared as the home for code that belonged nowhere.

**What must NEVER go inside.**
- Extension functions, string helpers, date formatters, `Constants.kt` — the attic starter kit.
- Anything used by only one subsystem. One user means it belongs to that user.
- Business logic of any kind. `BhashaBridgeApp` returns an engine; it never translates.

> **Note on a Phase 1 change.** `ENGINEERING_PLAN.md` §2.3 proposed a separate `MtEngineHolder`
> object. R10.1 rejects it: a holder that holds one thing is a synonym for the `Application` that
> already exists, and `Holder` is a shape name. The `Application` *is* the process-scoped owner.
> One fewer class, one fewer indirection, identical behaviour.

---

## `ui/` — everything the user sees or touches

**Why it exists.** Android's entry points (`Activity`) and the state that outlives configuration
changes (`ViewModel`) are one concern: presenting the app. They change together, for the same
reasons — a new screen affects both, a new model does not affect either.

**What belongs inside.**
- `MainActivity.kt` — view binding, listener wiring, rendering state. Nothing else.
- `TranslateViewModel.kt` — the **only** orchestrator. Decides what to translate and when, holds
  UI state, survives rotation. Owns no native resource (R4.4).
- `OnboardingActivity.kt`, `SetupActivity.kt` — first-run flow.
- `WaveformView.kt` — custom view, mic amplitude.
- `EmergencySheet.kt` — the emergency-phrase UI.
- `EmergencyPhrases.kt` — the static phrase table. It is presentation content with no behaviour, and
  it has exactly one consumer, one folder away.

**What must NEVER go inside.**
- `OrtSession`, `OrtEnvironment`, Vosk `Model`, `TextToSpeech` — or any reference to one. This is
  R4.4, and it is the single rule that would have prevented V3.4.1's 639 MB rotation leak.
- Tokenization, decoding, audio buffers, ONNX anything.
- User-visible strings as Kotlin literals. They go in `res/values/` and `res/values-hi/`.
  V3.4.1 shipped a `values-hi/strings.xml` and then bypassed it with inline `if (isHindi) "…" else "…"`
  pairs scattered through 961 lines, so the resource system existed and governed nothing.
- Orchestration inside `MainActivity`. The Activity renders; the ViewModel decides (R3.3).

---

## `mt/` — machine translation

**Why it exists.** The subsystem this app is judged on. It is the only place ONNX Runtime is
referenced, and the only place with a hot loop.

**What belongs inside.**
- `MtEngine.kt` — the subsystem's single public entry. `translate(text): TranslationResult`.
- `OnnxModels.kt` — session creation and the `release()` that `BhashaBridgeApp` calls. Nothing else
  may call it (R4.3).
- `GreedyDecoder.kt` — the decode loop. Held to zero-allocation-per-step (R11.2).
- `Tokenizer.kt` — text ↔ token ids.
- `RuntimeConfig.kt` — execution provider, thread count, affinity, KV on/off. One data class, swept
  by `bench/`.

**What must NEVER go inside.**
- Any `android.view`, `android.widget`, or `Context`-for-UI reference. `mt/` takes a `Context` only
  to reach assets, and nothing else.
- User-facing strings, including error messages. `mt/` returns typed failures (R12.3).
- A second decode strategy that nothing calls. Beam search is not being ported: 133 lines,
  zero callers, never validated. It returns only if Phase 8 quality data demands it, and then only
  behind `RuntimeConfig`, never on the user path.
- Sub-folders (R1.4). Five flat files is the whole subsystem.

---

## `speech/` — audio in and audio out

**Why it exists.** ASR and TTS are one subsystem: the app's audio boundary. They share the direction
concept, the same permission surface, and the same "is the model/voice actually installed" problem.

This folder is why `tts/` does not exist. The Phase 1 sketch had `tts/Speaker.kt` alone in its own
folder — R1.3 killed it, correctly.

**What belongs inside.**
- `SpeechInput.kt` — mic capture and Vosk recognition. In V3.4.1 this was split across
  `AudioCaptureController` and `VoskModelLoader`, which were separate classes that could not function
  without each other and were constructed with references to each other's executors.
- `AudioFileInput.kt` — imported file → 16 kHz mono PCM.
- `Speaker.kt` — TTS output.
- `TextCleanup.kt` — the ASR correction phrase tables.

**What must NEVER go inside.**
- Any reference to `mt/`. Speech produces text and consumes text; it does not know translation exists.
  `ui/` connects them. This keeps the dependency graph a tree (`DEPENDENCY_RULES.md`).
- View manipulation. `speech/` emits events; `ui/` decides what they look like.
- Ownership of the Vosk `Model`. Created and destroyed by `BhashaBridgeApp` (R4.4). `SpeechInput`
  borrows it and must not close it (R4.3).
- A `SpeechManager.kt`. V3.4.1's was 302 lines with zero references anywhere in the codebase. The
  name is banned by R10.1 and the file is not being ported.

---

## `bench/` — measurement

**Why it exists.** The Arm submission is judged on evidence. R13.1 requires instrumentation to land
before the optimization it measures, which means it needs a real home from the start, not a
`// TODO: measure this` in the decode loop.

**What belongs inside.**
- `Metrics.kt` — stage timers, TTFT, tokens/sec, allocation deltas. Structured JSONL to logcat.
- `BenchActivity.kt` — debug-only sweep runner, `adb`-launchable, exercises the `RuntimeConfig`
  matrix.

**What must NEVER go inside.**
- Anything reachable from a release build. `BuildConfig.DEBUG`-gated, and `BenchActivity` is
  `debug`-source-set only.
- Anything the app's behaviour depends on. Delete this folder and the app must still work
  identically — that is the test of whether measurement stayed separate from function.
- Timing code that lives in the hot path. R11.2 and R13.3 both forbid it.

---

## Non-app directories

### `docs/`
These five documents plus `ENGINEERING_PLAN.md`. Design decisions and rules only.

**Never:** per-phase status updates, changelogs, or reports narrating what the code does. V3.4.1
accumulated eleven root-level markdown reports — `Architecture_Reverse_Engineering_Report.md`,
`INDEPENDENT_AUDIT.md`, `SRP_ANALYSIS_MIGRATION_PLAN.md`, and more — describing code that kept
changing underneath them. Documentation that describes rather than governs goes stale silently and
then actively misleads. Git history is the changelog.

### `bench/` (repo root, host-side)
The existing shell/Python harness that produced the frozen baseline. It works. It is extended, not
rewritten.

### `model_pipeline/` (repo root)
ONNX export, KV-cache split, quantisation, offline graph optimisation. Python. This is where
Problem 1 is actually fixed — the decoder cannot cache because
`export_decoder_dynamic.py` exports a `forward()` with no `past_key_values` parameter. No amount of
Kotlin changes that.

**Never:** a checked-in Python virtualenv. V3.4.1 committed `translation_build/indic_env/` with the
full contents of numpy and onnx, headers and all.

### `app/src/main/assets/`
Model binaries at runtime. Covered by `.gitignore` from the first commit (R14.5).

---

## Growth rules

**Adding a file to an existing folder** — no approval needed if it obeys the folder's scope above.

**Adding a fifth subsystem folder** — requires appending a justification section to this document
and explicit approval. The bar: name the subsystem, name what breaks without it, and show it needs
two or more files (R1.3).

**Adding a file to the root package** — requires displacing one of the three, or a written argument
for raising the cap. Deliberately the hardest change in the project to make.

---

## Expected size

| Area | V3.4.1 | V4 target |
|---|---:|---:|
| `ui/` | — | ~750 |
| `mt/` | — | ~600 |
| `speech/` | — | ~520 |
| `bench/` | — | ~240 |
| root | — | ~60 |
| **Total Kotlin** | **3640** | **~2170** |

~40% smaller, with rotation-safety, explicit ownership, and instrumentation that V3.4.1 did not have.
The reduction is not cleverness — roughly 600 of the removed lines are code V3.4.1 was already not
executing.

These are targets, not quotas. Coming in under them by deleting something unjustified is a good
outcome. Coming in under them by cramming three responsibilities into one file is not
(`CODING_STANDARDS.md` §1).


---

# BhashaBridge V4 — Dependency Rules

Which parts of the app may reference which. Non-negotiable, and the cheapest architectural property
to keep if enforced from commit one — and among the most expensive to recover once lost.

---

## The graph

```
                    ┌──────────┐
                    │   ui/    │         Activities, ViewModel, views
                    └────┬─────┘
                ┌────────┴────────┐
                ▼                 ▼
          ┌──────────┐      ┌──────────┐
          │ speech/  │      │   mt/    │   ASR + TTS  |  translation
          └────┬─────┘      └────┬─────┘
                └────────┬────────┘
                         ▼
                  ┌─────────────┐
                  │    root     │        BhashaBridgeApp, Direction
                  └─────────────┘

    bench/  ──────► reads any subsystem. Nothing may depend on bench/.
```

Arrows point in the only permitted direction. There is no arrow back up, and no arrow between
`speech/` and `mt/`.

---

## Permission matrix

| From ↓ / May reference → | ui | speech | mt | root | bench |
|---|:--:|:--:|:--:|:--:|:--:|
| **ui** | — | ✅ | ✅ | ✅ | ⚠️² |
| **speech** | ❌ | — | ❌ | ✅ | ⚠️² |
| **mt** | ❌ | ❌ | — | ✅ | ⚠️² |
| **root** | ❌ | ✅¹ | ✅¹ | — | ❌ |
| **bench** | ✅ | ✅ | ✅ | ✅ | — |

² `bench/Metrics` only, never any other type in `bench/`. See D4 as amended.

¹ `BhashaBridgeApp` constructs and releases the native resources of `speech/` and `mt/` — that is its
one job as process-scoped owner (`ARCHITECTURE_RULES.md` R4.4). It calls constructors and `release()`.
It never calls `translate()` or `startListening()`. Ownership is not orchestration.

---

## The four rules

### D1 — Dependencies point downward. Always.

A lower layer never references a higher one. `mt/` must not import from `ui/`; `speech/` must not
import from `ui/`. Enforced by review and by import inspection.

**Why.** An upward reference is how a subsystem acquires a second responsibility without anyone
deciding it should. The moment `mt/` can see a `TextView`, formatting output becomes a one-line
change that lives in the wrong place forever.

### D2 — `speech/` and `mt/` never reference each other.

This is the load-bearing rule, and it is a deliberate departure from the layered sketch
`UI → Speech → Translation` in the Phase 2B brief.

Speech produces text. Translation consumes text. They never touch. `ui/` connects them:

```
SpeechInput ──emits──► ui/TranslateViewModel ──calls──► MtEngine
```

**Why not chain them.** Chaining reads naturally — speech feeds translation — but it makes `speech/`
depend on translation for a data flow it does not own. Then typed-text translation (no speech
involved) has to route around `speech/`, or `speech/` grows a passthrough path for text that never
came from a microphone. V3.4.1 had exactly one orchestrator for exactly this reason; the mistake
there was that the orchestrator was a 961-line Activity, not that orchestration existed.

Keeping them siblings means:
- Either can be tested with no knowledge of the other.
- Typed input and spoken input reach `mt/` by the same path, so they cannot diverge.
- The graph is a tree. Trees cannot cycle.

### D3 — No circular dependencies. Structurally, not by discipline.

The matrix above has no cycles by construction. Any proposed dependency that would create one is
rejected — and the correct fix is almost always that the shared thing belongs at the root, or that
the two classes are one class.

**The `Direction` case, worked.** `mt/` needs it (which model), `speech/` needs it (which Vosk model,
which TTS voice), `ui/` needs it (which labels). Putting it in `mt/` forces `speech/ → mt/` for a
two-value enum — an entire dependency edge bought to share a type with no behaviour. It lives at the
root instead, which both may depend on. This is what the root package is *for*, and it is the reason
the three-file cap exists to keep it from becoming anything else.

### D4 — `bench/` observes. Nothing observes `bench/` except through `Metrics`.

> **Amended in Phase 2** under R15. The original wording is preserved at the end of this section.

`bench/` may read any subsystem. A subsystem may import **`bench/Metrics` only**, and only through
its inline, `BuildConfig.DEBUG`-guarded API. No subsystem may reference anything else in `bench/`.

**The test:** in a release build, no instruction from `bench/` executes.

This holds by construction rather than by discipline. `Metrics`' entry points are `inline` and
guarded by a compile-time-constant `false`, so in release the call, its arguments, and any string
built inside are eliminated at the call site and the object is never class-loaded. Measurement
cannot perturb what it measures if it is not there (`ARCHITECTURE_RULES.md` R13.3).

**What changed and why.** The original test was *"delete `bench/` and the app still compiles and
behaves identically"*, and the original rule required each subsystem to declare its own metrics
interface for `bench/` to implement. Phase 2 showed the test was wrong, not merely inconvenient:

- The **intent** was always "measurement must not perturb function". A direct call to an
  eliminated-in-release API satisfies that intent completely. Compilation was a proxy for the real
  property, and a bad one.
- The **original mechanism cost more than it bought**: one interface per subsystem, each with
  exactly one implementation — banned by R7.4, and the kind of indirection R0 exists to stop.
  It would have produced four interfaces to avoid one import.

The behavioural guarantee is unchanged and is now stated directly instead of through a proxy.

> **Original wording (superseded):** *"`bench/` may read any subsystem. No subsystem may import from
> `bench/`. The test: delete `bench/` and the app still compiles and behaves identically. Metrics
> are emitted through a small interface owned by the subsystem, no-op in release — never by a
> subsystem calling into `bench/`."*

---

## External dependencies

| Library | Only permitted in |
|---|---|
| ONNX Runtime (`ai.onnxruntime.*`) | `mt/` |
| Vosk (`org.vosk.*`) | `speech/` |
| `android.speech.tts.*` | `speech/` |
| `android.view.*`, `android.widget.*` | `ui/` |
| Coroutines | anywhere |
| AndroidX lifecycle | `ui/`, root |

**An `import ai.onnxruntime.*` outside `mt/` is a build-breaking review failure.** It means ONNX
Runtime has leaked out of the one subsystem that is allowed to know it exists — and the Phase 7
runtime sweep, which changes execution providers and thread configuration, assumes exactly one place
in the codebase constructs a session.

Same reasoning for Vosk in `speech/`.

`Context` is permitted in `mt/` and `speech/` for asset access **only** — never for UI, resources, or
`startActivity`.

---

## What this buys

**Phase 7 (Arm runtime sweep)** touches `mt/RuntimeConfig.kt` and `mt/OnnxModels.kt`. Two files. If
session construction were spread across the app the way V3.4.1 spread direction handling, sweeping an
execution provider would be a cross-cutting edit with cross-cutting risk.

**Phase 5 (KV cache)** rewrites `mt/GreedyDecoder.kt`. `ui/` and `speech/` cannot observe the change
because they cannot see the decoder — which is what makes the parity gate meaningful: if output
changes, the decoder changed it, and nothing else could have.

**Rotation-safety** is structural, not vigilant. `ui/` cannot hold an `OrtSession` because `ui/`
cannot import `ai.onnxruntime`. V3.4.1's 639 MB rotation leak was possible because `MainActivity`
could reach all the way down to a native session; here the compiler refuses.

---

## Enforcement

**Today: review.** Every PR checks imports against the matrix. Grep is sufficient and honest:

```bash
grep -rn "ai.onnxruntime" app/src/main/java --include=*.kt | grep -v "/mt/"      # must be empty
grep -rn "org.vosk"       app/src/main/java --include=*.kt | grep -v "/speech/"  # must be empty
grep -rn "android.widget" app/src/main/java --include=*.kt | grep -v "/ui/"      # must be empty
```

**Not today: a module-per-subsystem Gradle build.** Real Gradle modules would make violations
impossible to compile rather than merely rejected in review — genuinely stronger. It is not being
built now because three greps enforce the same rules at a fraction of the build-time cost, and
splitting a ~2200-line app into four Gradle modules for compiler-enforced boundaries is exactly the
enterprise-imitating complexity `ARCHITECTURE_RULES.md` R0 exists to stop.

**The trigger to revisit:** if a boundary violation ever reaches `master`, the greps have failed as
enforcement and modules become justified. Recorded here so that decision is made on evidence rather
than taste.
