# Sources — Day 4

## Canonical Papers

**1. Dror, R., Baumer, G., Shlomov, S., and Reichart, R. (2018). "The Hitchhiker's Guide to Testing Statistical Significance in Natural Language Processing." ACL 2018.**
The practical reference for statistical testing in NLP and ML evaluation. Section 4 covers McNemar's test for paired binary comparisons — the correct test for comparing two conditions on the same task set. Section 5 covers bootstrap and permutation tests for aggregate metrics. Directly addresses the question of which test applies to pass-rate comparisons in agent benchmarks.
URL: https://aclanthology.org/P18-1128

**2. Cohen, J. (1988). "Statistical Power Analysis for the Behavioral Sciences." 2nd edition. Lawrence Erlbaum Associates.**
The canonical reference for effect size measures and power calculations. Chapter 6 covers Cohen's h (the effect size measure for proportions used in the MDE calculation above) and the arcsine transformation that makes proportion comparisons scale-independent. The MDE formula and the n≈122 calculation in the explainer derive directly from the formulas in this chapter.

## Tool / Pattern Used

**Python `statsmodels.stats.power` + `scipy.stats.fisher_exact`** — standard scientific Python libraries.
- `NormalIndPower().solve_power()` computes required sample size and MDE for two-proportion comparisons using Cohen's h
- `proportion_effectsize()` computes Cohen's h from two raw proportions
- `fisher_exact()` computes exact p-values for small-n 2×2 tables (used for F2 and F4 per-family analysis)

All code in the explainer runs without additional dependencies. Verified against Mistire's actual numbers: p1=0.146 (6/41), MDE≈18pp at n=41, Fisher's exact p≈0.50 for both F2 and F4.
