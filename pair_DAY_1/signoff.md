# Sign-off Document

**Asker:** Kidus Gashaw
**Explainer:** Partner

## Question
**How do prefill and decode phases contribute differently to end-to-end latency during large-context evaluation, and what specific telemetry fields must I add to `outreach_generator.py` to isolate Time-To-First-Token when processing a dense competitor gap brief?**

## Gap Closure Status: CLOSED

The explainer closed my gap. I now understand that prefill is a single parallelized forward pass over all input tokens — it scales with context length and is the bottleneck when the prompt is dense. Decode is autoregressive, one token per forward pass, and scales with output length. These require different optimizations and cannot be diagnosed from a single end-to-end latency metric.

The explainer gave me the two specific fields I needed: `ttft_ms` (time from API call to first decode token) and `tokens_per_second` (throughput across the decode phase). I added both to `outreach_generator.py` and Langfuse traces now isolate which phase to optimize before I touch the code.

**Signed:** Kidus Gashaw
**Date:** 2026-05-09
