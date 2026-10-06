# 01 - Measure: latency baseline

Model `Gemma 4 E2B` · host `Windows-AMD64` · llama.cpp `b10488`
Settings: `threads=4` `ngl=99` `ctx=2048`
`max_tokens=64` · warm-up discarded
Completed requests: `UD-Q4_K_XL` 10/10 · `UD-Q2_K_XL` 10/10

| Quantization | Size (GB) | Load (ms) | TTFT P50/P95 (ms) | TPOT P50/P95 (ms) | E2E P50/P95/P99 (ms) | Decode (tok/s) |
|:--|--:|--:|--:|--:|--:|--:|
| UD-Q4_K_XL | 2.97 | 7433 | 873 / 3233 | 151.0 / 667.6 | 10316 / 43790 / 43790 | 6.6 |
| UD-Q2_K_XL | 2.24 | 8536 | 1435 / 4172 | 858.4 / 938.3 | 49649 / 62268 / 62268 | 1.2 |

- **TTFT** = prefill. Short prompts keep it small; long-context RAG is where it explodes.
- **TPOT** = per-output-token decode cost, bounded by memory bandwidth. `decode tok/s = 1000 / TPOT_p50`.
- `UD-Q2_K_XL` decodes **5.50x SLOWER** than `UD-Q4_K_XL` here, despite being 0.73 GB smaller. The measured result suggests that Q2's extra dequantization work outweighs its memory savings on this setup; the hardware probe detects a 2 GB MX230, but this benchmark does not report how many layers actually offloaded.

## Your observation

Q2 is 0.73 GB smaller but decodes 5.5x slower in this run, so I prefer Q4 on my
machine. The same short smoke question produced a similar answer from both; that
single prompt is not a broad quality evaluation. The speed result is unusual for a
smaller quant. Dequantization overhead is one plausible explanation; thermal effects,
paging or background load could also contribute. I would repeat under stable conditions
before treating this speed ratio as a lasting property of these formats.
