# 03 - Integrate: RAG pipeline run

Host `Windows-AMD64` · llama.cpp `b10488` ·
retrieval backend: **keyword overlap** · 3 queries

| Query | Contexts retrieved | embed (ms) | retrieve (ms) | llm (ms) | total (ms) |
|:--|--:|--:|--:|--:|--:|
| Why is goodput more useful than raw throughp... | goodput, paged, radix | 0.0 | 0.2 | 8124.6 | 8124.8 |
| What problem does PagedAttention actually so... | paged, radix, disagg | 0.0 | 0.1 | 7757.2 | 7757.3 |
| When does splitting prefill and decode help?... | disagg, radix, batching | 0.0 | 0.1 | 11687.3 | 11687.5 |

Mean per stage (ms): embed **0.0** · retrieve **0.1** ·
llm **9189.7** · total **9189.9**
Dominant stage: **llm** (100% of total)

## Answers returned

**Why is goodput more useful than raw throughput?**

> Goodput@SLO counts only the requests per second that met the TTFT and TPOT targets. Throughput at saturation ignores SLOs.

**What problem does PagedAttention actually solve?**

> PagedAttention stores the KV cache in non-contiguous pages, removing the internal fragmentation that wasted most GPU memory.

**When does splitting prefill and decode help?**

> Splitting prefill and decode helps because prefill is compute-bound and decode is memory-bandwidth-bound.


## N16-N19 implementation status and latency finding

For this CP6 pipeline, **N16 is stubbed** (localhost only), **N17 is stubbed** (no
N17 batch job is connected; sample documents are loaded in memory), **N18 is stubbed**
(`TOY_DOCS` is an in-memory list of toy documents, with no lakehouse), and **N19 is stubbed** (`TOY_DOCS` plus keyword
overlap; no vector index or feature store). **N20 is real**: requests go to the local
`llama-server`. The LLM-dominated latency was expected. To cut total latency in half,
I would first reduce generated tokens if shorter answers are acceptable; retrieval
averages only 0.1 ms, so optimizing it would barely affect this run.
