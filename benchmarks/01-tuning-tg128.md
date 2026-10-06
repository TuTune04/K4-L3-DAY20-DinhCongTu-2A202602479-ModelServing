# 01 - Tune: thread-count sweep

Model `Qwen3.5-0.8B-Q4_K_M.gguf` · host `Linux-x86_64` · llama.cpp `b10488`
CPU: **1 physical · 2 logical** cores · `ngl=0` · metric `tg128`

| threads (-t) | tg128 (tok/s) | vs best |
|:--|--:|--:|
| 1 | 19.0 | 96% |
| 2 | 19.8 | 100% |

**Best**: `-t 2` at 19.8 tok/s
**Slowest tested**: `-t 1` at 19.0 tok/s (1.04x spread)
**Against the physical-core default** (`-t 1`, 19.0 tok/s): 1.04x

Use this in your run:

```bash
LAB_N_THREADS=2 make bench
```

## Your explanation

The best tested point, or observed knee within this sweep, is `-t 2` at exactly
19.8 tg128 tok/s. The physical-core default, `-t 1`, measured 19.0 tok/s in the
report (19.02 tok/s in the JSON); moving to the best setting gives the reported
1.04x speedup. Hardware reports 1 physical core and 2 logical cores, so these
2 vCPU likely represent hyper-threads sharing one core's execution units and
L1/L2 caches. The modest improvement contradicts the expected peak at the
physical-core count: a second thread may hide memory stalls or improve execution
utilization, and shared, virtualised Codespace host noise may also influence
this small difference. Decode is memory-bandwidth bound because weight bytes
must be streamed for each token; extra threads help only until the memory bus
is saturated. Oversubscription adds synchronization and barrier cost per token
and contention with the OS and Codespace processes. This sweep tests only
1 and 2 threads, so it does not establish a saturation point or measure a drop
beyond the best tested setting.
