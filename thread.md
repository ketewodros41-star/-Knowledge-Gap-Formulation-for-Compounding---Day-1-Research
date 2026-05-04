# Explainer Thread

**Tweet 1:**
Why does an instruction like "hedge if confidence is low" get weaker in a long outbound prompt? Not because the model literally forgets it, but because generation is a fresh competition over what should matter right now.

**Tweet 2:**
In transformers, each new token is chosen by comparing the current query against prior keys, then softmaxing the result. That means attention is always competitive. Long context gives the model more signals to weigh, and distant instructions can lose salience.

**Tweet 3:**
The evidence is not "attention weight proves behavior." The more defensible claim is narrower: longer prompts make it harder for a distant instruction to remain the most salient control signal when the model is deciding the next token.

**Tweet 4:**
Two other mechanisms matter too: training-time instruction hierarchy and local decoding pressure. A system-prompt rule starts with some privilege, but a strong nearby pattern can still override it during generation.

**Tweet 5:**
In my Week 10 Tenacious pipeline, [agent/main.py](c:/Users/Davea/Downloads/trp%20week%2010/agent/main.py) and [agent/outreach_generator.py](c:/Users/Davea/Downloads/trp%20week%2010/agent/outreach_generator.py) assemble a large evidence bundle before the email is written. That makes instruction drift plausible when the policy lives only at the top of the prompt.

**Tweet 6:**
The right test is an ablation: same prospect, same signal bundle, compare baseline vs long-context vs adjacent-constraint prompts, then measure policy adherence directly. If the adjacent constraint improves hedging, that is evidence that proximity helps.

**Tweet 7:**
SCAP v2 is an application-layer workaround, not a weight change. It repeats the policy immediately before generation so the constraint is salient at decision time. That is the practical lesson: keep the rule close, then measure whether behavior improves.
