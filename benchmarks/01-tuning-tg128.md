# 01 - Tune: thread-count sweep

Model `Qwen3.5-0.8B-Q4_K_M.gguf` · host `Windows-AMD64` · llama.cpp `b10488`
CPU: **4 physical · 8 logical** cores · `ngl=0` · metric `tg128`

| threads (-t) | tg128 (tok/s) | vs best |
|:--|--:|--:|
| 1 | 38.1 | 57% |
| 2 | 51.7 | 77% |
| 4 | 66.1 | 98% |
| 8 | 67.1 | 100% |
| 16 | 48.3 | 72% |

**Best**: `-t 8` at 67.1 tok/s
**Slowest tested**: `-t 1` at 38.1 tok/s (1.76x spread)
**Against the physical-core default** (`-t 4`, 66.1 tok/s): 1.02x

Use this in your run:

```bash
LAB_N_THREADS=8 make bench
```

## Your explanation

The knee is at `-t 8`, which equals the logical-core count on this 4C/8T machine. Throughput climbs steeply from 1 to 4 threads, then plateaus between 4 and 8 threads (66.1 ? 67.1 tok/s, only +1.6%), and drops sharply to 48.3 tok/s at 16 threads. This shape is consistent with decode being memory-bandwidth-bound: once all physical cores are busy, the extra hyperthreads can only hide some latency, so the curve flattens rather than continuing to rise. The sharp drop at 16 threads is classic oversubscription ? 16 decode workers are now competing for the same memory channels and scheduler time slices, so throughput collapses. The fact that 8 threads is barely better than 4 suggests this CPU?s memory subsystem is already saturated at the physical-core count, and the small remaining gain at 8 threads is likely just enough latency hiding to matter for this tiny 0.8B model. For this lab, `-t 8` is the practical choice, but the real lesson is that anything above the logical-core count is pure overhead.
