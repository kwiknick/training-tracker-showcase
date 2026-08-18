# AI/LLM engineering notes

This is a deeper look at how the LLM is actually used in the daily briefing pipeline described in the [main README](./README.md) — the prompt design, the reliability pattern for content that has to appear correctly every time, and the reasoning behind the model choice.

## The pattern: compute first, generate second

The model is never handed raw API responses. Everything it sees has already been computed and validated in plain Python before the prompt is built:

- Pace and heart-rate zones, derived from recent activity streams
- A training-load rollup (TRIMP-based, elevation-adjusted) with fitness/fatigue/form values
- A 7-day schedule audit that has already matched planned workouts against actual recorded activity and flagged specific anomalies
- Readiness signals from a Coros watch (HRV, sleep score, resting HR), when available

The LLM's job is narrow: turn a well-defined bundle of already-correct numbers and flags into a well-written, well-structured HTML email. It is not asked to do arithmetic, look anything up, or decide whether an anomaly occurred — those are computed deterministically upstream and handed over as facts. This keeps the LLM in the role it's actually good at (synthesis and natural-language framing) and out of the role it's unreliable at (precise numeric reasoning and consistent formatting of facts it wasn't given verbatim).

## Grounding beyond the day's numbers

The prompt also includes curated reference material relevant to that specific day: effort-zone terminology definitions, fueling principles, and any supplemental protocol documents referenced by that day's workout (e.g. a specific heat-adaptation or hill-training protocol). This is a simple, deliberately low-tech form of retrieval — the plan data itself names which reference documents apply to a given day, so there's no embedding search or vector database involved. The workout schedule *is* the retrieval index. This is a case where the simplest solution that fits the actual data shape beat reaching for a generic RAG framework.

## Reliability: don't ask the model to reproduce facts, splice them in instead

The most important lesson embedded in this design: **for any content that must appear exactly, or must not be editorialized, don't put it in the prompt as an instruction — exclude it from the model's job entirely and assemble it in code afterward.**

Early iterations tried to get the model to reliably reproduce certain fixed content (specific protocol instructions, certain must-include notes) by describing it carefully in the system prompt. That approach is fundamentally probabilistic — a model instructed to "include X verbatim" will usually do it, but "usually" is not a property you want for anything user-facing on a recurring basis. The fix was architectural, not a better prompt: the pipeline now generates the free-text narrative sections via the model, and separately splices in the fixed/must-appear sections (protocol text, certain structured data blocks) directly in Python after the model responds — string composition, not prompt engineering. The model is simply never given the opportunity to get those parts wrong, because it's never asked to produce them.

This generalizes: **treat prompt instructions as a request, not a guarantee.** Anything where correctness matters more than fluency should be moved out of the model's responsibility and into ordinary code, even if it means the model's job gets a little more constrained.

## Model selection

The system uses a smaller, cheaper Claude model rather than defaulting to the largest available one. The workload is one structured-to-HTML generation per user per day — a few thousand input tokens (the assembled prompt) and roughly a thousand output tokens. At that volume, model capability well beyond "reliable synthesis of clearly-structured input" is unnecessary, and the cost difference compounds daily. This is a case where matching model size to task complexity, rather than defaulting to the most capable model available, was the actual engineering decision — not a limitation, a choice.

## Cross-day context

Rather than treating each day's briefing as stateless, the pipeline persists a small amount of state between runs — prior coaching notes and adjustment context — so the model can reference *what changed and why* across recent days instead of only ever reacting to a single day's snapshot in isolation. This is a lightweight, purpose-built form of memory: not a general-purpose conversation history, just the specific fields that make the next day's briefing coherent with the last one.

---

**Nicholas Willard** — [github.com/kwiknick](https://github.com/kwiknick) · [linkedin.com/in/nicholas-willard](https://linkedin.com/in/nicholas-willard)
