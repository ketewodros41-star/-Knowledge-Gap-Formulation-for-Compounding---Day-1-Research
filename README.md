# Today Submission

This folder contains the pair-day submission for the Week 12 inference-time mechanics topic.

The package is built around one sharpened question and the artifacts that show how it was refined, answered, and signed off.

## What's Inside

- [question.md](question.md) - the final question for the day.
- [morning_call_summary.md](morning_call_summary.md) - the question-sharpening notes from the morning call.
- [explainer.md](explainer.md) - the explainer written in response to the partner's question.
- [thread.md](thread.md) - the short public version of the explainer.
- [evening_call_summary.md](evening_call_summary.md) - the evening call summary.
- [signoff.md](signoff.md) - the final gap-closure judgment.
- [question_sources.md](question_sources.md) - the canonical sources used in the explainer.

## Public Artifacts

- **Blog post:** [Why Your Instruction Gets Ignored: Attention Dynamics and Instruction Drift in LLMs](https://open.substack.com/pub/kidusgashaw/p/why-your-instruction-gets-ignored?r=8bo4le&utm_campaign=post&utm_medium=web&showWelcomeOnShare=true)
- **Tweet thread:** [https://x.com/Kidus5T99409/status/2051633404085502442?s=20](https://x.com/Kidus5T99409/status/2051633404085502442?s=20)

## Grounding

The question is grounded in the Week 10 conversion engine, especially the latency tracking gap in [agent/outreach_generator.py](c:/Users/Davea/Downloads/trp%20week%2010/agent/outreach_generator.py). The explainer focuses on separating prefill and decode latency so Time-To-First-Token can be measured directly instead of being hidden inside one coarse API timing bucket.
