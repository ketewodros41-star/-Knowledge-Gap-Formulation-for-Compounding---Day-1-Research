# Sign-off — Day 3

**Asker:** Kidus Gashaw | **Gap closure judgment:** Closed

## What I understand now that I did not before

Before this explainer I knew SCAP reduced signal over-claiming but could not explain why it did not eliminate it. I described the residual failures as the agent "not knowing when it is wrong" without being able to name the mechanism behind that.

The explainer closed the gap with one key distinction: prompting is runtime steering — it adds context tokens that shift next-token probabilities for that run, but the model's weights are unchanged and all its pretrained tendencies remain intact. When SCAP's constraint competed with the model's fluency and plausibility objectives on a low-signal input, those objectives could still win because the underlying weight function had not changed. LoRA changes the computation itself: W_effective = W + BA means every forward pass through the adapted layers produces different hidden states and logits, regardless of what the prompt says. This is a persistent correction stored in parameters, not a conditional reminder in context.

I can now explain why SCAP plateaued at a trigger rate above zero on P-005: the pretrained representation did not cleanly separate compliant from non-compliant signal-grounding cases, so the prompt instruction could bias but not reliably override the model's output distribution on borderline inputs. The LoRA judge I trained in Week 11 was the right response — not because SCAP was poorly written, but because the failure required moving a decision boundary in weight space, which prompting cannot do.

The concrete edit: the mechanistic explanation now lives in `method/method.md` where SCAP was previously described only as "a prompt-time mechanism" with no explanation of its ceiling.
