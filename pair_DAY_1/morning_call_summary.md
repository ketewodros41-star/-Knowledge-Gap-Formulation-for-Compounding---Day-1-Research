# Morning Call Alignment Summary

**Asker:** Me
**Explainer:** Partner

---

## Evolution of the Question

**Original Draft:** "How do I make my streaming agent faster?"

**Interrogation:** My partner immediately pushed back on this. She asked: *"Faster in what sense: Time-To-First-Token, total completion time, or downstream tool latency?"* She correctly pointed out that 'faster' is not a diagnostic metric we can measure. She asked me to open my Week 5 Ledger schema (`src/models/events.py`).

**The Realization:** When I opened `events.py` and looked at my `AgentNodeExecuted` class, I realized I had only coded it to track coarse status, not the telemetry needed to separate model latency from surrounding workflow latency.

**The Pivot:** We realized that since LLM inference has two fundamental phases - Prefill and Decode - a streaming agent can be "slow" in completely different ways (Time-To-First-Token vs. tokens-per-second). Because my schema does not log those separately, I have no idea which bottleneck I should optimize.

---

## What I Talked About

I explained my system setup — `outreach_generator.py` in the Tenacious Conversion Engine — and the problem I had observed. To generate personalised outbound emails, the system assembles a large enrichment block per prospect: Crunchbase firmographics, job post velocity, Layoffs.fyi signals, and a dense competitor gap brief. That pushes the input prompt well past 2,000 tokens per request, while the output is only around 150 tokens. When I checked my Langfuse traces, generation latency was consistently high, but I had no way to explain it because my telemetry was a single stopwatch — `total_latency = end - start` — logged after the API call returned. I knew something was slow. I had no idea whether the cost was in processing the input or in generating the output.

I also raised the competitor gap brief specifically as the suspected culprit, since it is the largest and most variable part of the prompt. My question became: if the brief is driving the latency, how do I confirm that, and what do I need to add to the code to measure it directly?

---

## What My Partner Talked About

She opened by separating prefill and decode as distinct problems with different cost profiles. Prefill is when the model processes the entire input simultaneously — attention is computed across all token pairs, which has quadratic complexity with respect to input length. That means doubling the prompt roughly quadruples the prefill cost. Decode is sequential and cannot be parallelised: the model generates one token at a time, each step depending on all prior tokens. Decode is memory-bandwidth bound, not compute-bound, and scales with output length rather than input length. Given my input-to-output ratio (2,000+ tokens in, ~150 out), she said prefill was almost certainly the dominant cost — but only measurement would confirm it.

She then walked through exactly what telemetry I need to add to `outreach_generator.py`. The fix is to move from a single stopwatch to a streaming measurement that captures three timestamps: request start, first token received, and last token received. From those, TTFT (`first_token_time - start_time`) isolates the prefill phase. Decode time (`end_time - first_token_time`) and tokens per second (`output_tokens / decode_time`) characterise generation. She said to log these five fields together in every trace: `ttft`, `decode_time`, `tokens_per_second`, `input_token_count`, and `output_token_count`. Once those are in place, I can correlate input token count against TTFT directly — if larger briefs produce higher TTFT, prefill is confirmed as the bottleneck and reducing the brief is the right intervention. If TTFT stays flat regardless of input size, the latency is coming from somewhere else and the investigation changes.

---

## How My Explainer Helped Her

Her question going into the call was about instruction drift — specifically why her Week 11 model kept generating emails from the company's perspective despite an explicit instruction to write as an external outreach agent. She knew what the failure looked like but did not understand why it was happening or what to do about it.

My explainer gave her a way to separate two problems she had been treating as one. Before the call she assumed the drift was a prompt-engineering failure — that she had written the instruction badly or placed it wrong. What my explainer showed her is that there are two distinct layers at work. The first is a training-data problem: if the fine-tuning examples paired hiring signals with company-voice outputs even a small percentage of the time, the model learned that association in its weights and will activate it at inference regardless of what the instruction says. The second is an attention-salience problem: at each decode step, attention is a zero-sum competition, and an instruction placed far from the generation site loses to nearby company-context tokens on both semantic and positional distance simultaneously. She had no framework to tell these two causes apart, which meant she had no way to choose the right fix.

The prefill/decode split she had just explained to me turned out to directly support the argument in my explainer. Because she understood that the instruction is processed during prefill but must keep winning during every decode step, she immediately grasped why proximity matters — repeating the instruction immediately before generation puts it back in the high-attention zone at the moment it needs to win. She said this reframed the problem for her: it was not that the instruction was absent, it was that the instruction had already lost the competition before the first output token was generated.

She also said the cascade mechanism — where a single drifted token reshapes the query vector for the next step — explained something she had noticed but could not account for: the drift was never partial. The model did not start in agent voice and slip halfway through. It produced company voice from the first word, consistently. Understanding that one early token changes the attention context for every subsequent token made that pattern make sense.

The practical outcome for her was a clearer diagnostic path: check the training data first to see if hiring-signal examples had company-voice outputs, then test whether repeating the instruction adjacent to generation reduces drift, and only if both of those fail, retrain on corrected data.

---

## Final Decision

We narrowed the gap explicitly around measuring **Prefill vs. Decode Latency** in `AgentNodeExecuted`. By the end of the explainer, I need to know exactly how those two phases behave mechanically and exactly what telemetry fields to add to track them separately.

*Visible movement achieved: Migrated from a generic "why is it slow" question to a surgical telemetry gap in my Week 5 event sourcing codebase.*
