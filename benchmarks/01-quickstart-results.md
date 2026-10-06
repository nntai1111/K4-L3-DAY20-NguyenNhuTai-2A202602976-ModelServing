# 01 - Measure: latency baseline

Model `Gemma 4 E2B` · host `Windows-AMD64` · llama.cpp `b10488`
Settings: `threads=8` `ngl=99` `ctx=2048`
`max_tokens=64` · warm-up discarded
Completed requests: `UD-Q4_K_XL` 10/10 · `UD-Q2_K_XL` 10/10

| Quantization | Size (GB) | Load (ms) | TTFT P50/P95 (ms) | TPOT P50/P95 (ms) | E2E P50/P95/P99 (ms) | Decode (tok/s) |
|:--|--:|--:|--:|--:|--:|--:|
| UD-Q4_K_XL | 2.97 | 9403 | 303 / 915 | 14.6 / 14.7 | 1210 / 1809 / 1809 | 68.3 |
| UD-Q2_K_XL | 2.24 | 6985 | 312 / 1166 | 15.8 / 16.2 | 1291 / 2150 / 2150 | 63.1 |

- **TTFT** = prefill. Short prompts keep it small; long-context RAG is where it explodes.
- **TPOT** = per-output-token decode cost, bounded by memory bandwidth. `decode tok/s = 1000 / TPOT_p50`.
- `UD-Q2_K_XL` decodes **1.08x SLOWER** than `UD-Q4_K_XL` here, despite being 0.73 GB smaller. That is a real result, not a mistake: fewer bits only buys speed when decode is limited by memory bandwidth. On a machine that is compute-limited instead — few cores, no GPU offload — the extra dequantization work of a heavily-quantized format can cost more than the bytes it saves. Say which case yours is.

## Your observation

**On this machine 2-bit is not worth it.** It is 0.73 GB (25%) smaller and loads
faster (7.0 s vs 9.4 s), but it is **slower**: TPOT P50 is 15.8 ms vs 14.6 ms, so
decode drops from 68.3 to 63.1 tok/s (1.08x slower). TTFT P50 is a wash (312 vs
303 ms); the large P95 values (915 / 1166 ms) are the first request of each run
paying one-off warm-up of the Vulkan pipelines, not steady-state prefill. An earlier
run of the same benchmark gave the same ordering (70.8 vs 67.9 tok/s, 1.04x), so
the gap is small but repeatable.

Why no speedup: the whole model sits in the RTX 3050 Ti's 4 GB VRAM (`ngl=99`,
Vulkan backend). At ~15 ms/token the GPU is not saturating its ~192 GB/s of
bandwidth on a 2-3 GB file; a big share of each step is fixed per-token overhead
(kernel dispatch across ~35 layers, sampling, HTTP streaming) that does not shrink
with fewer bits, while Q2_K's more complex dequantization adds work. Fewer bytes
only pay off when bytes are the bottleneck -- here they are not.

Quality check (same 3 prompts, `temperature=0`, 4-bit via `make serve`, 2-bit via
`serve.py --compare`): both got the arithmetic right (3 h 35 min). 4-bit's
explanation of decode was tighter (named FLOPs vs memory bus explicitly) and it
answered the translation request in Vietnamese; 2-bit was wordier and switched to
English for the explanation. Slower and slightly worse -> keep **UD-Q4_K_XL**.
2-bit only makes sense when the 4-bit file does not fit in VRAM/RAM at all.
