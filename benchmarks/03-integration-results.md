# 03 - Integrate: RAG pipeline run

Host `Windows-AMD64` · llama.cpp `b10488` ·
retrieval backend: **keyword overlap** · 3 queries

| Query | Contexts retrieved | embed (ms) | retrieve (ms) | llm (ms) | total (ms) |
|:--|--:|--:|--:|--:|--:|
| Why is goodput more useful than raw throughp... | goodput, paged, radix | 0.0 | 0.0 | 5828.4 | 5828.5 |
| What problem does PagedAttention actually so... | paged, radix, disagg | 0.0 | 0.0 | 3318.1 | 3318.2 |
| When does splitting prefill and decode help?... | disagg, radix, batching | 0.0 | 0.0 | 3639.9 | 3639.9 |

Mean per stage (ms): embed **0.0** · retrieve **0.0** ·
llm **4262.1** · total **4262.2**
Dominant stage: **llm** (100% of total)

## Answers returned

**Why is goodput more useful than raw throughput?**

> Based on the context provided, **Goodput** is more useful than raw throughput because it specifically addresses the issue of **slo (Service Level Objectives) compliance**.

Here is the breakdown based on the text:

1.  **Goodput's limitation**: It counts requests per second that met the TTFT (Total Time to Failure) and TPOT (Total Per-Operation Time) targets.
2.  **The problem with raw throughput*

**What problem does PagedAttention actually solve?**

> PagedAttention solves the problem of **GPU memory fragmentation** caused by storing key-value pairs in non-contiguous pages.

By storing KV cache in non-contiguous pages, the model avoids the internal fragmentation that would otherwise waste most of the GPU's memory capacity.

**When does splitting prefill and decode help?**

> Based on the provided context, splitting prefill and decode helps when **prefill is compute-bound and decode is memory-bandwidth-bound**.

The context explicitly states: "Disaggregated serving splits prefill and decode onto separate pools because prefill is compute-bound and decode is memory-bandwidth-bound." This indicates that splitting is necessary to utilize the specific performance characteri


## Which N16-N19 pieces are real

- **N16** (cluster / data-pipeline): stubbed. No real ingestion pipeline; the toy docs are in-memory.
- **N17** (lakehouse / warehouse): stubbed. TOY_DOCS is a Python list, not a real lakehouse.
- **N18** (vector index / embedding server): stubbed. Retrieval is keyword overlap on the toy corpus because no embedding server was started. If I had to classify, the embed stage is the one that would become real first with `make serve-embed` + a real corpus.
- **N19** (RAG orchestrator / retrieval logic): stubbed in the sense that `retrieve()` is a local function over a 6-document toy set, not a real index query.

The dominant stage is **llm** (100% of total, mean 4262 ms). That is exactly what I expected for a 0.8B model on CPU with no embedding server and a toy corpus: retrieval is effectively free because there are only 6 short documents to scan, and embed is 0.0 ms because there is no embedding call. If I had to halve end-to-end latency, I would attack the LLM stage first: switch to a faster thread count (the 1.02x speedup from `-t 4` to `-t 8` is modest but free), or reduce `max_tokens` in the generation call. A smaller context budget (`--ctx-size 1024`) would also shave prefill time, but on this tiny model the gain is smaller than on a larger one.
