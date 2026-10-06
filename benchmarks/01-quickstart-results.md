# 01 - Measure: latency baseline

Model `Qwen3.5 0.8B` · host `Linux-x86_64` · llama.cpp `b10488`
Settings: `threads=1` `ngl=0` `ctx=2048`
`max_tokens=64` · warm-up discarded
Completed requests: `Q4_K_M` 10/10 · `UD-Q2_K_XL` 10/10

| Quantization | Size (GB) | Load (ms) | TTFT P50/P95 (ms) | TPOT P50/P95 (ms) | E2E P50/P95/P99 (ms) | Decode (tok/s) |
|:--|--:|--:|--:|--:|--:|--:|
| Q4_K_M | 0.50 | 4388 | 577 / 683 | 56.9 / 61.9 | 4178 / 4542 / 4542 | 17.6 |
| UD-Q2_K_XL | 0.39 | 4173 | 953 / 1141 | 60.6 / 62.3 | 4787 / 4954 / 4954 | 16.5 |

- **TTFT** = prefill. Short prompts keep it small; long-context RAG is where it explodes.
- **TPOT** = per-output-token decode cost, bounded by memory bandwidth. `decode tok/s = 1000 / TPOT_p50`.
- `UD-Q2_K_XL` decodes **1.07x SLOWER** than `Q4_K_M` here, despite being 0.11 GB smaller. That is a real result, not a mistake: fewer bits only buys speed when decode is limited by memory bandwidth. On a machine that is compute-limited instead — few cores, no GPU offload — the extra dequantization work of a heavily-quantized format can cost more than the bytes it saves. Say which case yours is.

## Your observation

Q4_K_M has TPOT P50 56.93 ms (56.9 ms in the table) and decode throughput
17.6 tok/s; UD-Q2_K_XL has TPOT P50 60.60 ms (60.6 ms in the table) and
16.5 tok/s. Relative to Q4, Q2 increases TPOT by 6.45% and lowers decode
throughput by 6.25%. Its reported size falls from 0.50 GB to 0.39 GB:
0.11 GB, or 22.00%, less, calculated from the rounded report sizes. TTFT P50
rises from 576.9 ms to 953.0 ms (577 versus 953 ms in the table), a
376.1 ms increase, or 65.19%.

For the identical question, “Explain in two sentences why LLM decode speed is
limited by memory bandwidth,” Q4 incorrectly attributes the bottleneck to
fetching data from the “external network.” Q2 mentions data transfer, but repeats
“creates a bottleneck in the transfer of information” and ends mid-sentence at
the configured token limit; it also fails the requested sentence constraint.
Neither answer correctly explains streaming model weights, so this single
question provides no evidence of a useful quality improvement from Q2.

The smaller quant is not worth it on this machine for latency: it saves disk
space but slows decode and first-token delivery. Decode streams all weights per
token, making the bytes of weights transferred through memory a potential
bandwidth bottleneck. Fewer bytes improve speed only if the extra dequantisation
cost of the lower-bit kernels does not consume that saving. Hardware detection
reports one physical core and two logical CPUs, with no GPU; the benchmark uses
one thread. The measured slowdown is consistent with dequantisation compute
cost outweighing memory savings here, although this run cannot isolate that
cause. These measurements come from a shared, virtualised Codespace host and
are subject to scheduling noise.
