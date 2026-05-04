# Evening Call Summary

**Asker:** Me
**Explainer:** Partner

## What We Talked About

**My question to my partner:** I explained my Week 10 question about prefill versus decode latency and why I need to measure those phases separately in my outbound pipeline. I pointed to the gap in `outreach_generator.py`: I only have one latency bucket, so I cannot tell whether the slow part is prompt processing, Time-To-First-Token, or the actual token generation.

**My partner's question to me:** My partner explained her Week 12 question about instruction drift during inference. She asked how attention dynamics can make a distant instruction weaker in a long prompt and what other mechanisms, like learned instruction hierarchy and local decoding pressure, determine whether the instruction is preserved or overridden.

## Final Outcome

By the end of the call, each of us understood the other person's question clearly enough to write for it. The discussion was about explaining our own questions to each other, not reviewing the final explainer draft. That made the scope of both questions sharper and more diagnostic.