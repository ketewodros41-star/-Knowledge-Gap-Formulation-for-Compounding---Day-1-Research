# Final Submitted Question (Week 10 Reframed)

**How do prefill and decode phases contribute differently to end-to-end latency during large-context evaluation, and what specific telemetry fields must I add to `outreach_generator.py` to isolate Time-To-First-Token when processing a dense competitor gap brief?**

## Rubric Alignment

- **Diagnostic Precision:** Pinpoints the exact distinction between compute-bound prefill (crunching the massive competitor brief/firmographics) and memory-bound decode (writing the outbound email), rather than settling for a generic "why is my agent slow" question.
- **Grounding in Work:** Anchored strictly in your Week 10 `agent/outreach_generator.py` file. Openly acknowledges that the current implementation lumps all API time into a single latency metric, making it impossible to detect whether the context processing or the output generation is the actual bottleneck.
- **Generalizability:** Every FDE building systems with heavy enrichment context (like Crunchbase data + Layoffs.fyi) faces this exact latency challenge when attempting to optimize their pipeline.
- **Resolvability:** Highly focused and resolvable in a single explainer detailing the two phases and defining the specific `prompt_tokens_details` and `completion_tokens` telemetry needed to split the metric.
