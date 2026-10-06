# 03 - Integrate: RAG pipeline run

Host `Linux-x86_64` · llama.cpp `b10488` ·
retrieval backend: **keyword overlap** · 3 queries

| Query | Contexts retrieved | embed (ms) | retrieve (ms) | llm (ms) | total (ms) |
|:--|--:|--:|--:|--:|--:|
| Why is goodput more useful than raw throughp... | goodput, paged, radix | 0.0 | 0.0 | 12172.2 | 12172.3 |
| What problem does PagedAttention actually so... | paged, radix, disagg | 0.0 | 0.0 | 5582.0 | 5582.1 |
| When does splitting prefill and decode help?... | disagg, radix, batching | 0.0 | 0.1 | 8891.0 | 8891.1 |

Mean per stage (ms): embed **0.0** · retrieve **0.0** ·
llm **8881.7** · total **8881.8**
Dominant stage: **llm** (100% of total)

## Answers returned

**Why is goodput more useful than raw throughput?**

> Based on the provided context, **Goodput is more useful than raw throughput** because it focuses on the actual **requests per second (RPS) that met the Target Time-to-Fill (TTFT) and Target Time-to-Poll (TPOT) targets**, whereas raw throughput ignores SLOs (Service Level Objectives).

The text explicitly states:
> "Goodput@SLO counts only the requests per second that met the TTFT and TPOT targets.

**What problem does PagedAttention actually solve?**

> Based on the context provided, **PagedAttention** solves the problem of **internal fragmentation** in GPU memory.

The context explicitly states that PagedAttention "stores the KV cache in non-contiguous pages, removing the internal fragmentation that wasted most GPU memory."

**When does splitting prefill and decode help?**

> Splitting prefill and decode helps when **prefill is compute-bound and decode is memory-bound**.

This is because the context states that prefilling the model requires significant computation, while decoding requires significant memory bandwidth. By splitting these operations into separate pools (prefill and decode), the system can utilize different hardware resources more effectively: the compute


## Which N16-N19 pieces are real

- N16 — stub: no cloud provisioning or IaC is invoked; inference runs on the local llama-server.
- N17 — stub: no ingestion or data pipeline is built; documents are hard-coded in TOY_DOCS.
- N18 — stub: no lakehouse or persistent storage is queried; documents live in a Python list.
- N19 — stub for vector retrieval: the measured backend is keyword overlap, with no embedding endpoint or vector index. The RAG prompt construction and HTTP call to the local LLM are real.

Across the 3 queries, mean embed was 0.0 ms, retrieve 0.0 ms, LLM 8881.7 ms,
and total 8881.8 ms. The dominant LLM stage was 100% of total at the report's
precision. These rounded retrieval values do not imply free retrieval: one query
recorded 0.1 ms. This matches my expectation because embedding is skipped and
retrieval only scores the toy documents, while inference processes retrieved
context and generates an answer on the CPU.

To halve latency, I would attack the LLM stage: request fewer output tokens,
trim irrelevant retrieved context, and investigate prompt caching for a shared
system/context prefix when that prefix actually repeats. The first query spent
3287.9 ms on prefill and 8823.262 ms decoding 151 tokens, so reducing decode
alone would leave substantial prefill cost. The earlier Q4_K_M benchmark measured
TPOT P50 of 56.93 ms; each avoided output token removes roughly that measured
per-token cost, although this pipeline's timings can differ. CPU decode streams
weights through memory per token and is memory-bandwidth bound. Context reduction
and reusable prefix caching target prefill; neither guarantees a halving without
a new measurement. Shared Codespace host noise also limits timing comparisons.
