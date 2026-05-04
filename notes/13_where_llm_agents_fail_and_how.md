# 论文阅读笔记：《Where LLM Agents Fail and How They can Learn From Failures》

> **Created:** 2026-05-04
> **状态：** ✅ 已深读
> **arXiv:** [2509.25370](https://arxiv.org/abs/2509.25370)（v1, 2025-09-29）
> **作者:** Kunlun Zhu, Zijia Liu, Bingxuan Li, Muxin Tian, Yingxuan Yang, Jiaxun Zhang, Pengrui Han 等（UIUC / Stanford / AMD / OpenManus / Toronto / Likelihood Lab）
> **代码/数据:** https://github.com/ulab-uiuc/AgentDebug（仓库声明 "will be available"，本版本未确认是否已 release）
> **分类轴：** layer = cross_evaluation 为主、并伴随 L1（error 分诊）+ 部分 L3（iterative 修复 → task success）；signal_kind = cognitive（让 LLM 在轨迹上做模块级错误归因，再以 counterfactual / LLM 判定锁定 root cause）；cost_profile = llm_judge（detector 与 critical-error identifier 都是单 LLM 的结构化 prompt 调用，论文用 GPT-4.1 temperature=0；下游 re-rollout 用 GPT-4o-mini / Qwen3-8B / Qwen3-Next-80B）；lifecycle = deployment_time（轨迹完成后回放分析 + 再 rollout）兼有 online（"max attempts"循环 ≤5 次回挂式重试）；deployability = method_only（taxonomy 与 benchmark 标注公开，但 detector / re-rollout 完整 pipeline 取决于代码 release，且强依赖闭源模型 prompt 工程）。
> **角色定位：** 这是一篇**Agent-as-a-Judge 路线在错误归因任务上的具体实例**——以 17 类细粒度错误类型构建 taxonomy（AgentErrorTaxonomy）、人工标注 200 条失败轨迹（AgentErrorBench）、用 GPT-4.1 单 agent 三阶段定位 root-cause critical error 并生成 corrective feedback 驱动 re-rollout 的修复闭环（AgentDebug）。它对 KWeaver 主线的价值是双向的：(a) Taxonomy 本身可作为 KWeaver L1 信号本体的对照——它给出了五模块（Memory / Reflection / Planning / Action / System）+ 17 子类的"失败模式词表"，与 AgentHER 的 6 类、Signals 的 3×3、AHE 的 7 组件分解构成同一问题的不同切法；(b) 方法上是 thesis 明确反对的"front-line LLM-judge"路线：detector 与 critical-error finder 都是 LLM 调用，cost 没拆但量级上单条轨迹要 ≥4×LLM-pass（4 模块 × 每步），ALFWorld 任务步数中位 ~10–15，单条轨迹仅 detection 阶段就 50–100 次 LLM 调用——恰是 thesis 反对的部署模式。值得作为反例完整记录、并在 critical-error vs surface-error 这个**论文真正有方法论贡献的点**上做严格批判性吸收。

---

## Claims

1. LLM agent 失败的核心瓶颈不是单步错误率，而是**误差传播（error propagation）**——单一根因失败会沿轨迹级联放大，最终导致任务失败；这是论文从 500+ 失败轨迹的人工分析中得出的"central insight" [13: §1, §2.1]。
2. Agent 失败可被分解到 5 个模块、17 类细粒度错误类型上：Memory（hallucination / over-simplification / retrieval failure）、Reflection（progress misassessment / outcome misinterpretation / causal misattribution / hallucination）、Planning（constraint ignorance / impossible action / inefficient planning / planning–action disconnect）、Action（format error / parameter error / misalignment / invalid action）、System（step limit / tool execution / LLM limit / environment error）[13: §2.1, Appendix A.2 Table 2]。
3. AgentErrorBench 是首个系统标注的 agent 失败轨迹数据集：200 条轨迹（100 ALFWorld / 50 WebShop / 50 GAIA），10 名研究生标注者按 taxonomy 标到 decision-step 级，每条轨迹同时标记**最小 root-cause 失败集合**而不是穷尽地标记所有 surface 错误；inter-annotator agreement Cohen's κ = 0.55，论文称之为 "substantial agreement" [13: §2.2]。
4. 失败在轨迹中的位置呈中段聚集：多数失败发生在 step 6–15 之间（mid-trajectory），并且 Memory + Reflection 是最主要的根因来源（占 ~38% 的根因模块标记），Planning 类错误次之，Action 与 System 较少 [13: §2.2 Figure 3 / Appendix Figures 11–13]。
5. AgentDebug 的三阶段流程在 critical-error 检测任务上系统性优于直接 prompt、暴力枚举、二分搜索三类基线；在 ALFWorld + WebShop + GAIA 平均上：Step accuracy 45.0%（vs 最强基线 28.0%）、Step+Module 31.3%（vs 10.0%）、All-Correct 24.3%（vs 0.3%），即 +24 pp All-Correct 与 +17 pp Step [13: §4.1 Table 1]。
6. 检测准确率提升直接转化为下游任务成功率：在 ALFWorld 上，AgentDebug 把 GPT-4o-mini 从 21% 提到 55%、Qwen3-8B 从 48% 提到 74%、Qwen3-Next-80B 从 60% 提到 84%；跨三个 benchmark 上限相对增益达 26%（论文文本反复引用的总结性数字）[13: §4.2 Figure 5–6]。
7. 在与同等 token 预算的强基线对比下，AgentDebug 系统性优于 Self-Refine、Vanilla Debugger、Tree-of-Thought、Best-of-N——论文论点是"更细粒度的 root-cause 定位"比"更多次 unguided 重试"或"更宽的搜索空间"更高效 [13: §4.2]。
8. 模型规模消融显示：以 GPT-4.1 作 AgentDebug detector 的 Step / Step+Module / All-Correct 分别为 42.0% / 44.0% / 32.0%，远高于 Llama-3.3-70B（16.0% / 16.0% / 6.0%）、GPT-4o-mini（14.0% / 10.0% / 4.0%）、Qwen3-Next-80B（4.0% / 14.0% / 2.0%），即 detector 端的 root-cause 定位**强烈依赖最强基模** [13: §5.1 Figure 7b]。
9. Rollout strategy 消融：在 ALFWorld zero-shot 下，论文自定义的 Modular rollout 得 0.38，胜过 Memory+ReAct 0.34 / Reflection 0.32 / ReAct 0.26 / Act-only 0.10 [13: §5.1 Figure 7c]。
10. 最大重试次数（max attempts）消融：从 1 到 5 次重 rollout，三个基模上累积成功率单调上升；GPT-4o-mini 受益最大（21 → 55，+34 pp），Qwen3-Next-80B 受益最小（60 → 84，+24 pp），论文论点是"weak base model + strong debugger"是更有性价比的部署形态 [13: §5.1 Figure 7a]。
11. 误差传播在轨迹热力图上呈"早期错误进入即不可逆"模式：darker red 单元格说明初始错误代往后稳步累积；Memory / Reflection 错误的 cascade 长度最长，Action / System 错误更倾向于"立即终止"而非"持续放大" [13: §5.2 Figure 8]。
12. 论文提出的方法论建议：(a) 早期检测与修正是关键，cascade 一旦开始就难逆转；(b) 强化 memory retrieval 与 reflection（外部 memory / progress tracking / verification prompts）可显著降低 propagation 风险 [13: §5.2]。

## Assumptions

- **失败轨迹的"根因"是可良定义且唯一最小集合的。** §2.2 标注 protocol 假设每条失败轨迹存在一个最小 root-cause 集合解释下游 cascade；论文用 "minimal set" 与 "earliest critical step" 两个判定共同界定它，但**最小性与最早性可能冲突**——例如一个早期 Memory 错误未触发立刻失败、却在后续被一个 Plan 错误"放大"，按"最早"应记 Memory，按"最小集合"也许只需标 Plan。论文未公开判优规则。
- **GPT-4.1 与 GPT-4o 系列等模型在任务完成端与 critical-error 检测端的能力对称。** Detector 全用 GPT-4.1，code agent 用 GPT-4o-mini / Qwen 系列；如果 detector 实际比 code agent 强一个量级，那么"AgentDebug 提升"既可归因于 detector 准确性也可归因于"用强模型监督弱模型"——论文未做"detector 与 code agent 同模型同规模"的对照。
- **Algorithm 1 与 §3.2 的 Stage 2 描述为同一机制。** Algorithm 1 注释明确写 "Critical Error Detection via LLM (no rollout/counterfactuals)"，而 §3.2 文字说 "we perform counterfactual testing step by step: at each point, we substitute a corrected action and test whether the rollout would succeed"——这是同篇论文同节内容的内部不一致；附录 A.5 的 Detector Prompt 与 AgentDebug Prompt 也表明实际实现是 LLM-only，没有 actual counterfactual rollout。论文措辞上沿用了 "counterfactual" 一词，但语义上变成了"在 prompt 中让 LLM 想象 corrected action"。
- **Cohen's κ = 0.55 等价于 "substantial" 一致性。** 标准 Landis & Koch 解释中 0.41–0.60 = moderate、0.61–0.80 才是 substantial；论文把 0.55 写为 "substantial agreement" 在术语上偏乐观一档。
- **由 LLM 生成的 corrective feedback 在 re-rollout 阶段不会泄漏 ground-truth 任务信息。** Detector 输出 corrective_guidance 时已读完整失败轨迹（含 task description / observation / 观测反馈），feedback 文本会回灌给同 task 的下一轮 rollout——论文未对 feedback 文本是否泄露 oracle 信号做扫描。
- **Step+Module+ErrorType 的 "All-Correct" 是有意义的 strict metric。** 但 17 类 error type 中部分边界模糊（论文自承在 pilot 中需要"多轮校准"才能区分 retrieval failure vs constraint ignorance）；strict metric 依赖标注一致性的高地基，而 κ=0.55 已经表明这个地基不稳。
- **Best-of-N / ToT 与 AgentDebug 在 token 预算上严格匹配。** §4.2 描述 "max number of attempts of all baselines is matched to AgentDebug by total token usage"；但 AgentDebug 在 detector / debug / re-rollout 三段都消耗 LLM 调用，"匹配"是按总和匹配还是按单段——论文未拆，无法核验是否实际公平。
- **轨迹的 step 索引是稳定的标注锚点。** Step 范围如 "Step 6–10" 在 Figure 3 中作为聚合桶出现；不同长度的轨迹被归一到相同桶，但论文同时把 step 当作 critical-error 的 exact 定位坐标，桶化与精确定位的张力未讨论。

## Method

**问题框架。** 给定一条已完成且失败的轨迹 τ = {(s_t, a_t)}_{t=1..T}（每步含 state s_t 与 agent action a_t）、错误 taxonomy E_AET、critical-error 判定准则 C_crit、最大 iteration 数 I=5。AgentDebug 输出修正后的轨迹 τ* 或 Failure。

**Stage 1 — Fine-grained Analysis（细粒度模块级错误标注）。** 对每一步 t，对四个核心模块 m ∈ {memory, planning, reflection, action}，调用一个统一的 detector LLM（GPT-4.1, T=0）做单 turn 判定：是否存在错误、属于 17 类中的哪一类、提供 evidence quote 与 reasoning。Prompt 见附录 A.5 "Detector Prompt"，输出严格 JSON schema {error_detected, error_type, evidence, reasoning}。对每步而言，4 模块均有对应输出，构成 step×module 的 error profile e^m_t。

特殊规则：评估顺序为 Memory → Reflection → Planning → Action（同一 step 内"上层模块的输出可作为下层模块的输入"）；Step 1 仅评估 Planning + Action（无历史可调），Memory / Reflection 在 step 1 自动豁免；System 错误（step_limit / tool_execution_error 等）作为独立类别旁路标注。

**Stage 2 — Critical Error Detection（最早 root-cause 定位）。** 把全轨迹 τ、step×module 错误集 {e^m_t}、taxonomy E_AET、判定准则 C_crit 一并作为 context 喂给一个 critical-error LLM（同样 GPT-4.1），让它输出 (T*, M*, Z*, ϕ⁽⁰⁾)：
- T* = 关键 step 集合（论文实际只取 t* = min(T*) 作为最早 critical step）；
- M* = 该 step 的关键模块；
- Z* = 该 step / 模块的关键 error type；
- ϕ⁽⁰⁾ = 给下游 re-rollout 的 actionable feedback 文本（含 root_cause / evidence / correction_guidance / cascading_effects 字段）。

Algorithm 1 注释明确写 "via LLM (no rollout/counterfactuals)"——即没有真的回放 trajectory 并替换动作再跑环境，所谓 counterfactual 是在 prompt 中让 LLM 文字推演"如果这步换动作会怎样"。

Stage 2 的 prompt（附录 A.5 "AgentDebug Prompt"）显式要求模型遵守 7 条 holistic 指引：考虑全轨迹、找最早决策点、early exploration 步（1–3）默认不算 critical 除非有清晰错误、定义 critical = "纠正它就能扭转整个轨迹"、step 1 没有 memory/reflection 不能标为该类、把 System / Others 也作为可能的 critical category（如 step_limit 本身可能是真因）。

**Stage 3 — Iterative Debugging with Targeted Feedback（带反馈的重 rollout 循环）。** 给定 τ⁽⁰⁾ = τ、t*、ϕ⁽⁰⁾，循环 k=1..I：
1. τ⁽ᵏ⁾ = ReRollout(τ⁽ᵏ⁻¹⁾, t*, ϕ⁽ᵏ⁻¹⁾)——从 t* 起重新执行 trajectory（不是从头），把 ϕ⁽ᵏ⁻¹⁾ 作为额外指令注入 code agent 的 prompt；
2. 若 Eval(τ⁽ᵏ⁾) = 1，返回成功的 τ⁽ᵏ⁾；
3. 否则用 UpdateFeedback(τ⁽ᵏ⁾, ϕ⁽ᵏ⁻¹⁾) 让 detector 读新的失败轨迹再产出一份更细化的 feedback ϕ⁽ᵏ⁾，进入下一轮。

I 上限固定 5；论文 §4.2 称 "concentrating computational effort where it is most impactful"——核心论点是"从 critical step 重 rollout 比从 0 重 rollout 更省 token"。

**实现细节。** Detector / critical-error finder：GPT-4.1, T=0；code agent 基模：GPT-4o-mini / Qwen3-8B / Qwen3-Next-80B；max attempts = 5；baselines 按 total token usage 与 AgentDebug 匹配。Environment rollout prompts 见附录 A.6（ALFWorld / WebShop / GAIA 各自有 _NO_HIS、_LAST_STEP、normal 三个版本，强制要求 agent 在 `<memory>`、`<reflection>`、`<plan>`、`<action>` tag 中输出——这种"强制四模块外显化"是 detector 能逐模块判错的工程前提）。

**总结。** 整个 pipeline 是 "LLM-judge 套娃 LLM-judge"：4 个模块 × T 步的细粒度 detector 调用 + 1 次 critical-error LLM 调用 + 最多 5 次 re-rollout（每次都可能引发再一次 detector 调用作 UpdateFeedback）；对单条 ALFWorld 轨迹（中位 step 数 ~10–15）就要 40–60 次细粒度 detector LLM 调用 + 1 次 critical finder + 最多 5 次重 rollout。但论文从未把 cost 拆开报告。

## Eval

- **Critical-error detection 主结果（§4.1 Table 1，AgentErrorBench / 200 任务）。**
  - 三类 baseline + AgentDebug，metric = Step (S) / Step+Module (S+M) / All-Correct (ALL)。
  - 平均：Direct Prompting 28.0/10.0/0.3；Brute Force 12.0/4.3/0.0；Binary Search 18.7/8.0/0.3；**AgentDebug 45.0/31.3/24.3**。
  - 子集：ALFWorld（35/28/21）、WebShop（42/22/14）、GAIA（58/44/38）；GAIA 上 Step 几乎翻倍（30 → 58）、All-Correct 三倍（12 → 38）。
- **下游任务成功率（§4.2 Figure 5，ALFWorld 100 任务，三个基模在 attempt 1..5 的累积成功率）。**
  - GPT-4o-mini：21 → 30 (+9) → 41 (+11) → 48 (+7) → 55 (+7)；
  - Qwen3-8B：48 → 58 (+10) → 63 (+5) → 70 (+7) → 74 (+4)；
  - Qwen3-Next-80B：60 → 76 (+16) → 80 (+4) → 80 (+0) → 84 (+4)。
  - "26% 相对增益"指的是其中某个 cell（论文反复引用但不指名）；从 GPT-4o-mini 21 → 55 的绝对增益 +34 pp 折成相对增益约 162%，从 Qwen3-Next-80B 60 → 84 折成 40%——故 "26%" 是论文跨 ALFWorld / GAIA / WebShop 的某种平均，未拆给读者。
- **Cross-benchmark generality（§4.2 Figure 6，与 Self-Refine / Best-of-N 的对比）。** AgentDebug 在 ALFWorld 增益最大（论文文本：up to 26%），GAIA / WebShop 上"strong average"——具体数字 Figure 6 中给出但 paper text 未在这一节展开数据点；这一节的论证强度比 §4.2 弱。
- **Detector base-model ablation（§5.1 Figure 7b）。** GPT-4.1 42/44/32（Step / Error / S+M / ALL）显著领先；Llama-3.3-70B 16/16/6/2、GPT-4o-mini 14/10/4/2、Qwen3-Next-80B 4/14/2/2——意味着把 detector 从 GPT-4.1 换为更小模型，All-Correct 直接掉到接近 0；这是对 detector 端 LLM-judge 路线的最强模型依赖证据。
- **Rollout strategy ablation（§5.1 Figure 7c，ALFWorld zero-shot）。** Modular 0.38 > Memory+ReAct 0.34 > Reflection 0.32 > ReAct 0.26 > Act Only 0.10——意味着"四模块外显化"本身就比 ReAct/Reflection 一体化模式有约 +12 pp 的成功率优势，且这部分增益**不需要任何 detector 介入**就能拿到。
- **Max-attempts ablation（§5.1 Figure 7a）。** 单调递增、收益递减；在 5 attempts 处三个基模都接近自己的上限。
- **κ = 0.55 的独立验证。** 论文报告 inter-annotator agreement 为 Cohen's κ = 0.55；按标准 Landis & Koch 解释为"moderate"而非论文叙事中的 "substantial"；这是 AgentErrorBench 数据质量的上界——detector 的 strict All-Correct 不可能稳定超过标注者间一致性的天花板。
- **缺失的对照。** 没有"detector 替换成 deterministic / rule-based 错误检测器"的对照（如 Near-Miss 路线的 guard code），也就无法量化 LLM-judge 相对于 cheaper alternatives 的边际贡献；没有"feedback 替换成 random hint / oracle hint"的对照（无法判断 corrective_guidance 的语义价值 vs 仅仅"从 critical step 重启"的结构价值）；没有"全模块 detector 调用 vs 只调用关键模块"的成本-收益曲线（论文 setup 强制 4 模块全跑）；没有"feedback leakage 检查"——detector 已读完整轨迹（含 task description），其 corrective_guidance 是否泄漏 oracle 信号未做扫描。
- **内部数字不一致。** §4.2 Findings 段写 "50.0% step accuracy and 42.5% all-correct accuracy"，但 Table 1 平均行是 45.0% / 24.3%——这两组数字都不能从 Table 1 三个子集（ALFWorld/WebShop/GAIA）的简单平均得到；论文未解释这一段的口径，疑为不同子集 aggregate 或不同 base-model 的报告（与 §5.1 Figure 7b 也不直接吻合）。

## Weaknesses

1. **Front-line LLM-judge 路线，cost 完全不透明。** 单条 ALFWorld 轨迹（~10–15 步）的 detection 阶段需要 4 模块 × 步数 ≈ 40–60 次 GPT-4.1 调用，加 critical finder 1 次、最多 5 次重 rollout 的 detector 再调用，单条轨迹的 LLM 总成本与 baseline Best-of-N 同量级；论文从未拆 detection / critical-finder / re-rollout 三段的 token 占比，"matched by total token usage" 的"匹配"无从核验。在 thesis "lightweight signals beat LLM-judge for front-line filtering"的论证方向上，本文是 thesis 反对的部署模式的具体实例。
2. **Algorithm 1 与 §3.2 的 Stage 2 描述明显冲突。** Algorithm 1 注释明确写 "via LLM (no rollout/counterfactuals)"，§3.2 文字却说 "perform counterfactual testing step by step: at each point, we substitute a corrected action and test whether the rollout would succeed"。两段描述不能同时为真——附录 A.5 的 prompt 可证实际实现是 LLM-only 的"想象式 counterfactual"，并非真的执行替换动作的 rollout。这种术语滑移在评估方法论上是误导性的：真 counterfactual rollout 是有 oracle 反馈的实验，prompt 内的 "想象 counterfactual" 仍是 LLM 自我推断，没有外部判决。
3. **κ = 0.55 被叙事为 "substantial"，但 strict All-Correct 24.3% 的天花板恰恰被这个一致性约束。** Landis & Koch 标准解释 0.41–0.60 = moderate，0.61–0.80 才是 substantial；论文术语越档一档。更关键的是：在标注者本身一致性只有 moderate 的数据上，detector 的 All-Correct（同时命中 step、module、error type）即使训练得再好，理论上限也被 κ 限制——这意味着 24.3% 的 All-Correct 数字在对比"无监督基线 0.0–0.3%"时显得很大，但相对"标注 ground truth 自己的可信度"仍偏低。
4. **AgentErrorBench 单标注者偏置。** §2.2 描述 "Disagreements were adjudicated collectively"，但 §5.1 标注规模与人力（10 名研究生 / 200 条轨迹）下，"calibration → double-annotation on shared subset → adjudication" 流程仅在子集上跑过；其余轨迹由单人标注。结合 17 类细粒度错误类型与 5 模块判定（其中 retrieval failure 与 constraint ignorance 边界明确需要"多轮校准"），单人标的部分一致性可能远低于 0.55。
5. **Detector 端强烈依赖 GPT-4.1。** §5.1 Figure 7b 显示，detector 换成 Llama-3.3-70B / GPT-4o-mini / Qwen3-Next-80B 后 All-Correct 直接从 32% 掉到 2–6%；这意味着方法的有效性几乎完全绑在最强闭源 base 上。论文从未尝试 ensemble / 多次抽样投票等可降低单模型脆弱性的设计——AgentDebug 实际是"用 GPT-4.1 解读轨迹"的细化产品，而非一个可移植的 detection framework。
6. **"Critical = 最早 + 最小"两个判据并存却未给冲突时的解。** §3.2 Stage 2 prompt 同时要求模型找"最早能扭转结果的错误"与"最小 root-cause 集合"。当一个早期 Memory 错误本身不必然导致失败、但被一个后续 Plan 错误放大时，按"最早"标 Memory，按"最小"也许只标 Plan 即可——两者结论不同。论文没给 tie-breaking 协议，也未让标注者对此类案例分布做拆分，于是 Step / S+M / ALL 三个 metric 在这类案例上的"正确答案"本身就有歧义。
7. **Modular rollout 0.38 vs ReAct 0.26 已经吃掉了大半增益；AgentDebug 的"detector + re-rollout"边际贡献比叙事小。** §5.1 Figure 7c 表明：仅仅强制 agent 把 memory / reflection / plan / action 四模块外显化（即 ALFWorld_TEMPLATE 模板），相对 ReAct 的 0.26 → 0.38 约 +12 pp 成功率，已经大幅缩小了"什么是 agent 失败"的歧义空间。这份增益来自 prompt 工程而非 critical-error 检测；剔除它之后，detector + re-rollout 的真正净增益要小得多——但论文把"Modular rollout 优于 ReAct" 与"AgentDebug 优于 Self-Refine / Best-of-N"放在两个不同节，读者难以做到 apples-to-apples 对比。
8. **"全部失败 25% 是显式策略违规"这一脚注引用未对齐。** §1 footnote 给出"约 25% 是显式 policy violation"——这个数字与本论文 AgentErrorBench 的 5 模块标注完全不同源，似乎来自外部基准引用；上下文混淆了"AgentErrorBench 的 root-cause 分布"（不在论文主表里）与"τ²-verified 上的 ToolGuard violation 比率"。读者会误读为"taxonomy 中 25% 的根因属于 explicit violation"，但实际两者不可比。
9. **Re-rollout feedback 可能泄漏 oracle 信号。** Detector 读完整失败轨迹（含 task description、observation、admissible_actions、reward 流），写出 corrective_guidance 之后回灌给 code agent。在 ALFWorld 这类有清晰目标分解的 benchmark 上，detector 完全有可能写出"go to drawer 4 and look for soapbar"级别的具体提示——这等价于把 oracle 路径替代灌注。论文没做"feedback 是否仅含元规则（如 'expand search to countertops'）" vs "是否含具体动作（如 'take saltshaker 2 from countertop 2'）"的扫描；附录 A.3 的 example output（"expand to countertops/tables"）是论文挑选的友好样例，不构成系统性证据。
10. **Brute-Force baseline 的 12.0% Step accuracy 反常低。** 一个枚举地从 t=1 起逐步替换动作并实际跑 rollout 的方法理论上是 critical-error 的"上界"——只要 simulator 是 deterministic 且 corrected action 集合可枚举，Brute Force 应当接近 100% Step 命中。论文报告它只有 12.0% / 4.3% / 0.0%，远低于 Direct Prompting 的 28.0% / 10.0% / 0.3%——这意味着 Brute Force 的实现里"corrected action"也是 LLM 想象产生的，并非真 oracle 替换；如果 Brute Force 本身就是"LLM 提议替换动作 → 仍由 LLM 判定是否成功"，那么它就**不是真的 brute force**，整个 baseline 设置在术语上误导。
11. **Cross-benchmark Figure 6 数字不全。** §4.2 主文叙述 "up to 26% on ALFWorld and strong average performance on GAIA and WebShop"——把最大增益挑出来叙事，GAIA 与 WebShop 的具体增益数字 paper text 未给出（仅 Figure 6 中可读）。"strong average" 是修辞，不是 evidence。
12. **§4.2 Findings 段 "50.0% / 42.5%" 与 Table 1 平均 "45.0% / 24.3%" 不一致。** 论文同节出现两组数字而无解释；任一组都可能是 typo，但更可能是两段对应不同 base-model 配置（一处用 GPT-4.1 detector + GPT-4.1 code agent；一处用 GPT-4.1 detector + GPT-4o-mini code agent）。论文不澄清，读者无法核对哪组数字适用于"AgentDebug 主张"。
13. **Memory + Reflection 占根因 ~38% 的论点对 KWeaver 的可迁移性受限。** §2.2 Figure 3 的根因分布是在 ALFWorld / WebShop / GAIA 三个文本游戏 / 网购 / 通用助手 benchmark 上得到的；它们都强依赖于 agent 的 long-context 记忆与状态跟踪。在 KWeaver 关心的 DPH 编排式调用（弱用户对话、强 tool / API 链）场景下，Memory 错误的频度可能远低，Action / System 错误（API 不一致、参数 mismatch、tool fail）的占比应远高。论文未做跨 benchmark family 的根因分布对比，把"Memory + Reflection 主导"的结论当作普适性发现是过度推广。
14. **没有任何 detector 调用次数控制变量的对照。** 一个直接的对照是：把 detector 只跑在 step t* 附近 ±2 步而非全步全模块——若性能只跌一两个 pp，就证明大部分 detector 调用是浪费。论文 setup 默认 detector 跑全图，使得"detection accuracy → success rate"的因果链中混进了大量冗余调用的 cost；任何想搬到生产的实践者必须自行设计稀疏 detection。

## Relations

- **builds-on `02_agenther_hindsight_relabeling` [med]**：两篇都把"失败模式 taxonomy + 失败轨迹结构化"作为 agent 改进的入口；AgentHER 给 6 类（Incomplete / Constraint_Violation / Wrong_Result / Tool_Error / Hallucination / Off_Topic），AgentDebug 给 5 模块 × 17 子类。AgentHER 的 Constraint_Violation 与 AgentDebug 的 Planning.Constraint_Ignorance 是同一现象的不同切片（前者更靠 outcome / 任务描述对比，后者更靠 step×module 模块归因）；AgentHER 的 Tool_Error 对应 AgentDebug 的 Action.Format_Error + System.Tool_Execution_Error。但两者下游消费方完全不同：AgentHER 用 taxonomy 做训练数据筛选与 DPO 偏好对构造（offline），AgentDebug 用 taxonomy 做 inference-time 重 rollout 反馈（online）。在 thesis "successful trajectory hindsight is a feature" 视角下，AgentDebug 的 critical-error 标注方法可以反过来供给 AgentHER 路线作为更细粒度的 stage-1 detector——这是一个尚未被任一论文写出的组合点。
- **competes-with `07_agent_as_a_judge` [high]**：AgentDebug 是 Agent-as-a-Judge 在 root-cause 任务上的具体实现：detector + critical finder 都是 LLM 单 agent 调用；与 AaaJ 的差别仅在于 (a) 输出粒度更细（step×module×error_type 而非 single judgment）；(b) 不输出"对错评分"而输出"actionable feedback"。Cost 与可解释性问题完全继承自 AaaJ 范式：detector 严重依赖 GPT-4.1，Llama-3.3-70B 上 All-Correct 直接掉到 6%；这是 AaaJ "judging accuracy varies across base models" 论点在 root-cause 任务上的再次应证。在 thesis "把 LLM-judging 留给 triaged 后小子集"的判断方向上，AgentDebug 把 LLM-judging 用在了**全部失败轨迹的全部步全部模块**——thesis 明确反对的部署模式。
- **competes-with `08_tide_trace_diagnostics` [high]**：TIDE 是 deployment-time post-hoc trajectory 诊断且交付给人类工程师，AgentDebug 是 deployment-time post-hoc trajectory 诊断 + iterative re-rollout 闭环（直接交付 agent 自身）。两者方法论同源（"用 LLM 解读轨迹找根因"），区别在 downstream 消费方：TIDE 给人，AgentDebug 给 agent 自己。AgentDebug 在 TIDE 的基础上再多一步——把诊断结果**自动反馈进新的 rollout**——这一步是 thesis 中"L1 → L2 自动化反馈闭环"的实例化。但 AHE [12: §4.4.2] 已显示 evolve agent 的 fix-prediction 仅 5× random、regression-prediction 仅 2× random；同一类"自动反馈闭环"的可靠性问题 AgentDebug 完全没做对应度量。
- **contradicts `11_near_miss_latent_policy_failure_detection` [high]**：Near-Miss 主张"判定应当机制化、可审、不依赖 LLM 推理"——用 ToolGuard 自动生成的 Python guard code + 结构化历史搜索代替 LLM-judge；AgentDebug 走完全相反路线——detector / critical-finder / feedback writer 全部为 LLM 调用，且强制依赖 GPT-4.1 这一闭源最强基模。两篇论文对"trajectory-level 自动化判定该不该用 LLM"给出了对立答案。在 thesis "mechanistic explanations over correlation studies" / "reproducibility-of-method matters more than reproducibility-of-numbers" 的判断方向上，Near-Miss 路线明显受偏好；AgentDebug 在 GPT-4.1 上达到 All-Correct 24.3%，但 GPT-4o-mini 上仅 4.0%——同一方法在不同 base 上 6× 差距，机制可移植性几乎为零。这条 contradiction 应进入 contradictions.md（与 `11 ↔ 12` 的对立同源）。
- **orthogonal `01_signals_trajectory_triage` [med]**：Signals 在 deployment-time 做 L1 triage 分诊（哪些轨迹值得 LLM-judge / 人审），AgentDebug 在 deployment-time 做 root-cause 修复（已经判定失败的轨迹要怎么修）。两者是同一栈不同层：Signals → AgentDebug 在工程上可串联（Signals 只把高 informativeness 的失败轨迹送进 AgentDebug，从而把 detector cost 摊薄到一个小子集上）。但两者范式对立（rule-based vs LLM-judge），在 thesis 立场上 Signals 是 thesis 标准实例，AgentDebug 是 thesis 反例。
- **builds-on `04_agenttrace_structured_logging` [med]**：AgentDebug 的 Stage 1 之所以能逐 step×模块 判定，**前提**是 trajectory 已经被分解为 (s_t, a_t) 的 step 序列且每步 a_t 已显式拆为 memory / reflection / plan / action 四个 tag 的输出（见附录 A.6 三个 environment 的 rollout prompts）。这等价于 AgentTrace Operational + Cognitive surface 在 prompt 层级的强制实现——agent 必须在 `<memory>` `<reflection>` `<plan>` `<action>` tag 内输出，detector 才有结构化锚点。在 thesis "L0 schema 是 silent gating constraint" 的论证里，AgentDebug 提供了一个新的极端例证：当 agent 的 cognitive surface 没有显式 tag 化（即 ReAct 一体输出）时，rollout strategy ablation（Figure 7c）显示成功率从 0.38 跌到 0.26——schema 缺失直接拿走 31% 的相对增益。
- **orthogonal `03_tsr_trajectory_search_rollouts` [low]**：TSR 是 training-time rollout selection（哪条 rollout 进 RL 训练数据），AgentDebug 是 deployment-time root-cause 定位 + re-rollout。两者都涉及 "rollout 选取"，但 TSR 选数据进训练，AgentDebug 选 step 做修复。对 thesis 的相关性低于 AgentDebug 与 02 / 07 / 08 / 11 的关系。
- **orthogonal `12_agentic_harness_engineering` [med]**：AHE 是 training-loop-time 自演化 coding-agent harness，AgentDebug 是 inference-time root-cause 修复。两者都把"失败 trace → 用 LLM 解读 → 输出修复指令"做成自动化闭环，可视为"自动化失败修复"的两个时间尺度变体（AHE 改 harness 文件持久化、AgentDebug 改单条轨迹的 re-rollout 临时化）。**关键差别**：AHE §4.4.2 显式量化了 evolve agent 的 fix / regression 预测准确率（fix 5× random、regression 2× random）作为对自身可靠性的 self-audit；AgentDebug 完全没做这种 self-audit——它的 corrective_guidance 是否真的 actionable、re-rollout 失败时是因为 feedback 错了还是因为 base model 弱了，论文从未拆解。这意味着 AgentDebug 的 "+26% relative" 包含了大量"代价昂贵的盲试"成分，而 AHE 路线至少有 manifest contract 做 falsifiable 边界。两篇论文方法论上同向、自我审查严格度上 AHE 显著更好——这是 KWeaver 在做 inference-time auto-fix 时应当借鉴 AHE 的 manifest 模式而不是 AgentDebug 的 free-form feedback。
- **orthogonal `06_agentseer_agentic_vulnerabilities` [low]**：AgentSeer 在 trajectory 层做 action-component graph 异常检测，AgentDebug 在 trajectory 层做模块级错误归因。前者抓"图结构异常"（拓扑），后者抓"语义错误类型"（标签）。两者可叠加：先用 AgentSeer 找"哪段轨迹结构异常"再用 AgentDebug 标"该段属于哪类 cognitive 错误"——但 AgentSeer 走 cheap / non-LLM 路线，AgentDebug 走 expensive / LLM 路线，叠加后总成本被后者主导。
- **orthogonal `05_breaking_observability_tax` [low]**：Observability tax 论文关心 production trace 的采样与压缩成本，AgentDebug 完全不关心——它假设全量失败轨迹已经可用、且每条都值得跑 ~50 次 detector LLM 调用。两者面对的 trace 量级问题不同（前者 10⁶ traces × 1× cost / per-trace；后者 10² traces × 50× cost / per-trace），不在同一成本结构。但 thesis 视角下，AgentDebug 的设计前提（"全量 detection 是可行的"）只在 production scale 之外的实验环境成立——这是 AgentDebug 与生产化 KWeaver 路线的根本不兼容点。
- **orthogonal `09_trajectory_guard_a_lightweight_sequence_aware` [low]**：TrajectoryGuard 用 Siamese RNN 做 trajectory anomaly 二元判定（F1 ~0.92），AgentDebug 用 LLM 做 root-cause 多类标注 + actionable feedback。两条路线代表"learned small surrogate vs LLM-as-judge in detection role"的极化对照。TrajectoryGuard 输出 0/1 告警值给下游处理，AgentDebug 输出可读 root cause + correction 指令给 agent 自己消费——任务设定不同但 cost / reliability 维度构成实质对比。
- **orthogonal `10_policy_invisible_violations_in_llm_based` [low]**：Sentinel 在 deployment-time 做 declarative invariant 在线拦截，AgentDebug 在 deployment-time 做 post-hoc root-cause 修复。两者目标不同（拦截 vs 修复），但都属"非训练时" agent 治理；Sentinel 完全不依赖 LLM 推理（用 KG 反事实模拟），AgentDebug 完全依赖 LLM——又是 thesis 范式光谱上的两极。
