# kimi-tide: A Study of Dual-Model Collaboration — Architecture, Cost, and Feasibility

> **Subject**: the open-source project [tafcear/kimi-tide](https://github.com/tafcear/kimi-tide) ("moon-tide") — a routing plugin for the DeepSeek Harness (DSH) ecosystem that lets **Kimi** and **DeepSeek** models divide work automatically, and the "dual-model development loop" methodology the project itself practices and documents. This study covers three threads: the plugin's routing and guardrail **architecture**, the **cost/loss structure** of the "cheap generator + strong reviewer" pattern, and the **feasibility boundary** of that pattern.
>
> **Authors**: [tafcear](https://github.com/tafcear) (author of kimi-tide) × Kimi (AI research collaboration)
> Study date: 2026-08-21 ｜ Latest release: v0.4.0 (2026-08-20) ｜ v0.5.0 rule-driven routing design finalized, not yet implemented.

## Abstract

kimi-tide is an engineering experiment built around one question: **how to pick the right model for every step of an agent session**. Inside DeepSeek Harness (DSH), an "everything-is-a-plugin" agent host, the plugin routes cheap, fast DeepSeek V4 to daily text work and multimodal, long-context Kimi K3 to hard problems and image tasks — showing the reason for every choice live on a panel. Within one week of August 2026 the project went through five phases: a self-built OAuth access layer (0.1.x) → a dual-model router (0.2.x) → a six-dimension capability scoring engine (0.3.0) → convergence onto the official API-key path with ~740 lines of self-built code deleted (0.4.x) → a finalized design for rule-driven routing that retires the scoring engine (0.5.0). This study argues the project's value exceeds the plugin itself: first, it provides a **complete decision record of a routing system retreating from score-driven to rule-driven** — a rare sample of how academic LLM-routing paradigms (FrugalGPT, RouteLLM, Hybrid LLM) get trimmed in real engineering; second, its own development process demonstrates a **"implementer writes code, independent model reviews, tests catch the rest, recheck accepts"** collaboration loop — about 55 findings closed across three applications, all archived and re-checkable.

## 1 What the plugin does

DSH binds a session to one model start to finish. But DeepSeek V4 is cheap and fast yet cannot see images, while Kimi K3 is multimodal with a 1M context and strong coding — at quota and cost. kimi-tide is the automatic traffic controller for that trade-off: per-step routing decisions hooked on DSH's official `agent/pre-step` / `agent/request` waterfall events, a heuristic classifier (keywords/length → dimension weights), a weighted capability score with a cost penalty (`score = Σ weight[dim] × scores[dim] − λ × costTier`), three modes (`off` / `cost` with a sliding-window premium budget / `capability`), a three-layer image guard (per-step rerouting, host admission probe, session latching), and full decision observability on a dock panel.

Every capability score in the baseline table carries an **evidence grade** — primary benchmark / inferred / pending — written into source comments: "inference never masquerades as fact." Kimi K3's code score (4.7) traces to SWE-bench Verified 93.40% on an independent leaderboard; DeepSeek V4 figures are marked as third-party-aggregate, pending verification.

## 2 The collaboration loop (methodology)

The project was itself built by two models: **DeepSeek implements → Kimi reviews independently (read-only) → DeepSeek fixes with tests → Kimi rechecks**. Three applications:

| Round | Object | Scale | Outcome |
|---|---|---|---|
| 1 (2026-08-15) | Code | 23 findings + 5 new in recheck | 28 closed, 5/5 tests green, "usable maturity" |
| 2 (2026-08-17) | Design docs | R1 13 → R2 7 → R3 7 items | Plan finalized v2.2 after three rounds |
| 3 (2026-08-18) | Implementation | 28 sub-agent dispatches (11 implement / 10 verify / 5 scope-check / 2 final) | No open findings, 154/154 green |

Key evidence: the root-cause EEXIST bug was caught by **tests**, not by the reviewer — "the review judged the symptom; the test dug out the lesion." And 3 of the 5 recheck findings were doc-code desync, "which only independent recheck systematically catches."

## 3 The cost of being reviewed (the acceptor side)

What does it cost a cheap primary model to *absorb* quality control from a strong model, versus the strong model doing the work alone? Five loss categories:

1. **Direct token cost — bounded and favorable**: review loops cost 2–3× tokens of single-shot (adversarial loops up to 4–220×), but premium-priced tokens only pay for review *slices* (task brief + diff list + graded findings), never full generation. Kimi K2.7 Code prices at $0.95/$4.00 per 1M tokens make this leverage real.
2. **Latency — the most certain loss**: each review roundtrip is a full strong-side generation + fix + recheck; industry experience turns 10 minutes of solo work into 30–45.
3. **Residual quality gap — the most hidden**: reviews are item-by-item checks, not rewrites. Review recall is <100% (the EEXIST case; industry AI review tools detect ~42–48% of bugs), and fix fidelity is capped by the cheap model's repair ability. Weaver (Stanford, 2025) shows weak generators + verifier ensembles can match strong reasoning models (87.7% vs o3-mini's 86.7%) — but on *selection* tasks; code review is "fix the wrong," and the bottleneck sits with the cheap model.
4. **Semantic friction — the core acceptor-side cost**: kimi-tide's loop documents the implementer verifying every finding rather than accepting wholesale; ~2 of 23 initial findings were semantic disagreements between review and implementation, arbitrated by test runs. Industry false-positive rates for AI review: 5–15%; 40% of alerts ignored after fatigue. kimi-tide's answer — test arbitration + mandatory recheck — turns "accept" into "**verified acceptance**."
5. **Context reconstruction**: each review round starts a fresh context; self-contained task briefs cost human + token effort.

The theoretical criterion is the generation-verification gap (GV-gap): tasks where verification is easier than generation (code review, test-backed engineering) have large GV-gaps and favor the loop; tasks where verification is as hard as generation (fact recall, open-ended writing, architectural taste) have GV-gap ≈ 0 — just use the strong model. Small models' *self-review* GV-gap is non-positive, which is exactly why a heterogeneous external reviewer matters.

## 4 Feasibility boundary

The pattern is feasible under five conditions, and degrades past each boundary:

1. **The task must be verifiable** (hardest constraint) — can you write an objective acceptance checklist for the deliverable?
2. **A deterministic backstop must exist** — reviews never stand alone; without tests or human acceptance, the pattern is infeasible.
3. **Scale past the break-even point** — small one-shot tasks make loop overhead pure waste; review-token-to-quality gain is logarithmic (Weaver), so an optimal review budget exists.
4. **Host and quota must hold** — Kimi Code enforces a 5-hour rolling window (300–1200 calls) plus weekly quota; DSH is developer preview with breaking changes; independent multi-agent architectures amplify errors 17.2× and fail >40% in production without orchestration. kimi-tide's four mitigations — read-only reviewer, test arbitration, decision trails, off-mode escape hatch — are exactly the right medicine.
5. **Process discipline, not just technology** — only 12% of agent projects reach sustained production; the successes share process discipline, none share a technical breakthrough.

**Verdict**: this is not a universal quality enhancer, but a precision instrument that trades process discipline for quality leverage on verifiable tasks. Recommended adoption order: verify task verifiability → add deterministic backstop → calibrate break-even with small deliverables → then scale automation.

## 5 The v0.5.0 pivot: retiring the scoring engine

The finalized v0.5.0 design replaces the six-dimension scoring engine with **presets = default model + N ordered rules** (conditions: has-image / named keyword groups; first match wins; miss → preset default). The scoring machinery (λ, thresholds, budget windows, slider UI) enters the deletion list. Motivations: the parameter space was too heavy for end users; a capability score table is a perishable asset (models drift monthly); and keyword-rule routing was already a "red ocean" in the DSH community — differentiation moved to the preset-management layer. The cost: the sliding-window premium budget — a rare engineering implementation of budget-constrained routing — has no successor in the rule world.

Positioned against the academic landscape (FrugalGPT's up-to-98% cost reduction, Hybrid LLM's ~40% fewer expensive calls, RouteLLM's >2× cheaper at parity): kimi-tide is the **minimal non-learning member** of the routing family — heuristic classification + a human-auditable score table — distinguished by per-step granularity inside agent tool loops, stateful correctness guardrails (image latching), and decision observability as a first-class feature. Its clearest gap versus academic routers: no offline evaluation on a RouterBench-style benchmark.

## 6 Takeaways

1. **The durable value of dual-model collaboration lives in mechanisms, not score tables** — guardrails, observability, audit trails, and migration discipline survived three major versions; the scoring engine did not.
2. **Heterogeneous independent review is among the highest-ROI multi-model collaboration forms** — no training, no parallel inference; just context isolation, tool constraints, and task-brief engineering.
3. **Per-step routing is a new problem domain for conversational agents** — stateful correctness issues (modality latching, irreversible image history) have no counterpart in stateless query-routing literature.
4. **Honesty is an engineering asset** — from the positioning statement ("don't build on rented time") to evidence grading and upstream bug reports, every claim in the project is third-party checkable. This study itself is a beneficiary and verification of that fact.

---

> **Notes**: Based on a full read of the public repo tafcear/kimi-tide (MIT) at commit a144739 (2026-08-20) plus public academic and engineering literature (FrugalGPT arXiv:2305.05176; RouteLLM arXiv:2406.18665; MoA arXiv:2406.04692; Multiagent Debate arXiv:2305.14325; Weaver arXiv:2506.18203; Mind the Gap arXiv:2412.02674). Internal project facts follow repo documentation; model benchmarks prefer independent measurements. Research process: initiated and framed by tafcear (project author); research, analysis, and writing by Kimi — a human-AI collaboration. For research reference only; not investment, procurement, or compliance advice.
