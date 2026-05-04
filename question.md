# Final Submitted Question (Week 10 Reframed)

**How do prefill and decode phases contribute differently to end-to-end latency during large-context evaluation, and what specific telemetry fields must I add to `outreach_generator.py` to isolate Time-To-First-Token when processing a dense competitor gap brief?**

## My Gap

I chose this gap because I built `agent/outreach_generator.py` to generate personalized outbound emails, and that forced me to stuff the prompt with Crunchbase firmographics, job-post velocity, layoffs.fyi data, and dense competitor briefs.

When I checked my Langfuse traces, I saw high latency but only one coarse metric: total API wait time. That means I still cannot tell whether the slowdown comes from prefill, where the model reads the long context, or decode, where it writes the email. Until I log Time-To-First-Token and tokens-per-second separately in `outreach_generator.py`, I am guessing when I try to optimize the agent.
