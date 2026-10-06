# 03 - Integrate: RAG pipeline run

Host `Windows-AMD64` · llama.cpp `b10488` ·
retrieval backend: **keyword overlap** · 3 queries

| Query | Contexts retrieved | embed (ms) | retrieve (ms) | llm (ms) | total (ms) |
|:--|--:|--:|--:|--:|--:|
| Why is goodput more useful than raw throughp... | goodput, paged, radix | 0.0 | 0.0 | 990.2 | 990.3 |
| What problem does PagedAttention actually so... | paged, radix, disagg | 0.0 | 0.0 | 560.8 | 560.9 |
| When does splitting prefill and decode help?... | disagg, radix, batching | 0.0 | 0.0 | 595.7 | 595.8 |

Mean per stage (ms): embed **0.0** · retrieve **0.0** ·
llm **715.6** · total **715.7**
Dominant stage: **llm** (100% of total)

## Answers returned

**Why is goodput more useful than raw throughput?**

> Goodput@SLO counts only the requests per second that met the TTFT and TPOT targets. Throughput at saturation ignores SLOs.

**What problem does PagedAttention actually solve?**

> PagedAttention stores the KV cache in non-contiguous pages, removing the internal fragmentation that wasted most GPU memory.

**When does splitting prefill and decode help?**

> Splitting prefill and decode helps because prefill is compute-bound and decode is memory-bandwidth-bound.


## Which N16-N19 pieces are real

| Day | Piece | Real or stub |
|:--|:--|:--|
| N16 Cloud/IaC | none -- everything runs on my laptop | stub |
| N17 Data pipeline | `TOY_DOCS` hard-coded in `pipeline.py` | stub |
| N18 Lakehouse | no storage layer, docs live in memory | stub |
| N19 Vector + features | keyword-overlap `retrieve()`, no embedding server | stub |
| N20 Serving | `llama-server` (Gemma 4 E2B UD-Q4_K_XL, Vulkan, `--parallel 4`) | **real** |

`embed` = 0.0 ms and `retrieve` = 0.0 ms are honest for the stub: no embedding model
is called and keyword overlap over 6 short docs is microseconds. So `llm` being 100%
of the total is expected here, not a finding about RAG in general.

**The finding is in the llm stage itself.** The first run used the default
`http://localhost:8080` and measured llm = **2873 ms** mean, while the server's own
timings were only ~0.6 s (prefill 113-150 tok in 185-323 ms + decode 23-30 tok in
327-414 ms). The missing ~2.2 s is not the model: on Windows `localhost` resolves to
`::1` first, `llama-server` listens only on `127.0.0.1`, and the client waits ~2 s
before falling back to IPv4. Measured directly: `GET /health` takes 2020-2042 ms via
`localhost` vs 1-2 ms via `127.0.0.1`. Because `pipeline.py` opens a new connection
per query, every query paid it. Re-running with `--base-url http://127.0.0.1:8080`
(the table above) gives llm = **716 ms** mean -> **4.0x** faster with no model change.

The second run also shows prefix caching: prefill dropped to 5 tokens per query
because the identical system+context prompt was already in the slot's KV cache from
the first run, so only decode (~310-430 ms for 23-30 tokens, ~14 ms/token) remains.

**To halve it again:** the remaining cost is decode, so cap `max_tokens` / ask for
shorter answers (latency is linear in output tokens at ~14 ms each), and keep the
shared prompt prefix stable so prefill stays cached. A real N19 embedding model would
add an `embed` stage of its own; it should run on the same warm server (`--embedding`)
over a reused connection, not a fresh one per call.
