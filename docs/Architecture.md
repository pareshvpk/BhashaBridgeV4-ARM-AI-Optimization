# Architecture

How BhashaBridge V4 is structured, how translation is decoded, and how the cached INT8 decoder runs on device.


---

# BhashaBridge V4 — Architecture Rules

Binding on every phase. A change that violates a rule here is rejected in review, even if it works.

Companion documents — each rule lives in exactly one place, no restatement:
- Folder layout and per-folder scope → `PROJECT_STRUCTURE.md`
- Mechanical limits (file size, naming, nullability, formatting) → `CODING_STANDARDS.md`
- Allowed dependency edges → `DEPENDENCY_RULES.md`
- Evidence behind these rules → `LESSONS_FROM_V3.md`

---

## 0. The governing test

Before adding **any** folder, file, class, interface, or dependency, answer in one sentence:

> What breaks today if this does not exist?

"Future flexibility", "it's cleaner", "standard practice", and "we might need it" are not answers.
If there is no answer, the thing does not get created.

V3.4.1 shipped 3640 lines of Kotlin, of which ~600 were dead or inert: a 302-line `SpeechManager.kt`
with zero references, a 133-line beam search with zero callers, two `SpannableString` blocks built
and discarded on every call, and a `MainActivity_backup.kt` sitting outside the source set. None of
it was written maliciously. Each piece passed a "might be useful" test instead of the question above.

---

## 1. Folder organization

**R1.1** — Exactly four subsystem folders: `ui/`, `mt/`, `speech/`, `bench/`. Adding a fifth
requires a written justification appended to `PROJECT_STRUCTURE.md` and explicit approval.

**R1.2** — A folder must name a **subsystem** (a real, nameable part of the running app), never a
**layer of indirection**. Permanently banned folder names, regardless of contents:

```
controllers/  services/  helpers/  utils/  managers/  repositories/
interfaces/   base/      common/   core/   misc/      extensions/
```

These names describe how code is shaped, not what it does. A folder named for a shape has no
admission criteria, so everything qualifies — which is how V3.4.1's `FileUtils.kt` ended up as the
place where an asset-copy loop lived because it belonged nowhere else.

**R1.3** — A folder holding one file is not a folder. Merge it upward or accept that the subsystem
does not exist yet. (This rule deleted the `tts/` folder from the Phase 1 sketch: one `Speaker.kt`
did not justify it, and TTS is audio output, so it belongs with audio input in `speech/`.)

**R1.4** — No nesting inside a subsystem folder. `mt/` is flat. If `mt/` needs sub-folders, `mt/`
has become two subsystems and must be split at the top level, or — far more likely — it has grown
speculative code that R0 should have stopped.

---

## 2. Package depth

**R2.1** — Maximum depth is `com.bhashabridge.app.<subsystem>`. Two segments below the app package
is a violation.

**R2.2** — The root package `com.bhashabridge.app` holds **at most three files**: the `Application`
class, the shared `Direction` type, and (if ever justified) one more. It is the app's shared
vocabulary, not its attic. The three-file cap is the enforcement mechanism — without a hard number,
"shared" degrades into "unclassified".

**R2.3** — Anything that wants to live at the root and is not shared by at least two subsystems
belongs in a subsystem instead.

---

## 3. Class responsibilities

**R3.1** — One class, one responsibility, stated in one sentence in its KDoc without the word
"and". If the sentence needs "and", the class needs splitting — or the second half needs deleting.

**R3.2** — A class that both *decides* and *renders* is two classes. V3.4.1's `MainActivity` chose
which translator to use, applied the partial-result discard policy, timed the call, mutated six
views, spoke the result, and wrote history — inside a single 70-line `runTranslation()`. No single
edit to that function was wrong; the accumulation was.

**R3.3** — Orchestration is itself a responsibility, and it belongs to exactly one class per screen
(the `ViewModel`). An Activity that orchestrates is a violation. So is a `ViewModel` that touches a
`View`.

**R3.4** — See `CODING_STANDARDS.md` §1 for the size limits that make violations of R3.1 visible,
and §8 for when splitting is the *wrong* fix.

---

## 4. Ownership

The rule that V3.4.1 most needed and most lacked.

**R4.1** — Every object with a non-trivial lifetime has exactly **one owner**: the single object
responsible for creating it, holding the only long-lived reference, and destroying it. Written as
`Creator == Holder == Destroyer`. If those are three different objects, there is no owner.

**R4.2** — Ownership is declared in the owning class's KDoc under an `Owns:` heading, listing what
it creates and what releases it. Not optional, not implied.

**R4.3** — A borrower never releases. If class B receives a reference from class A, B may use it and
must not close, shut down, or null it. Enforced by visibility: release methods are `internal` to the
owner's file, never `public`.

**R4.4** — **Native resources are owned by the process, never by an Activity.** `OrtSession`,
`OrtEnvironment`, and Vosk `Model` are owned by `BhashaBridgeApp` (the `Application`). Activities and
ViewModels borrow.

V3.4.1's failure, exactly: `OnnxSessionManager.release()` existed at line 113, was correct, and had
**zero call sites**. `MainActivity.onDestroy()` released audio, Vosk, TTS and three executors, then
dropped both `Translator` references without releasing them. Every rotation leaked ~639 MB of native
heap the JVM cannot reclaim, and paid the 8.6-second load again. The method was not missing. The
owner was.

**R4.5** — Any `release()`/`close()`/`shutdown()` method must have a call site in the same commit
that introduces it. A cleanup method with no caller is dead code that reads as safety.

**R4.6** — A call site is not enough; the platform must still be calling it. `onTrimMemory` was gated
on `TRIM_MEMORY_COMPLETE`, which Android stopped delivering to apps targeting API 34+ — so from the
`targetSdk` bump onward the release path was unreachable and V4 held ~600 MB for the life of every
process on a modern device, with a call site that looked correct in review. Any trigger owned by the
platform (a trim level, a lifecycle callback, a broadcast) needs a test or a log line that proves it
*fires*, not just code that would run if it did. This is R4.5's blind spot, found by audit.

---

## 5. Lifecycle

**R5.1** — Three lifetimes exist. Every stateful object is assigned to exactly one, in its KDoc:

| Lifetime | Scope | Holds | Survives rotation |
|---|---|---|---|
| **Process** | `BhashaBridgeApp` | ONNX sessions, Vosk models, TTS engine | yes |
| **Screen** | `ViewModel` | UI state, in-flight request state, history | yes |
| **View** | `Activity` | view references, listeners, animations | no |

**R5.2** — Nothing expensive is assigned to the View lifetime. "Expensive" = costs more than ~50 ms
to recreate, or holds native memory.

**R5.3** — Rotation must reload nothing and lose nothing. This is a testable exit criterion, not an
aspiration: rotate during an active translation, confirm the result still arrives.

**R5.4** — Process-lifetime native resources are released on `onTrimMemory(TRIM_MEMORY_BACKGROUND)`.
Returning ~639 MB when the OS asks is not optional at this footprint.

`BACKGROUND`, not `COMPLETE`: of the seven trim levels only `BACKGROUND` and `UI_HIDDEN` are still
delivered on API 34+, and `BACKGROUND` is the one that means what this rule intends — the process is
on the background LRU list and a kill candidate. `UI_HIDDEN` fires on every home-press and would
charge a ~27 s reload to a user who is coming straight back. See R4.6.

**R5.4a** — A release may only run when nothing is using the resource. `BhashaBridgeApp` keeps a
borrower count; a trim arriving during a translation or a recording session sets a "release when
idle" flag rather than closing a session out from under a live caller. Ownership says *who* frees,
this says *when* it is safe to — without it, the correct fix to R5.4 is a native use-after-free.

**R5.5** — Every release trigger and every idle timeout is a **named constant with a comment giving
its unit and the reason for its value**. These are device-dependent tuning knobs, not truths. A
60-second idle release that is right on a 6 GB phone may be wrong on a 12 GB one, and the next
engineer must be able to find and change it without reading the whole file.

---

## 6. Threading

**R6.1** — Coroutines, not raw `Executor` + `Handler`. V3.4.1 ran three `SingleThreadExecutor`s plus
a `Handler`, and manually reimplemented "wait for two parallel jobs" with two `AtomicBoolean`s and a
shared closure.

**R6.2** — One dispatcher per subsystem, declared in one place:

| Work | Dispatcher | Why |
|---|---|---|
| MT inference | single-threaded, dedicated | the decoder reuses preallocated buffers |
| ASR capture | single-threaded, dedicated | continuous mic read loop |
| File/asset I/O | `Dispatchers.IO` | blocking, parallel-safe |
| UI state | `Dispatchers.Main` | Android requirement |

**R6.3** — **MT inference is single-thread-confined, and this is load-bearing.** `OrtSession.run()`
is itself thread-safe, but the V4 decoder deliberately reuses preallocated logit, token, and KV
buffers to eliminate per-step allocation. Two concurrent translations on one engine would corrupt
each other's buffers. The confinement is what makes the optimization safe — document it at the class,
never "clean it up".

**R6.4** — Shared mutable state crossing threads is `@Volatile`, atomic, or confined. No exceptions.
V3.4.1's `hiEnLoading` was a plain `Boolean` written from the translation thread and read from the
main thread — a real race that happened to be survivable.

**R6.5** — Cancellation is structured. A screen leaving means its coroutines are cancelled, not
orphaned to post results into a dead view. V3.4.1 needed two separate `isFinalPending` checks in
`runTranslation()` — one before the work, one inside the posted block — precisely because it had no
cancellation.

---

## 7. Dependencies

**R7.1** — Adding a third-party dependency requires justification against `CODING_STANDARDS.md` §0
(the ladder): stdlib, then platform, then an already-present dependency, then — last — a new one.

**R7.2** — Library versions are pinned in `gradle/libs.versions.toml` only. No inline version
strings in a build file.

**R7.3** — The ML stack (ONNX Runtime 1.27.0, Vosk 0.3.47) moves only in an **isolated commit gated
on a before/after benchmark**, never as a side effect of other work. Silently changing a version
invalidates every number in `ENGINEERING_PLAN.md` §1.4. ORT was frozen at 1.17.1 through Phase 12
and bumped to 1.27.0 under this rule; Vosk stays at v3.4.1's 0.3.47.

**R7.4** — No dependency injection framework. The graph is one `Application`, four subsystems, and
constructor parameters. A DI container here is configuration wearing the costume of architecture.

**R7.5** — Module-to-module dependency direction is governed by `DEPENDENCY_RULES.md` and is
non-negotiable.

---

## 8. Logging

**R8.1** — One tag constant per subsystem: `"BB.MT"`, `"BB.Speech"`, `"BB.UI"`, `"BB.Bench"`. The
`BB.` prefix makes `adb logcat -s BB.*` show the whole app and nothing else.

**R8.2** — **User speech and translation text must never be logged in release builds.** V3.4.1 logged
`"[$direction] $text → $result"` at `Log.d` on every translation. That is the user's spoken words,
in an app whose emergency-phrase feature implies medical and safety contexts. Guard all content
logging behind `BuildConfig.DEBUG`.

**R8.3** — Levels have fixed meanings: `e` = broken, user affected. `w` = degraded, recovered. `d` =
debug-only, `BuildConfig.DEBUG`-gated. `i` and `v` are unused — two unused levels is two fewer
judgement calls.

**R8.4** — Never log inside the decode loop. It runs up to 18 times per translation, and string
formatting there is exactly the allocation R9 exists to prevent. Measurements go through `bench/`.

**R8.5** — A swallowed exception must log at `w` with the reason it is safe to swallow. Silent
`catch` blocks are banned.

---

## 9. Documentation

**R9.1** — Every class carries KDoc with these headings, in this order, no others:

```
Purpose:    one sentence, no "and"
Owns:       what it creates and destroys, or "nothing"
Lifetime:   Process | Screen | View
Thread:     which dispatcher, and whether it is confinement-sensitive
```

Four lines. V3.4.1's headers had grown to six sections including prose "Performance notes" that
narrated the code below them; those notes were accurate and still went stale, because a comment
describing *how* code works competes with the code and loses.

**R9.2** — Comments explain **why**, never **what**. `// increment counter` is noise. `// ponytail:
single lock, per-direction locks if throughput matters` is a decision with a stated ceiling.

**R9.3** — A deliberate simplification is marked `// ponytail:` naming the shortcut and its upgrade
path. These are harvestable into a debt ledger; an unmarked shortcut is indistinguishable from a bug.

**R9.4** — Non-obvious constants carry unit and rationale: `// ms — Vosk emits partials every ~100ms;
250 keeps the translator from flooding`.

**R9.5** — No commented-out code. Ever. Git remembers.

---

## 10. Naming

**R10.1** — Principle: a name states what the thing **is**, not what pattern it participates in.
Banned suffixes on new classes: `Manager`, `Helper`, `Util`, `Service`, `Handler`, `Processor`,
`Controller`, `Impl`, `Base`, `Abstract`.

`OnnxSessionManager` does not manage sessions — it creates two and copies two files. Naming it
`Manager` made "and it also copies assets" feel natural, because a Manager manages whatever it is
handed. Precise names resist scope creep; vague names invite it.

**R10.2** — Mechanical conventions (casing, boolean prefixes, constant style) → `CODING_STANDARDS.md` §4.

---

## 11. Performance

**R11.1** — Optimize only what is measured. A performance change without a before/after number is
not accepted, regardless of how obviously correct it seems.

**R11.2** — The decode loop is the one hot path in this app and is held to a stricter standard than
all other code: **zero allocation per decode step.** No `copyOf()`, no boxing, no `MutableList<Long>`,
no per-step `Set`, no string formatting. All buffers preallocated at engine construction.

**R11.3** — Read ONNX outputs through `getFloatBuffer()`, never `OnnxTensor.value`. `value`
materializes the entire `[1, seq, vocab]` output as boxed nested Java arrays — roughly 4.6 MB per
step at seq=18, vocab≈64k — in order to read 64k floats. This single line
(`Translator.kt:204`) dominates the measured ~50 MB transient allocation per translation.

**R11.4** — Outside the decode loop, clarity wins over micro-optimization. Allocating in a click
listener is fine. R11.2 is a scalpel, not a lifestyle — applying it everywhere would produce
unreadable code in exchange for unmeasurable gains.

**R11.5** — Every tunable that touches hardware — thread counts, affinity masks, buffer sizes,
timeouts — stays exposed as a named constant or a `RuntimeConfig` field. Real silicon does not match
the datasheet, and a value hardcoded into an expression cannot be swept by a benchmark.

---

## 12. Error handling

**R12.1** — **Failure is never encoded in a success-typed value.** V3.4.1's `translate()` returned
the literal string `"Translation failed"`, which `MainActivity` rendered in the output field in the
same colour as a real translation, and which the history feature would happily store as a
translation. A `String` return type promised a translation; the function sometimes returned prose
about not having one.

V4 returns a sealed result type. The caller cannot render a failure as a success without writing
code that visibly says so.

**R12.2** — Catch only what you can handle. Catching `Exception` around a 40-line block converts
every bug inside it — including `NullPointerException` and `OutOfMemoryError`-adjacent failures —
into the same user-facing message, and hides the stack trace that would have identified it.

**R12.3** — Every user-facing error string is a resource, resolved at the UI layer. Subsystems return
typed failures, never display text. A subsystem that knows about `values-hi/` has the wrong
responsibility.

**R12.4** — Input validation happens at trust boundaries — file import, ASR output, user text — and
is never removed in the name of simplicity. This rule is exempt from R0.

---

## 13. Benchmarking

**R13.1** — Instrumentation lands **before** the optimization it measures, never after. V3.4.1's
audit could describe its problems but could not size them, because measurement arrived last. This
inverted the whole engineering effort: work was prioritized by intuition for months.

**R13.2** — A performance claim is only valid stated as: **device name, n, median, p95, σ, before →
after**, reproducible by one command. Numbers without a device and an n are not evidence.

**R13.3** — Instrumentation must not perturb what it measures. `bench/` is `BuildConfig.DEBUG`-gated
and writes off the hot path. One release-build run with instrumentation disabled is recorded as the
control.

**R13.4** — The baseline is frozen and never retroactively edited. `bench/results/20260714_183707`
is the number V4 is judged against.

**R13.5** — A behaviour change that improves speed must pass the parity gate
(`ENGINEERING_PLAN.md` §3.2) first. A faster app that translates differently has not been optimized;
it has been broken, and the benchmark will report the breakage as a win.

---

## 14. Git workflow

**R14.1** — One fix, one commit. Never batch unrelated fixes.

**R14.2** — Re-verify after each commit — build, and run the relevant check — before starting the
next fix. Batched changes make bisecting a regression impossible.

**R14.3** — Conventional Commits: `feat:`, `fix:`, `perf:`, `refactor:`, `docs:`, `test:`, `build:`.
`perf:` commits must include the before/after numbers in the body, per R13.2.

**R14.4** — A phase ends on a commit where the app builds, installs, runs, and its benchmark slice is
recorded. No phase ends red.

**R14.5** — Model binaries (`*.onnx`, `model.*`, `dict.*.json`) are never committed. 636 MB in git
history is unrecoverable without rewriting it. `.gitignore` covers this from the first commit.

**R14.6** — v3.4.1 is never modified. It is the control in the experiment.

---

## 15. Amending these rules

A rule may be changed. It may not be quietly ignored.

To change one: edit this file, state what broke under the old rule, and commit the amendment
separately with `docs:`. If a rule turns out to be wrong, the amendment is the correct outcome — a
rules document nobody can change gets routed around instead, which is how V3.4.1 acquired six
architecture reports that described the code without governing it.


---

# BhashaBridge V4 — Decoding Architecture

Phase 4. The decoding subsystem: one abstraction, two interchangeable strategies (greedy, beam
search), designed so the final choice is made by measurement, not assumption.

Status: **abstraction + both strategies implemented and unit-tested. Device benchmark and
recommendation are BLOCKED on the `mt/` runtime — see §6. No recommendation is made here, on
purpose (§7).**

---

## 1. The abstraction

Three small types in `mt/`, no more:

```
Decoder        interface: decode(logits: LogitsSource, sourceLen: Int): LongArray
LogitsSource   fun interface: nextLogits(prefix: LongArray): FloatArray
DecodeConfig   data class: start/eos token, maxSteps, minTargetLen, repetitionPenalty, noRepeatNgram
```

`GreedyDecoder` and `BeamSearchDecoder` both implement `Decoder`. That is the whole surface.

No dependency injection, no service locator, no factory hierarchy — the brief forbade them and they
would earn nothing here. Selecting a strategy is `val decoder = GreedyDecoder()` versus
`BeamSearchDecoder(beamWidth = 2)`. MtEngine will hold one `Decoder` and never mention which.

Shared per-step rules (repetition penalty, no-repeat-ngram blocking, argmax) are internal top-level
functions in `Decoder.kt`, not a base class: they are pure transforms on a `FloatArray` with no
state to inherit, and a one-method superclass is exactly the ceremony `ARCHITECTURE_RULES.md` R0
exists to stop.

---

## 2. LogitsSource is the seam that pays for itself in Phase 5

A decoder's only view of the model is `nextLogits(prefix) -> FloatArray`. It does not know ONNX
Runtime, the tokenizer, or the encoder exist. This is not abstraction for its own sake — it is the
exact boundary the KV-cache work needs:

- **Today (uncached graph):** MtEngine's `LogitsSource` runs the decoder session on the *whole*
  prefix and returns the last logits row. O(n²) over the sequence, as documented in
  `model_pipeline/MODEL_PIPELINE.md` §4.
- **After the cached export (Phase 5):** the same `LogitsSource` feeds only the last token plus
  cached key/values. O(n).

The signature does not change, so **neither `GreedyDecoder` nor `BeamSearchDecoder` changes.** The
decoders are written once, against the interface, and survive the single biggest optimisation in the
plan untouched. That is the return on keeping decoding logic and model I/O in separate types.

---

## 3. The two strategies (semantics mirror v3.4.1)

Both reproduce `Translator`'s behaviour so the eventual parity gate has a like-for-like baseline:
start/eos token `2`, length cap `max(14, sourceLen)`, repetition penalty `1.1`, no-repeat-3-gram
blocking, and — for beam — length-normalised scoring at width `2`.

**One deliberate divergence from v3.4.1: `maxSteps` is `128`, not `18`.** v3.4.1 paired a length cap
of `max(14, sourceLen)` with a decode loop bounded at 18 steps, so the cap was fiction above 18
tokens and longer inputs were silently truncated mid-sentence. `maxSteps` is now an absolute runaway
ceiling rather than the working limit, and `DecodeConfig.targetCap` clamps into it so a single number
bounds generation. Outputs at 2/6/12 tokens — every benchmark sentence and every parity golden in the
project — are unaffected; only sources over 18 tokens change, and they change from truncated to
complete. The ceiling stays finite because `Tokenizer.encode` applies no source-length limit; see the
KDoc on `DecodeConfig.maxSteps` for the latency arithmetic.

- **Greedy:** one `nextLogits` call per token; argmax after the penalty/blocking rules; stop on eos
  or the cap. This is what v3.4.1 shipped for both directions.
- **Beam:** keeps the best `beamWidth` partial sequences, expands each by its top-`beamWidth`
  tokens, ranks all expansions by length-normalised log-probability, retires finished beams, keeps
  the best active ones. Returns the best completed beam. v3.4.1 implemented this identically but
  **never called it** — it was dead code; here it is a real, selectable, tested strategy.

**Cost, stated but not yet measured:** on the uncached graph each active beam is a full decoder
forward pass every step, so width-K beam does ~K× greedy's decoder work per step, and beams run at
least as long. The latency cost is real; whether the quality gain justifies it on this device is §7's
open question.

---

## 4. Correctness — what is proven now

`DecoderTest` (7 tests, JVM, no Android, no model — `app/src/test/.../mt/DecoderTest.kt`), all green:

| Test | Proves |
|---|---|
| greedy takes locally-best token, stops at eos | argmax selection + eos stop |
| length cap stops without emitting eos | the `max(minTargetLen, sourceLen)` halt |
| beam looks ahead and beats greedy | beam keeps a lower-immediate/higher-total path greedy discards |
| beam width 1 is greedy | width-1 beam ≡ greedy (invariant) |
| repetition penalty damps each distinct prior token once | penalty sign handling + dedup |
| no-repeat-ngram blocks the recreating token | n-gram suffix scan |
| argmax returns lowest index on ties | tie-break matches v3.4.1's strict `>` |

**What these do NOT prove:** parity with v3.4.1's actual translations. That needs the real decoder
graph and the tokenizer, and is an on-device check in Phase 5/6. These tests verify the *algorithms*
against a synthetic `LogitsSource`; they are decisive for the decode logic and silent on the model.

---

## 5. Where benchmark instrumentation goes

Not inside the decoders — they stay pure and JVM-testable, and `android.util.Log` (which `Metrics`
uses) is not available under plain JUnit. Instrumentation lives at the MtEngine boundary, which owns
the run and can read the token count off the returned `LongArray`:

```
Metrics.begin("translate")
Metrics.stage("tokenize"); … ; Metrics.stage("encode"); …
Metrics.stage("decode");   val ids = decoder.decode(source, srcLen)
Metrics.counter("tokens", (ids.size - 1).toLong())
Metrics.counter("decode_steps", stepsRun.toLong())
Metrics.end()
```

`decode_steps` is what separates greedy from beam in the data: beam's step count and per-step cost
both rise. This is a call-site plan, not code — MtEngine does not exist yet (§6).

---

## 6. The blocker: the device benchmark needs the `mt/` runtime

Implementation-order steps 3 (V3 parity), 6 (Metrics integration), and 7 (device benchmark), plus
every latency/memory/throughput deliverable, require running real translation on the SM-M315F. That
requires machinery Phase 4 does not build and Phase 3 explicitly forbade:

- **Tokenizer** — SentencePiece over `model.SRC`/`model.TGT` (and HI→EN variants).
- **ONNX sessions** — encoder + decoder `OrtSession` construction in `mt/` (the `LogitsSource` impl).
- **MtEngine** — ties tokenizer + sessions + a `Decoder`, owns `Metrics` instrumentation.
- **~278 MB of int8 assets** wired into the app.

That is Phases 5–6 of `ENGINEERING_PLAN.md`, not "one simple decoder abstraction." **No device
numbers are reported, and none are invented.** Fabricated benchmarks would violate the phase's own
RULE ("do not use a single run as evidence") more deeply than reporting none.

---

## 7. Recommendation: WITHHELD, pending evidence

The phase rule is explicit: the recommendation must be evidence-based, beam search must not be
assumed superior, and greedy is a valid outcome. **There is no device evidence yet, so there is no
recommendation.** Presenting one now would be the exact failure the rule guards against.

What is decided is the *procedure*, so the recommendation is mechanical once §6's runtime exists:

**Benchmark method** — on the SM-M315F, both strategies, identical inputs (a fixed sentence set
spanning short/medium/long), warm-up run discarded, **n ≥ 30** timed iterations each. Report per
metric: **n, median, p95, standard deviation** — never a single run.

**Metrics:** total translation latency · decoder latency · tokens/sec · peak memory (`meminfo`) ·
output token count · CPU utilisation if practical.

**Decision rule (set before the data, to keep it honest):**
- **A — keep beam** only if it improves translation quality measurably *and* median total latency
  stays within the app's interaction budget (target < 900 ms, `ENGINEERING_PLAN.md`).
- **B — revert to greedy** if beam's latency blows the budget without a quality win the user would
  notice — the likely outcome given §3's K× cost on an uncached O(n²) graph, but that is a
  hypothesis to test, not a conclusion.
- **C — keep both, runtime-selectable** only if quality and latency genuinely trade off per input
  length and there is a real use case for choosing per translation. Default to **not** C: two live
  strategies is complexity that must earn its place.

The abstraction supports all three at zero extra cost, which is the point — the architecture does not
prejudge the measurement.


---

# KV-Cache Runtime (Phase 6B)

How the Android runtime drives the verified cached ONNX graphs from Phase 6A. This
phase changed **only** the runtime — `OnnxModels`, `MtEngine`, and the assets. The
`Decoder` abstraction (`Decoder.kt`, `GreedyDecoder.kt`, `BeamSearchDecoder.kt`,
`DecodeConfig`), the tokenizer, and the benchmark subsystem are untouched.

## The seam stayed put

The whole cache lives behind `LogitsSource`, the fun-interface introduced in Phase 4:

```
fun interface LogitsSource { fun nextLogits(prefix: LongArray): FloatArray }
```

A decoder still asks one thing — "logits for this prefix" — and knows nothing about
sessions, cache tensors, or init-vs-step. Phase 4 predicted this exact swap: "when
the KV-cache export lands the source changes from re-feed-the-whole-prefix to
feed-one-token-plus-cached-state and NO decoder code changes." That held. The only
new code is `CachedLogitsSource` inside `MtEngine.kt`.

## Session ownership

`OnnxModels` owns three `OrtSession`s per direction (was one encoder + one decoder):

| session | asset (EN→HI) | role |
|---|---|---|
| encoder | `encoder.onnx` | source → `encoder_hidden_states` |
| decoder_init | `decoder_init.onnx` | first step: prefix → logits + fresh cache |
| decoder_step | `decoder_step.onnx` | later steps: 1 token + cache → logits + grown cache |

Lifetime unchanged from Phase 5: `BhashaBridgeApp` constructs one `MtEngine` per
direction at process scope and is the sole caller of `release()`, which now closes
all three sessions. The `OrtEnvironment` remains a process singleton, not closed here.
This is still the structural answer to v3.4.1's L2 leak — one owner, one release, a
real call site.

## Cache tensor mapping

The cache is the MBart 4-tensors-per-layer layout from Phase 6A (self-attn K/V that
grow each step, cross-attn K/V constant for the translation). ONNX carries it as a
flat, ordered list of named tensors:

- `decoder_init` / `decoder_step` **output**: `logits`, then `present.{i}.{decoder|encoder}.{key|value}`.
- `decoder_step` **input**: `decoder_input_ids`, `encoder_attention_mask`, then
  `past_key_values.{i}.{decoder|encoder}.{key|value}` — **no `encoder_hidden_states`**
  (pruned; the graph reuses cached cross-attn K/V).

The mapping is read from the model, never hard-coded: `OnnxModels.pastInputNames` is
the step graph's input names minus the two non-cache inputs, in graph order. Because
the export declared past inputs and present outputs in the same layer-major order,
`pastInputNames[i]` is fed from the previous run's output `i + 1` (output `0` is
`logits`). A `require` asserts `pastInputNames.size == initOutputs − 1`, so a graph
whose ordering ever drifts fails loudly at construction.

## decoder_init → decoder_step flow

`CachedLogitsSource` is created once per `translate()` and owns the cache for that one
translation. The decoder hands it the full prefix each step; it maps that to the graphs:

```
first call (prefix = [start])                  -> decoder_init(prefix)            -> logits + present
prefix == prevPrefix + one token               -> decoder_step(newToken, present) -> logits + present'
prefix does not extend prevPrefix (e.g. beam)  -> decoder_init(prefix)            -> rebuild cache
```

Greedy always extends by exactly one token, so it runs init once then step for every
later token — the intended fast path. A decoder that reorders or shortens prefixes
falls back to init: correct, just not accelerated. That is deliberate; this phase
integrates the cache, it does not tune decode strategy.

Native lifetime: each run's `OrtSession.Result` holds the `present` tensors. They are
fed straight into the next `decoder_step` as `past_key_values`, so the previous Result
is kept open until the next run supersedes it, then closed. `CachedLogitsSource.close()`
(called in `translate`'s `finally`) releases the last one; `encoder_hidden_states` and
the attention mask are closed there too. Cache lifetime is exactly one translation.

## Metrics

`translate` emits one structured line via the frozen `Metrics` API. Wall-clock stages:
`tokenize`, `encoder`, `decode` (total decode). Counters: `tokens`, plus the cache-flow
split `init_us` (single decoder_init) and `step_us` / `steps` (summed decoder_step).
`total_ms` is the whole translation. No benchmarking — timing collection only.

## Validation results

On the SM-M315F (RZ8N93BC18A, Exynos 9611, Android 12), 2026-07-21,
`connectedDebugAndroidTest` **3/3 green** with the fp32 cached graphs (~1.8 GB).

| check | result |
|---|---|
| translation succeeds | ✓ EN→HI produced Devanagari |
| **output identical to uncached runtime** | ✓ **exact**, both sentences (below) |
| cache grows correctly | ✓ `steps == tokens`, one init + one step per token (Metrics) |
| no native leaks | ✓ `release()` in `translate`'s `finally`; cache closed per translation |
| repeated translations succeed | ✓ "Water." ×2 deterministic; two engines, many translates |
| release() works | ✓ closes all three sessions, no throw |

Parity, cached fp32 vs the committed uncached int8 runtime (golden captured just
before the swap, same device, same greedy decoder):

```
'Hello, how are you?'  ->  'हैलो , आप कैसे हैं ?'     (both runtimes, identical)
'Water.'               ->  'पानी ।'                    (both runtimes, deterministic)
```

Exact match across a precision change (fp32 vs int8) — stronger than required, and
consistent with Phase 6A's graph-level proof (fp32 cached == fp32 uncached, greedy
tokens identical, max_abs_diff 9.06e-06).

Cache-flow Metrics (debug build, **not** a benchmark):

```
"Hello, how are you?"  total 2297.8 ms  encoder 1227.4  decode 1021.9   init_us 142801  step_us 782684  steps 6  tokens 6
"Water." (warm)        total  359.9 ms  encoder   59.0  decode  299.3   init_us 112925  step_us 178349  steps 2  tokens 2
```

`steps == tokens` confirms one decoder_init then one decoder_step per generated token.

**Timing note (observation, not a benchmark — Phase 6C owns that):** the fp32 cached
runtime is *slower* than the uncached int8 it replaced (Hello 2298 ms vs 774 ms). That
is precision, not caching: fp32 is 4× the weight data, and quantise/tuning are out of
scope here. The cache's O(1)-per-step win is real but masked by fp32 weight cost until
int8 quantisation lands. Phase 6C measures cached vs uncached greedy at equal precision.

## Known limitations

- **fp32, not int8.** The verified graphs are fp32 (~1.8 GB). Quantise + re-verify is a
  later phase; this phase must ship the *verified* graphs, and quantising forbidden here.
- **EN→HI only.** HI→EN cached graphs are not yet exported (Phase 6A did en_hi; R-PROV).
  `OnnxModels` names the `hi_en_*` cached assets so the export drops straight in.
- **Beam is unaccelerated.** Non-extending prefixes rebuild the cache via init. Greedy
  (the shipped strategy) is unaffected.
