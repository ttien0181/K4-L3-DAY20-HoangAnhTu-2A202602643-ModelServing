# 02 - Serve: load test + saturation reading

Host `Windows-AMD64` · llama.cpp `b10488` ·
`--parallel 4` · `ctx=2048` · `threads=4` ·
`ngl=0`

| Users | Reqs | RPS | P50 (ms) | P95 (ms) | P99 (ms) | Eff. concurrency | Failures |
|:--|--:|--:|--:|--:|--:|--:|--:|
| 10 | 74 | 1.31 | 5900 | 10000 | 11000 | 8.2 | 0.0% |
| 50 | 77 | 1.31 | 31000 | 39000 | 41000 | 35.2 | 0.0% |

*Effective concurrency = RPS x average latency (Little's Law) -- how many requests were
really in flight, regardless of how many users locust simulated. It counts queued requests
too, so the occupancy/slot ratio can legitimately exceed 1.0; it is occupancy, not
utilisation. For true slot utilisation use the server's own gauges (`make metrics`).*

## What these two runs say

| Going from 10 to 50 users | |
|:--|--:|
| Offered load | 5x |
| Throughput actually delivered | **1.00x** (20% of linear) |
| P95 latency | **3.90x** |
| Effective concurrency at 50 users | 35.2 vs `--parallel 4` slots (occupancy/slot ratio 8.79) |

**Saturated.** Throughput delivered only 1.00x for 5x the offered load, and effective concurrency (35.2) is at or above all 4 decode slots. Saturation sets in somewhere at or below 50 users; the load you added beyond that point became queue time rather than throughput.

Throughput moved 1.00x while P95 moved 3.90x. That gap is the goodput argument: past saturation you buy throughput by spending latency, and if your SLO is a P95 target then the requests you added are no longer being served within it. (This lab does not fix an SLO number for you -- pick one in your write-up and state how much goodput you keep at it.)

## Your reading

The server is already saturated at 50 users. The number that convinced me is the effective concurrency: 35.2 in-flight requests against only 4 parallel slots (occupancy/slot ratio 8.79). That means on average 8?9 requests were queued or in-flight for every decode slot the engine has. The proof is in the throughput: going from 10 to 50 users (5x offered load) delivered exactly 1.00x throughput (1.31 RPS in both runs), while P95 latency ballooned from 10.0 s to 39.0 s (3.90x). The 3.9x latency increase with zero throughput increase is the textbook signature of queue-time dominance.

To raise goodput at a fixed SLO, the first knob I would change is `--parallel` (increase the number of slots) or reduce the per-request token budget (`max_tokens`) in the load test mix. More slots directly raises the ceiling of the scheduler, while shorter outputs reduce the time each slot is occupied. Context size (`--ctx-size`) is a secondary lever: shorter contexts reduce prefill cost and KV memory pressure, but on this CPU-only machine the primary bottleneck is slot count, not KV capacity.
