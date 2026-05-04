# Thesis

> Working thesis lives here. Researcher reads this every run and uses it to
> decide whether new material supports / extends / challenges / is orthogonal
> to your current view. Researcher will *report* contradictions but never
> edit this file — thesis changes are always your decision.

## Working thesis

The bottleneck for post-deployment agent improvement is not model capability or
evaluation accuracy, but the missing bridge between the trace stream that
production agents emit and the preference / SFT data that alignment pipelines
consume. That bridge is best built as a four-layer stack — structured tracing
(L0) → lightweight signal triage (L1) → hindsight relabeling (L2) → model
iteration (L3) — where each upper layer consumes the output of the layer below
without re-doing its work.

L1 triage should be implemented predominantly with non-semantic, rule-based or
small-surrogate detectors over interaction / execution / environment surfaces,
not with per-trajectory LLM-as-Judge calls; the cost ratio between the two
makes LLM-judging economically infeasible at production scale, and the Signals
paper's 82% informativeness at ~1.5× sampling efficiency on τ-bench is a real,
if unreproduced, demonstration that lightweight signals can reach a useful
operating point. The most under-recognized payoff of signal triage is on
*successful* trajectories: roughly two-thirds of "task-completed" traces still
contain learnable hidden friction (policy violations, inefficient tool use),
which is exactly the regime KWeaver needs for ongoing model improvement once
gross failures are rare.

Schema design (L0) is the silent gating constraint: AgentTrace-style operational
+ cognitive + contextual surfaces are necessary but not sufficient — a complete
KWeaver schema must additionally capture user-interaction discourse and
system-resource state, or L1 Interaction and Environment signals cannot be
computed at all. Observability cost should be controlled by topology-aware
sentinel sampling that uses L1 signals as upgrade triggers, not by uniform
downsampling.

The dominant industry alternative — Agent-as-a-Judge / LLM-as-Judge — is best
treated as a *complementary* deep-diagnostic tool that runs on the small subset
already triaged by L1, never as the front-line filter. Signal-based triage and
LLM-judging are not competing on the same metric: triage measures sampling
informativeness, judging measures evaluation accuracy. Conflating the two is a
recurring rhetorical error in the literature that we will explicitly resist.

This thesis is falsifiable on at least three claims: (a) that lightweight
signals reach >70% informativeness on a realistic non-τ-bench corpus; (b) that
hindsight relabeling of L1-triaged traces produces measurable downstream win
rates over random-sampled preference data; (c) that signal-driven sentinel
sampling reduces observability cost by a meaningful factor (e.g. >5×) without
degrading downstream training data quality. Counter-evidence on any of these
should force a reframing.

**Methodological constraint on the judgment layer.** Judgment chains should
drive determinism as deep into the tree as possible. When judgment rules are
enumerable (e.g. `BKN.PreCondition` → required RO set), mechanistic oracles
strictly dominate LLM-judge — the reason is not cost, it is that LLM variance
is itself harmful to production-grade judgment, regardless of mean accuracy.
Where judgment unavoidably depends on the LLM (genuinely irreducible semantic
predicates such as memory hallucination or reflection misassessment), the
LLM must be wrapped in a structured decision tree of narrow yes/no predicates
with explicit aggregation, thresholds, and abstain conditions; *free-form*
"is this trajectory good?" prompting is rejected as a judgment form, even
when it benchmarks well. This is the same principle as L1's lightweight-
signal discipline, applied one layer deeper — not a new principle.

## Taste

- **Favor lightweight, deployable signals over LLM-judge approaches** for
  front-line filtering. Heavyweight methods are admitted only as upper bounds
  or as deep-diagnostic tools downstream of triage.
- **Prefer mechanistic explanations over correlation studies.** A paper that
  defines a detector, threshold, or schema field beats one that reports
  end-to-end metrics without exposing the mechanism.
- **Reproducibility-of-method matters more than reproducibility-of-numbers.**
  Open thresholds, phrase lists, and aggregation formulas are worth more than
  benchmark deltas — the benchmarks won't transfer, the mechanisms will.
- **Production-grade detail earns priority.** Cost, latency, drift,
  versioning, multilingual robustness, and schema evolution are first-class
  concerns, not appendix material.
- **Trajectory-level and pipeline-level work over single-turn work.** Most
  agent failures are visible only across multiple turns or tool calls; methods
  that only inspect single steps are pre-emptively suspect.
- **Successful-trajectory hindsight is a feature, not a curiosity.** Methods
  that only mine failures undervalue the largest learnable surface in mature
  systems.

## Anti-patterns

- **Benchmark-only papers without an underlying method or detector design.**
- **Pure LLM-as-Judge papers that ignore per-trajectory judging cost** and
  thus implicitly assume infinite eval budget.
- **Survey or position papers that don't introduce a new framing or
  implementable taxonomy.** A renamed taxonomy is not a contribution.
- **Token-level or single-turn evaluation work** dressed up as agent
  evaluation, with no trajectory-level extension.
- **"Black-box composite scores"** that aggregate signals without disclosing
  weights, thresholds, or ablations — non-actionable for deployment.
- **Static-rule papers** that don't acknowledge drift, multilingual reality,
  or detector versioning — production blind spots.
- **Papers that conflate sampling informativeness with judgment accuracy** —
  see thesis section above; this is a category error we will not absorb.
- **Auto-feedback / auto-fix loops that do not audit their own reliability.**
  Reporting only end-to-end success deltas without measuring fix-prediction
  or regression-prediction accuracy of the *feedback itself*. AHE [12] reports
  both (fix ≈ 5× random, regression ≈ 2× random); AgentDebug [13] reports
  neither. End-to-end gains in this regime cannot distinguish "the feedback
  was right" from "the base model recovered despite noisy feedback" — and
  the latter is far more common than authors imply.

## Examples

- Good inclusion (canonical): `notes/01_signals_trajectory_triage.md` —
  defines a 2×2 taxonomy and 7 signal classes, reports informativeness on a
  real testbed, and is honest about its non-released detectors.
- Good inclusion (complementary): `notes/02_agenther_hindsight_relabeling.md`
  — explicit failure-mode taxonomy and relabeling pipeline; the natural L2
  consumer of L1 triage.
- Good inclusion (infrastructure): `notes/04_agenttrace_structured_logging.md`
  and `notes/05_breaking_observability_tax.md` — schema and cost-control
  primitives that L1 cannot live without.
- Borderline / contrast cases: `notes/07_agent_as_a_judge.md` and
  `notes/08_tide_trace_diagnostics.md` — kept as the deep-diagnostic /
  evaluation contrast group, not as front-line triage.
- Lifecycle-adjacent (kept for contrast):
  `notes/03_tsr_trajectory_search_rollouts.md` — training-time rollout
  selection; valuable as the training-side mirror of deployment-time triage,
  but not directly substitutable.
