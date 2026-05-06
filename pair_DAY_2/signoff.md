# Asker Sign-Off — Day 2

**Asker:** Amare Kassa
**Gap closure judgment:** Closed

Before this explainer I understood that my probe outputs were generic and attributed it to the LLM ignoring structured constraints. I now understand two things I did not before.

First: my probe evaluation results were not measuring the LLM at all. The missing `objection` case in `generate_followup_email()` meant every probe fired the hardcoded fallback. My Act III pass rates were measuring a deterministic fallback function, not my agent's constraint-following behaviour. That is a fundamental problem with how I interpreted my own results, and it changes what my probe library actually proves.

Second: the difference between a plain-text policy label and a typed tool schema is not a formatting preference — it is the difference between advice the model can ignore and a generation constraint it must satisfy before producing output. My `policy_result` dict serialised to `Tone mode: exploratory` was never a tool output. It was a prompt suggestion competing against a strong pretraining prior. Understanding the mechanism — next-token prediction, pretraining prior dominance, forced scratchpad as the fix — gives me a concrete intervention I can apply to the generation layer rather than relying on fallback rules to enforce policy.

The one outstanding item before full closure: real model outputs from running the before/after scratchpad prompts against a live model.
