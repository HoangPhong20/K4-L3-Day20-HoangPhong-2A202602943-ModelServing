# 02 - Serve: load test + saturation reading

Host `Windows-AMD64` · llama.cpp `b10488` ·
`--parallel 4` · `ctx=2048` · `threads=4` ·
`ngl=99`

| Users | Reqs | RPS | P50 (ms) | P95 (ms) | P99 (ms) | Eff. concurrency | Failures |
|:--|--:|--:|--:|--:|--:|--:|--:|
| 10 | 7 | 0.14 | 13000 | 47000 | 47000 | 3.5 | 0.0% |
| 50 | 7 | 0.14 | 14000 | 51000 | 51000 | 3.7 | 0.0% |

*Effective concurrency = RPS x average latency (Little's Law) -- how many requests were
really in flight, regardless of how many users locust simulated. It counts queued requests
too, so the occupancy/slot ratio can legitimately exceed 1.0; it is occupancy, not
utilisation. For true slot utilisation use the server's own gauges (`make metrics`).*

## What these two runs say

| Going from 10 to 50 users | |
|:--|--:|
| Offered load | 5x |
| Throughput actually delivered | **0.94x** (19% of linear) |
| P95 latency | **1.09x** |
| Effective concurrency at 50 users | 3.7 vs `--parallel 4` slots (occupancy/slot ratio 0.92) |

**Saturated.** Throughput delivered only 0.94x for 5x the offered load, and effective concurrency (3.7) is at or above all 4 decode slots. Saturation sets in somewhere at or below 50 users; the load you added beyond that point became queue time rather than throughput.

Throughput moved 0.94x while P95 moved 1.09x. That gap is the goodput argument: past saturation you buy throughput by spending latency, and if your SLO is a P95 target then the requests you added are no longer being served within it. (This lab does not fix an SLO number for you -- pick one in your write-up and state how much goodput you keep at it.)

> **Small sample.** Only 7 requests completed in the
> shorter run, so these percentiles are indicative rather than solid. Note also that
> locust averages only *completed* requests: when the run ends with requests still
> queued, effective concurrency is an **under**-estimate. Trust the throughput-scaling
> row over the concurrency row here, and run longer (`-t 3m`) if you want firmer numbers.

## Your reading

The server is saturated by 50 users: a fivefold increase in offered users delivered only
0.94x the throughput, while P95 rose from 47 to 51 seconds. At 50 users, effective
concurrency was 3.7 against 4 slots; the metrics run independently showed a peak of
3.62 busy slots, all 4 requests processing, and up to 46 deferred. The throughput
plateau and deferred requests are the clearest evidence of saturation and queueing.
These estimates are uncertain because only 7 requests completed in each run, and
queued requests at the end are not counted.

For this lab I use a provisional P95 SLO of 60 seconds: all 7 completed samples in
each run were below it, leaving observed completed-request goodput around 0.14
requests/second at both loads. This is a weak estimate with such a small sample. To
improve goodput at a tighter P95 target, I would first test a lower output-token cap:
decode occupies the slots, so shorter generations may release them sooner and reduce
queueing, at the cost of shorter answers.