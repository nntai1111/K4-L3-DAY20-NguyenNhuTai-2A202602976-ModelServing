# 02 - Serve: load test + saturation reading

Host `Windows-AMD64` · llama.cpp `b10488` ·
`--parallel 4` · `ctx=2048` · `threads=8` ·
`ngl=99`

| Users | Reqs | RPS | P50 (ms) | P95 (ms) | P99 (ms) | Eff. concurrency | Failures |
|:--|--:|--:|--:|--:|--:|--:|--:|
| 10 | 82 | 1.38 | 3700 | 24000 | 26000 | 8.4 | 0.0% |
| 50 | 132 | 2.27 | 20000 | 22000 | 22000 | 39.4 | 0.0% |

*Effective concurrency = RPS x average latency (Little's Law) -- how many requests were
really in flight, regardless of how many users locust simulated. It counts queued requests
too, so the occupancy/slot ratio can legitimately exceed 1.0; it is occupancy, not
utilisation. For true slot utilisation use the server's own gauges (`make metrics`).*

## What these two runs say

| Going from 10 to 50 users | |
|:--|--:|
| Offered load | 5x |
| Throughput actually delivered | **1.64x** (33% of linear) |
| P95 latency | **0.92x** |
| Effective concurrency at 50 users | 39.4 vs `--parallel 4` slots (occupancy/slot ratio 9.86) |

**Saturated.** Throughput delivered only 1.64x for 5x the offered load, and effective concurrency (39.4) is at or above all 4 decode slots. Saturation sets in somewhere at or below 50 users; the load you added beyond that point became queue time rather than throughput.

P95 grew no faster than throughput (0.92x vs 1.64x), so this server still has headroom at 50 users.

## Your reading

**The server is already saturated at 10 users, not somewhere between 10 and 50.**
The auto-summary above ("1.64x throughput, P95 0.92x, still has headroom") is
misleading because the 10-user run is polluted by a cold start. From
`locust-10_stats_history.csv`: no request completed in the first ~15 s, and the
first ~15 requests took 23-26 s each (that is the whole P95 = 24000 / P99 = 26000).
This was the first time the server ran multi-slot batches, so the Vulkan backend
had to build pipelines for the new batch shapes -- the same one-off cost as the
825 ms first-request TTFT in `make bench`. After ~25 s the run is steady:

| Steady state (from `*_stats_history.csv`) | 10 users | 50 users |
|:--|--:|--:|
| RPS | 2.2-2.5 | 2.1-2.6 |
| P50 latency | 3.1-3.7 s | 20-21 s |

**The number that convinced me: RPS plateaus at ~2.3-2.4 in both runs.** 5x the
users bought ~1.0x the throughput in steady state; the 1.64x in the table only
exists because the cold start drags the 10-user average RPS down to 1.38.

**Queue time vs compute time (Little's Law).** With all 4 slots busy and
~2.35 req/s leaving, each request occupies a slot for about 4 / 2.35 ≈ **1.7 s**
of compute. At 50 users the P50 is ~20 s, so roughly **18 s (~90%) is queue
time**. That matches the server's own gauges in `02-server-batching-u50.md`:
`n_busy_slots_per_decode` peaked at 3.96/4 and `requests_deferred` hit 45 -- i.e.
4 requests decoding, ~45 waiting. Effective concurrency 39.4 (50 users) vs
4 slots is occupancy ≈ 10x the slot count: the extra latency is all waiting, not
slower decoding. Even at 10 users ~6 requests are queued, which is why P50 there
(3.5 s) is already ~2x the 1.7 s compute time.

**First knob for goodput@SLO: `--parallel` (4 → 8, with `LAB_N_CTX` 2048 → 4096 so
each slot keeps 512 tokens of context).** Decode on this GPU costs ~14 ms per step
mostly to stream the weights; adding sequences to the same step reuses that one
weight read, so tokens/s should scale with batch width until the step becomes
compute- or KV-bound. That raises the ceiling the queue is waiting behind. The
model is 2.97 GB of the 4 GB VRAM, so the KV cache for 8 slots is the limit to
watch. Second knob: admission control (cap concurrent users / reject past a queue
depth) -- requests that wait 18 s miss any interactive SLO anyway, so serving fewer
of them on time *raises* goodput even though raw throughput is unchanged.
