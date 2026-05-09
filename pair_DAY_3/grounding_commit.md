# Grounding Commit — Day 3

## Artifact edited

`method/method.md` — added a paragraph at the end of the SCAP mechanism description titled **"Why SCAP Has a Ceiling"** — **[commit e3c601c](https://github.com/ketewodros41-star/The-Conversion-Engine/commit/e3c601c)**

## What changed

The original `method.md` described SCAP as "a prompt-time mechanism that directly attacks the highest-ROI failure mode that can be reduced by a prompt-time mechanism" — correct but circular. It named SCAP's limitation without explaining it.

The new paragraph reads:

> SCAP reduces average signal over-claiming by adding confidence-calibration instructions to the system prompt before generation. It does not eliminate the failure because prompting is runtime steering: the model's weights are unchanged, and all pretrained tendencies — toward fluency, plausibility, and confident assertion — remain intact. When the instruction competes with those tendencies on a low-signal input, the instruction can be outweighed. The residual violations in P-005 (trigger rate above zero after SCAP) are the signature of this: borderline inputs where the model's internal representation did not cleanly separate over-claiming from grounded language, so the prompt biased but could not reliably override the output distribution. The Week 11 LoRA judge addresses this gap not by adding a better instruction but by changing the effective weights (W_effective = W + BA) so that inputs resembling the failure pattern map to different internal representations on every forward pass — a persistent correction in weight space rather than a conditional reminder in context.

## Why this change matters

`method.md` is the document a future engineer or the Tenacious team would read to understand why the system is designed the way it is. The previous version left the prompt ceiling unexplained, which meant the rationale for adding a trained judge in Week 11 had no mechanistic foundation in the Week 10 artifact. The new paragraph closes that gap and makes the design chain — SCAP first, LoRA judge second — defensible on first principles.
