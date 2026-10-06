# 02 - Continuous batching under load (u50)

Host `Linux-x86_64` · `--parallel 4` · 27 samples over
60s at 2.0s intervals · raw CSV: `02-server-metrics-u50.csv`

| Gauge | Peak observed |
|:--|--:|
| `n_busy_slots_per_decode` (avg/decode) | 4.00 of 4 slots (100%) |
| `requests_processing` | 4 |
| `requests_deferred` | 46 |
| `kv_cache_usage_ratio` | n/a — not exported by llama.cpp `b10488` |
| `tokens_predicted_total` (final) | 1745 |

Highest sampled value was **4.00 of 4** slots. Note this gauge is llama.cpp's *average* busy slots per decode step, so the number below is the highest average we sampled, not an instantaneous maximum batch width. A peak near 1 means
requests were served one at a time -- either the load was too light to overlap, or
they arrived too far apart. A peak approaching `--parallel` means the scheduler was
genuinely packing concurrent requests into shared decode steps.
`requests_deferred` went above zero: more requests arrived than there were slots, so some waited. That wait is the queue time in your P95.

## Your observation

The highest sampled average busy slots per decode step was 4.00 of 4 slots
(100%), with requests_processing peaking at 4 and requests_deferred at 46.
This confirms continuous batching under the overlapping load. The 50-user
report gives effective concurrency of 7.4, exceeding the available slots because
Little's Law counts queued requests as well as requests being decoded. I trust
the server gauge for slot utilisation: it measures useful work in decode steps,
whereas effective concurrency measures occupancy of the whole system. The gauge
is an average per decode step, not an instantaneous batch-width measurement.
The completed-request latency estimate also misses unfinished queued requests,
so the deferred gauge provides stronger direct evidence of the queue.
