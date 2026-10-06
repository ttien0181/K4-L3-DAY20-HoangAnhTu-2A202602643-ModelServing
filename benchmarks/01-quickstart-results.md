# 01 - Measure: latency baseline

Model `Qwen3.5 0.8B` · host `Windows-AMD64` · llama.cpp `b10488`
Settings: `threads=4` `ngl=0` `ctx=2048`
`max_tokens=64` · warm-up discarded
Completed requests: `Q4_K_M` 10/10 · `UD-Q2_K_XL` 10/10

| Quantization | Size (GB) | Load (ms) | TTFT P50/P95 (ms) | TPOT P50/P95 (ms) | E2E P50/P95/P99 (ms) | Decode (tok/s) |
|:--|--:|--:|--:|--:|--:|--:|
| Q4_K_M | 0.50 | 2157 | 136 / 148 | 16.1 / 18.9 | 1152 / 1325 / 1325 | 62.0 |
| UD-Q2_K_XL | 0.39 | 1539 | 181 / 218 | 15.6 / 16.1 | 1153 / 1208 / 1208 | 64.1 |

- **TTFT** = prefill. Short prompts keep it small; long-context RAG is where it explodes.
- **TPOT** = per-output-token decode cost, bounded by memory bandwidth. `decode tok/s = 1000 / TPOT_p50`.
- `UD-Q2_K_XL` decodes **1.03x faster** than `Q4_K_M` here, for 0.11 GB less on disk.

## Your observation

On this CPU-only Windows laptop, `UD-Q2_K_XL` is indeed smaller (0.39 GB vs 0.50 GB) and decodes slightly faster (64.1 vs 62.0 tok/s, +1.03x), exactly the memory-bandwidth regime the lab describes. However, its TTFT is meaningfully worse (P50 181 ms vs 136 ms, +33%; P95 218 ms vs 148 ms, +47%), because the heavier dequantization cost hits prefill on a compute-starved CPU. For a latency-sensitive chat endpoint where users wait for the first token, the 2-bit model is not a clear win here; the 4-bit model gives lower TTFT variance and a better first-token experience for only 0.11 GB more.
