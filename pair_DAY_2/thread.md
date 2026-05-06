# Tweet Thread — Day 2

**Topic:** Why Tool Outputs Get Ignored Inside Agent Systems

---

**Tweet 1**
Your agent received the tool output. The constraint was in the prompt. The model still ignored it.

This isn't the model being dumb. There are two separate failures happening — and understanding both changes how you build every agent system going forward. 🧵

---

**Tweet 2**
Failure one: the context never reached the model at all.

In a real system I audited, 32 probe evaluations returned the same generic output: "Understood. Thanks for the reply."

That wasn't the LLM ignoring context. That was a hardcoded fallback firing because the code had no case for `reply_type = "objection"`. The model was never called.

---

**Tweet 3**
Failure two: policy constraints passed as plain text.

The agent received `Tone mode: exploratory` and `Claim strength: soft` as lines in a user prompt.

These aren't tool outputs. They're advisory text. The model's pretraining prior for "what a good sales email sounds like" competes with them directly — and the prior wins.

---

**Tweet 4**
The fix: force a scratchpad step.

Instead of `Tone mode: exploratory`, end your prompt with:

`Constraint check: Given tone_mode=exploratory, I must [complete this].`

The model now generates reasoning tokens that encode the constraint before producing the response. The prior no longer dominates — the constraint is in the generation path.

---

**Tweet 5**
The deeper issue: tool schema design.

A plain-text label the model can ignore is not a tool. A typed OpenAI-format tool schema forces the model to emit a structured JSON token sequence before proceeding — that's the difference between advice and a binding constraint.

Most agent systems skip this layer entirely.

---

**Tweet 6**
Full breakdown — two failure modes, the load-bearing mechanism, three concrete fixes, and the ReAct + Lost-in-the-Middle research behind it:

https://kidusgashaw.substack.com/p/why-tool-outputs-get-ignored-two

Sources: Liu et al. 2023 (arXiv:2307.03172) · Yao et al. 2022 (arXiv:2210.03629)

Full thread: https://x.com/Kidus5T99409/status/2052066470230659446?s=20
