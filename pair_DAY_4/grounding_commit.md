# Grounding Commit — Day 4

**Artifact edited:** `C:\Users\Davea\Downloads\trp week 11\methodology_rationale.md`
**Asker making the edit:** kidus tewodros

**Section revised — Preference Pair Construction Rationale:**

Added the following paragraph after the existing Score Gap Threshold section:

> **Length Bias Check (added post-Week-12 Day 4 review)**
>
> Chosen outputs (Claude Sonnet rewrites) and rejected outputs (original agent failures) were not length-matched before training. This structural asymmetry — a more capable model rewriting weaker outputs — means chosen outputs are likely longer and more elaborately structured, creating a potential length confound in the training signal. Before deploying the trained judge as a rejection-sampling layer in production, run Test 1: compute the mean word count of chosen vs. rejected outputs and the Wilcoxon signed-rank p-value for the length difference across the 69 pairs. If the ratio exceeds 1.3x and p < 0.05, run Test 2: compute Spearman ρ between output word count and judge score on the 40 held-out tasks. ρ > 0.3 indicates length is a real driver of judge scores and the adapter should be retrained on length-matched pairs before production deployment. If both tests clear, Delta B = +0.3204 can be attributed to rubric learning rather than verbosity bias.

**What changed and why:** The original `methodology_rationale.md` described the preference pair construction and the preference leakage rotation (Claude generates, Qwen judges) but contained no analysis of whether chosen and rejected outputs differ systematically in length. Mistire's explainer identified this as the specific gap that makes Delta B uninterpretable as evidence of rubric learning without the length check. The edit adds the two concrete tests to run and the decision rule for whether retraining is needed — turning a gap in the existing documentation into an actionable v0.2 checklist item.
