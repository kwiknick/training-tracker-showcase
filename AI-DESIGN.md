# AI/LLM engineering notes

This document looks closer at how the LLM works inside the daily briefing pipeline described in the [main README](./README.md), covering the prompt design, the reliability pattern for content that must appear correctly every time, and the reasoning behind the model choice.

## The pattern: compute first, generate second

The model never sees raw API responses. Every input gets computed and validated in plain Python before the prompt gets built:

- Pace and heart-rate zones, derived from recent activity streams
- A training-load rollup (TRIMP-based, elevation-adjusted) with fitness/fatigue/form values
- A 7-day schedule audit, already matching planned workouts against recorded activity and flagging specific anomalies
- Readiness signals from a Coros watch (HRV, sleep score, resting HR), when available

The model's job stays narrow: turn a bundle of already-correct numbers and flags into a well-written, well-structured HTML email. The model does no arithmetic, looks nothing up, and never decides whether an anomaly occurred. Code computes those facts upstream and hands them over ready to use. This setup keeps the LLM working in its strength, turning structured facts into clear prose, and out of its weak spot, precise numeric reasoning and consistent formatting of facts it never received directly.

## Grounding beyond the day's numbers

The prompt also carries reference material for that specific day: effort-zone definitions, fueling principles, and any supplemental protocol documents tied to the day's workout, such as a heat-adaptation or hill-training protocol. This works as a simple, low-tech form of retrieval. The plan data names which reference documents apply to a given day, so no embedding search or vector database gets involved. The workout schedule *is* the retrieval index. The simplest solution matching the data shape beat a generic RAG framework here.

## Reliability: don't ask the model to reproduce facts, splice them in instead

The core lesson from this design: **for content that must appear exactly, or must never get rewritten, keep it out of the prompt entirely and assemble it in code afterward.**

Early versions tried to get the model to reproduce fixed content, specific protocol instructions and required notes, by describing the rules carefully in the system prompt. That approach stays probabilistic. If you tell a model to include text word for word, it usually complies, but "usually" fails as a standard for content your users see every day. The real fix changed the architecture, not the prompt. The pipeline now generates the free-text narrative through the model, and separately splices the fixed sections, protocol text and required data blocks, into the output directly in Python after the model responds. This works as string composition, not prompt engineering. The model never gets the chance to get those parts wrong, because the pipeline never asks it to produce them.

This generalizes: **treat prompt instructions as a request, not a guarantee.** Move anything where correctness matters more than fluency out of the model's responsibility and into ordinary code, even when doing so narrows the model's job.

## Model selection

The system runs on a smaller, cheaper Claude model instead of the largest one available. The workload stays simple: one structured-to-HTML generation per user per day, a few thousand input tokens for the assembled prompt, and roughly a thousand output tokens for the reply. At that volume, the model needs reliable synthesis of clearly structured input and nothing more. The cost difference compounds daily at that scale. Matching model size to task complexity, instead of defaulting to the most capable model, was the real engineering decision here, not a limitation forced on the project.

## Cross-day context

Each day's briefing avoids full statelessness. The pipeline saves a small amount of state between runs, prior coaching notes and adjustment context, so the model can reference what changed and why across recent days, instead of reacting to one day's snapshot alone. This works as a lightweight, purpose-built form of memory, not a general-purpose conversation history. The pipeline stores only the specific fields that keep the next day's briefing consistent with the last one.

---

**Nicholas Willard**. [github.com/kwiknick](https://github.com/kwiknick) · [linkedin.com/in/nicholas-willard](https://linkedin.com/in/nicholas-willard)
