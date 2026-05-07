# Day 3 Question — Training and Post-Training Mechanics

In Week 10 I built SCAP — a mechanism that appends signal-confidence instructions to the system prompt before the agent generates outreach. It reduced signal over-claiming (P-005 cluster, pre-fix trigger rate 0.70) but did not eliminate it. In Week 11 I trained a LoRA judge to catch what SCAP missed. I justified the training by saying "the agent does not know when it is wrong" — but I cannot explain what training the LoRA actually changed in the model weights that SCAP's prompt instructions could not.

**The question:** When a prompt-based constraint like SCAP partially fixes a failure but leaves residual violations, what is the mechanistic difference between what that prompt is doing at inference time and what fine-tuning a LoRA adapter would change in the weights — and why does one work where the other does not?

**Grounded in:** `probes/target_failure_mode.md` (P-005 trigger rate 0.70, SCAP selected as the prompt-time fix) and `method/method.md` (SCAP mechanism). The method.md explicitly says SCAP was chosen because it "directly attacks the highest-ROI failure mode that can be reduced by a prompt-time mechanism" — but I never explained why a prompt-time mechanism has a ceiling that training does not, which is the gap I am naming.

**Why it generalizes:** Any FDE who builds a constrained agent will hit this same decision point: add another prompt instruction, or fine-tune. Understanding what these two approaches actually change — one at inference time, one in the weights — tells you when prompt engineering is sufficient and when training is the only fix.
