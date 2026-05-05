# Explainer Thread

**Tweet 1:**
My partner fine-tuned a model to write outreach emails as an external agent. Every time a hiring signal appeared, it wrote as the company instead — "We are hiring, here is a job opening…"

The instruction was in the prompt. The model had read it. So why was it being ignored?

**Tweet 2:**
The model didn't ignore the instruction. It lost a competition.

At every decode step, attention scores are passed through a softmax — a distribution that sums to 1. That makes attention zero-sum. If hiring-context tokens score high, the instruction's weight is suppressed by the same normalisation. It's still in context. It just lost.

**Tweet 3:**
Same prompt. One variable changed: where the instruction sits.

Condition A — instruction at the top, 2,000 tokens of hiring context below → model outputs company voice from token 1.

Condition B — instruction repeated immediately before generation → model outputs agent voice.

Same words. Different position. Completely different output.

**Tweet 4:**
Two things make hiring signals especially hard to override:

1. Training prior — hiring tokens appear almost exclusively in company-authored text in pretraining data. The model learned to complete them in company voice before your instruction ever had a chance.

2. Cascade — one drifted token reshapes the query for the next step. A single slip compounds into a fully drifted output.

**Tweet 5:**
The fix depends on which layer is broken.

Consistent drift on one signal type → training data problem. Audit and relabel.

Drift that worsens with longer inputs → attention attenuation. Repeat the instruction adjacent to generation + add a negative example ("Do NOT write like: 'We are hiring…'").

**Tweet 6:**
Full breakdown of the mechanism, the side-by-side example, and when each fix applies:

https://open.substack.com/pub/kidusgashaw/p/why-your-instruction-gets-ignored?r=8bo4le&utm_campaign=post&utm_medium=web&showWelcomeOnShare=true
