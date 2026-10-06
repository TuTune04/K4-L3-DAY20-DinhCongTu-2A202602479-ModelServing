# 02 - Serve: load test + saturation reading

Host `Linux-x86_64` · llama.cpp `b10488` ·
`--parallel 4` · `ctx=2048` · `threads=1` ·
`ngl=0`

| Users | Reqs | RPS | P50 (ms) | P95 (ms) | P99 (ms) | Eff. concurrency | Failures |
|:--|--:|--:|--:|--:|--:|--:|--:|
| 10 | 14 | 0.24 | 32000 | 51000 | 51000 | 6.7 | 0.0% |
| 50 | 14 | 0.25 | 39000 | 57000 | 57000 | 7.4 | 0.0% |

*Effective concurrency = RPS x average latency (Little's Law) -- how many requests were
really in flight, regardless of how many users locust simulated. It counts queued requests
too, so the occupancy/slot ratio can legitimately exceed 1.0; it is occupancy, not
utilisation. For true slot utilisation use the server's own gauges (`make metrics`).*

## What these two runs say

| Going from 10 to 50 users | |
|:--|--:|
| Offered load | 5x |
| Throughput actually delivered | **1.04x** (21% of linear) |
| P95 latency | **1.12x** |
| Effective concurrency at 50 users | 7.4 vs `--parallel 4` slots (occupancy/slot ratio 1.84) |

**Saturated.** Throughput delivered only 1.04x for 5x the offered load, and effective concurrency (7.4) is at or above all 4 decode slots. Saturation sets in somewhere at or below 50 users; the load you added beyond that point became queue time rather than throughput.

Throughput moved 1.04x while P95 moved 1.12x. That gap is the goodput argument: past saturation you buy throughput by spending latency, and if your SLO is a P95 target then the requests you added are no longer being served within it. (This lab does not fix an SLO number for you -- pick one in your write-up and state how much goodput you keep at it.)

> **Small sample.** Only 14 requests completed in the
> shorter run, so these percentiles are indicative rather than solid. Note also that
> locust averages only *completed* requests: when the run ends with requests still
> queued, effective concurrency is an **under**-estimate. Trust the throughput-scaling
> row over the concurrency row here, and run longer (`-t 3m`) if you want firmer numbers.

## Your reading

From 10 to 50 users, RPS moved from 0.24 to 0.25: the generated ratio is
1.04x, only 21% of linear scaling. P95 rose from 51000 to 57000 ms, or
1.12x. That nearly flat throughput is the strongest saturation evidence.
Effective concurrency was already 6.7 at 10 users and reached 7.4 at 50,
both above --parallel 4 because occupancy includes queued requests. Together
with 4.00 busy slots and 46 deferred requests in the batching report, this
suggests capacity pressure already at 10 users and clear saturation by 50;
these runs cannot locate the precise knee below that range.

The extra P95 is consistent with queue time: full slots force arriving requests
to wait, while the workload and compute per request remain roughly constant.
TPOT is shared across at most 4 active slots on the measured 2 logical vCPU
(1 physical core). CPU-only decode is constrained by memory bandwidth as weight
bytes are streamed per token; increasing slots can improve batching throughput
while worsening each request's TPOT. These runs do not separately measure queue
time or TPOT, so the queue attribution is an inference supported by deferred
requests, rather than a direct timing decomposition.

I would first apply admission control to limit queued work for goodput at a
latency SLO. Rejecting excess demand promptly protects accepted requests from
unbounded waiting; increasing --parallel first would add contention on this CPU.
Shorter max_tokens is another useful option when answer quality permits it.
Both runs completed 14 requests with 0 failures (0.0%). The short samples omit
unfinished requests, underestimate occupancy, and make tail percentiles uncertain;
shared Codespace host noise also limits the precision of this comparison.
