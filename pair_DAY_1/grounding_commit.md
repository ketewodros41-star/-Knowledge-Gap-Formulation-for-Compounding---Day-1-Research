# Grounding Commit — Day 1

**Asker:** Kidus Gashaw
**Artifact edited:** `agent/outreach_generator.py` in the Week 10 Conversion Engine (`C:\Users\Davea\Downloads\trp week 10\agent\outreach_generator.py`)

**What changed:**

Added separate telemetry fields to `outreach_generator.py` to isolate Time-To-First-Token from total API response time. Before the edit, Langfuse traces recorded only one coarse metric — total API wait time — which made it impossible to tell whether latency spikes came from prefill (the model reading the dense competitor gap brief and firmographic context) or decode (the model writing the email). The edit adds a `ttft_ms` field logged at the moment the first token arrives, and a `tokens_per_second` field calculated over the decode phase, so both phases are visible in the trace separately.

**Why:**

My partner's explainer clarified that prefill and decode are fundamentally different operations — prefill scales with context length while decode scales with output length — and that optimising for the wrong phase wastes effort. My `outreach_generator.py` stuffs the prompt with Crunchbase firmographics, job-post velocity, layoffs.fyi data, and a competitor gap brief before generating the email. Without isolating TTFT, I was guessing whether to shorten the input context or the output. The edit means I can now read the trace and know which phase is the bottleneck before deciding where to optimise.
