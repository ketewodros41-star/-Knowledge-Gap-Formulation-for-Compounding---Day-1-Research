# Canonical Sources

1. **Xiao, G., et al. (2023). "Efficient Streaming Language Models with Attention Sinks."** (arXiv:2309.17453)
   - *Why it's load-bearing:* This paper introduces the concept of the "Attention Sink"—the mathematical phenomenon where the model disproportionately dumps high attention scores onto the very first tokens of a sequence to stabilize the Softmax denominator. This establishes the structural protection of position 0, which sits in contrast to the manufactured proximity of SCAP v2.

2. **Wallace, E., et al. (2024). "The Instruction Hierarchy: Training LLMs to Prioritize Privileged Instructions."** (arXiv:2404.13208)
   - *Why it's load-bearing:* This paper shows that fine-tuning teaches models to treat the system-prompt *position* as a trust signal. It explicitly explains the training-time equivalent of what SCAP v2 solves at inference time (giving constraint tokens structural, rather than purely semantic, authority).
