# 01 - Tune: thread-count sweep

Model `gemma-4-E2B-it-UD-Q4_K_XL.gguf` · host `Windows-AMD64` · llama.cpp `b10488`
CPU: **8 physical · 16 logical** cores · `ngl=0` · metric `tg128`

| threads (-t) | tg128 (tok/s) | vs best |
|:--|--:|--:|
| 1 | 8.9 | 48% |
| 4 | 18.4 | 100% |
| 8 | 12.3 | 67% |
| 16 | 5.2 | 28% |
| 32 | 2.9 | 16% |

**Best**: `-t 4` at 18.4 tok/s
**Slowest tested**: `-t 32` at 2.9 tok/s (6.44x spread)
**Against the physical-core default** (`-t 8`, 12.3 tok/s): 1.49x

Use this in your run:

```bash
LAB_N_THREADS=4 make bench
```

## Your explanation

**This table is the CPU-only sweep** (`LAB_N_GPU_LAYERS=0 make tune`). I ran it on
purpose, because the default sweep with the model fully offloaded to the GPU
(`ngl=99`, Vulkan, RTX 3050 Ti) is almost flat and says nothing about threads:

| threads (-t), `ngl=99` | 1 | 4 | 8 | 16 | 32 |
|:--|--:|--:|--:|--:|--:|
| tg128 (tok/s) | 70.0 | 73.6 | 74.9 | 74.9 | 74.8 |

With every layer on the GPU the CPU threads only feed the GPU queue and sample, so
1 thread is already 93% of best.

**The `-t 4` = 18.4 tok/s point is not reproducible.** Re-measuring with
`llama-bench -ngl 0 -t 2,4,6,8 -r 3` gave 10.8 ± 0.9 / 12.6 ± 0.8 / 12.8 ± 0.3 /
12.2 ± 1.3 tok/s. The first sweep ran `-t 4` second, while the laptop was still
cool and boosting; I treat 18.4 as an outlier, not the knee. The honest reading:

- **1 → 4 threads: climbs** (8.9 → ~12.6). A single core cannot issue enough loads
  to fill the memory bus.
- **4 → 8 threads: flat** (~12.2-12.8, inside the noise). This is the knee. Decode
  must stream every weight once per token; ~12.5 tok/s x the bytes touched per
  token is roughly what dual-channel DDR4 delivers in practice. Once the bus is
  full, extra cores just wait on the same memory channels.
- **8 → 16 → 32: collapses** (12.3 → 5.2 → 2.9). Above 8 physical cores, the extra
  threads are SMT siblings sharing one core's L1/L2 and execution ports, and at 32
  the OS time-slices two threads per logical CPU. llama.cpp synchronizes all
  threads at a barrier after every op, so each step waits for the slowest,
  preempted thread -- oversubscription turns into pure stall time.

So on this machine the thread knob is worth **1.0x on the GPU path** and only
matters on the CPU path, where the right setting is 4-8 threads (not 16 logical)
and the wrong one (16) costs 2.4x. The change that actually mattered was moving
decode off the CPU entirely: 12.3 tok/s (`ngl=0`, `-t 8`) → 74.9 tok/s
(`ngl=99`, `-t 8`) = **6.1x** -- see REFLECTION §5.
