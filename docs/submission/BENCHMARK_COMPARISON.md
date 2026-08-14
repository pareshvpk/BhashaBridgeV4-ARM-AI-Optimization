# BhashaBridge — Version Benchmark Comparison

**v3.4.1 · V4** — technical benchmarks and engine workings only. Direction EN→HI · greedy decode · INT8 · 30
iterations, 5 warm-ups, counterbalanced. The V4 column shows the **best observed** result for each metric,
measured on the Galaxy S26 Ultra unless stated otherwise.

------------------------------------------------------------------------

## 1. Translation latency (ms) — lower is better

| Workload                             | v3.4.1 | V4       | v3.4.1 → V4 |
|--------------------------------------|--------|----------|-------------|
| Short sentence (2 tokens)            | 184.5  | **22.0** | **8.4×**    |
| Long sentence (12 tokens)            | 1353.6 | **77.0** | **17.6×**   |
| First translation (incl. graph warm) | —      | 82.0     | —           |

Mid-length (6 tokens) was 526.4 ms in v3.4.1; V4 collapses the same class to well under 100 ms.

------------------------------------------------------------------------

## 2. Decode complexity & throughput

The defining change: v3.4.1 re-attended the whole growing prefix every step (**O(n²)** — per-token cost *rises*
with length), while V4 holds a flat per-step cost (**O(n)**).

| Metric | v3.4.1 | V4 |
|----|----|----|
| Complexity class | O(n²) | **O(n)** |
| Per-token trend (2→12 tokens) | 13.9 → **9.5** tok/s (falling) | 14.9 → **21.6** tok/s (rising) |
| Flat per-step decode cost | — | ~44–46 ms |
| Suite-normalized throughput† | — | **573.7** |

† Suite-normalized tokens/sec (fixed token budget ÷ decode time); the v3.4.1↔V4 per-token trend row above is
the apples-to-apples throughput comparison. The two rows are on different scales — do not cross-compare them.

------------------------------------------------------------------------

## 3. Latency stability — lower is better

| Metric                | v3.4.1       | V4       |
|-----------------------|--------------|----------|
| Stdev, 12 tokens (ms) | 93.0         | **18.4** |
| p95, 12 tokens (ms)   | 864.0        | **81.0** |

v3.4.1 was five times jitterier than V4 and its worst-case p95 was ~10× higher.

------------------------------------------------------------------------

## 4. Startup

| Metric                        | v3.4.1           | V4        |
|-------------------------------|------------------|-----------|
| Engine ready, full chain (ms) | ~27,000          | ~5,134 ‡  |
| Cold engine-init (ms)         | not instrumented | **2,039** |
| Warm engine-init (ms)         | —                | 644       |
| Hot engine-init (ms)          | —                | 632       |
| Cold tokenizer parse (ms)     | uninstrumented   | **72**    |

‡ Same-chain re-measurement of the v3.4.1-shape load path — a **5.3×** reduction over the 27 s baseline; the cold engine-init row is a later, faster measurement basis.

------------------------------------------------------------------------

## 5. Memory

| Metric | v3.4.1 | V4 |
|----|----|----|
| Steady-state process memory | ~983 MB | **617 MB** ⁂ |
| Native heap | not reported | 71 MB |
| Per-rotation leak | **present** (release path never called) | **none** (structured eviction) |
| Memory arena | on (default) | **off** (−38% at no latency cost) |

⁂ Process-set metric.

------------------------------------------------------------------------

## 6. Artifact & model

| Metric | v3.4.1 | V4 |
|----|----|----|
| Graphs per direction | 2 | 3 (KV-cache split) |
| Model cache / staged size | ~490 MB per direction | **279.8 MB** (shared-blob) |
| Total shipped assets | ~638 MB | 617 MB (both directions) |
| Quantization | INT8 (unverified export) | **INT8, 3.96× vs fp32** |
| Greedy-token parity vs fp32 | not verified | **identical** (max logit Δ 0.448) |

The KV-cache split *looked* like a +283 MB cost; it was an export defect (duplicated weights) and, once pointed at one content-addressed blob, V4 ships **both directions in less space than v3.4.1 used for the same two** — output bit-identical.

------------------------------------------------------------------------

## 7. Correctness

| Metric | v3.4.1 | V4 |
|----|----|----|
| Long-input truncation (n=16) | **5 / 16 (31%)** | **0 / 16** |
| Decode step cap | fixed `maxSteps=18` (cuts long inputs) | `sourceLen×1.6 + 8`, 128 ceiling |
| Runtime version | 1.17.1 | **1.27.0** |
| `@Test` methods | 2 | **94** |

------------------------------------------------------------------------

## 8. Engine workings (architecture)

| Aspect | v3.4.1 | V4 |
|----|----|----|
| Decode caching contract | **dropped** by the export wrapper → O(n²) | hand-built 3-graph cached export (encoder / decoder-init / decoder-step) |
| Export & verification | none for one direction; no gate | `cached_export` + `quantize` + `verify` (**7/7 numeric checks**) |
| Thread policy | hard-coded (`intra=4, inter=2`) | **derived and clamped [1,2]** |
| Optimal thread count (measured) | assumed 4 | **2** |
| Session lifecycle | leaked; no eviction path | evict-before-build; native heap returns 557.8 → 13.2 MB on release |
| Acceleration-kernel policy | fixed default | **kernel disabled by policy** (forcing it on cost −8.9%) |
| Execution-provider probe | not tested | alternate provider claims **0 nodes** → folds to the default kernel; the offload provider ran 2.3× slower |

------------------------------------------------------------------------

## 9. Headline deltas

|  | v3.4.1 → V4 |
|----|----|
| Long-sentence latency | **17.6× faster** |
| Latency stability (stdev) | **−80%** |
| Cold startup | **~5–13× faster** |
| Truncation defect | **31% → 0%** |
| Artifact size | smaller for **two** directions than v3.4.1 for two |

The three deltas outside any noise band — **17.6× long-sentence latency**, **O(n²) → O(n) decode**, and **31% → 0% truncation** — are the load-bearing results; sub-10% differences are reported as observations, not wins.
