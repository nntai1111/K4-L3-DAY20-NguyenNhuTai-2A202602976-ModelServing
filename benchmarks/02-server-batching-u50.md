# 02 - Continuous batching under load (u50)

Host `Windows-AMD64` · `--parallel 4` · 15 samples over
60s at 2.0s intervals · raw CSV: `02-server-metrics-u50.csv`

| Gauge | Peak observed |
|:--|--:|
| `n_busy_slots_per_decode` (avg/decode) | 3.96 of 4 slots (99%) |
| `requests_processing` | 4 |
| `requests_deferred` | 45 |
| `kv_cache_usage_ratio` | n/a — not exported by llama.cpp `b10488` |
| `tokens_predicted_total` (final) | 13513 |

Highest sampled value was **3.96 of 4** slots. Note this gauge is llama.cpp's *average* busy slots per decode step, so the number below is the highest average we sampled, not an instantaneous maximum batch width. A peak near 1 means
requests were served one at a time -- either the load was too light to overlap, or
they arrived too far apart. A peak approaching `--parallel` means the scheduler was
genuinely packing concurrent requests into shared decode steps.
`requests_deferred` went above zero: more requests arrived than there were slots, so some waited. That wait is the queue time in your P95.

## Your observation

Peak batch width was **3.96 of 4 slots**, and it stayed between 3.92 and 3.96 for
all 15 samples while `make load-50` was running: every decode step was packing ~4
concurrent requests. That is continuous batching working -- with one-at-a-time
serving this gauge would sit at 1.00 (which is exactly what `make smoke` showed on
an idle server).

It does **not** match the effective concurrency of 39.4 in `02-server-results.md`,
and it should not: Little's Law counts every request in the system, queued or not,
while this gauge counts only requests inside a decode step. The difference is the
queue, and the server reports it directly -- `requests_deferred` was 42-45 for most
of the window (dropping to 19 as locust stopped spawning new requests at the end).
4 decoding + ~45 deferred ≈ 49-50 users. I trust this gauge for *batch width*
(it is the server's own count) and Little's Law for *how long people wait*; read
together they say the 4 slots are full and ~90% of the latency is queueing.
