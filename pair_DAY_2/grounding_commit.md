# Grounding Commit — Day 2

**Asker:** Kidus Gashaw
**Artifact edited:** `method.md` in the Week 10 Conversion Engine (`C:\Users\Davea\Downloads\trp week 10\method\method.md`)

**What changed:**

Rewrote the bench-gated constraint paragraph in `method.md`. The original paragraph used language that implied the model was making an active tool-calling decision — describing the system as an agent that chooses whether to use a staffing tool. Amare's explainer named exactly what needed to change. The old framing:

> "an autonomous agent choosing whether to use a staffing tool."

The revised paragraph now reads:

> "a runtime-enforced enrichment pipeline where scaffolding executes the bench availability check before generation and injects the result into model context. The model never decides whether the check runs — the runtime owns that execution, making the constraint deterministic rather than probabilistic."

This reflects what the code in `agent/main.py` actually does: `check_bench_availability()` is called unconditionally by the scaffolding before the model is involved. The model cannot bypass it through prompt phrasing or reasoning drift because it never owned the decision in the first place.

**Why:**

Before Amare's explainer, I described the bench-gated constraint as if the model were choosing to call a tool. That framing was wrong — it implied the constraint was probabilistic when it is actually deterministic. The edit makes `method.md` honest about what mechanism is doing the enforcing, and converts a claim I could not defend into one I can.
