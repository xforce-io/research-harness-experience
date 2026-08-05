---
zone: active
tags: []
pin: false
score: 0.6996328029375765
dwell: 1
---
# 论文阅读笔记：《When Agents go Astray: Course-Correcting SWE Agents with PRMs》

> **Created:** 2026-06-02
> **状态：** ✅ 已深读
> **arXiv:** [2509.02360](https://arxiv.org/abs/2509.02360)（v2, 2025-10-21；NeurIPS 2025 Workshop: Scaling Environments for Agents, SEA）
> **作者:** Shubham Gandhi（CMU，IBM Research 实习期间完成）, Jason Tsay, Jatin Ganhotra, Kiran Kate, Yara Rizk（IBM Research）
> **代码/数据:** 无公开仓库；但全部 PRM 变体 prompt 在附录 A.1（Listing 1–6）逐字给出。
> **分类轴：** layer = cross_evaluation 为主（**inference-time process supervision**，不在 thesis 四层栈内；最接近"不改策略参数的 L3 模型迭代"，但运行在单条轨迹执行中途）；signal_kind = cognitive（PRM 读 thought/action/observation 三元组、按 12 类失误 taxonomy 做推理判定）兼 execution（输入含 tool 调用与 observation）；cost_profile = llm_judge（强变体用 C LAUDE -S ONNET-4 做 PRM，每 n=5 步调一次）；lifecycle = **online**（执行中途、轨迹未结束时介入——这是它与本语料库主流 deployment_time post-hoc 工作的关键差异点）；deployability = method_only（prompt 全公开，可复现到 prompt 级；无代码、无检测器阈值、强依赖闭源 PRM）。
> **角色定位：** 这是一篇**把 TRAIL / MAST 式失误 taxonomy 从"事后诊断"搬到"执行中途实时纠偏"**的工作。它对生产级 agent 系统的价值是双向的：(a) 它给出了一个**新的生命周期位置**——把 LLM-judge 当作 in-flight course-corrector 而非 post-hoc filter——这是 thesis 四层栈里没有显式覆盖的一个面，值得作为"判断层放在哪一时刻"的对照；(b) 方法上它是 thesis 明确反对的"front-line LLM-judge"+"不审计自身反馈可靠性的 auto-fix 闭环"的又一个具体实例，且**比 AgentDebug [13] 更典型**：每 5 步无条件触发一次 LLM critique，几乎每次都判"suboptimal"（7.21/7.24），却从不度量这些 critique 本身的命中率。同时它有一个真正有方法论价值的发现——**taxonomy-guided（结构化 12 类 yes/no 判定）严格优于 unguided free-form critique 也优于 action-prescriptive 硬指令**——这条恰好部分印证 thesis "结构化决策树优于自由式 prompt"的判断层立场，值得严格吸收。

---

## Claims

1. SWE agent 轨迹普遍含有**代价高昂但非即时错误**的低效模式（冗余探索、动作循环、解出后不终止），这些"trajectory-level inefficiencies"会沿长程轨迹累积、浪费 compute、抬高 latency、在完成前耗尽 step 预算 [16: §1]。
2. 现有处理路线（TRAIL [6] / MAST [3]）是 post-mortem 的——把整条轨迹 dump 给 LLM judge 做事后错误分析，无法阻止已经发生的浪费，且部署中通常还需人工分析员在两次运行之间手改 prompt / 启发式 / 控制逻辑 [16: §1, §2.2]。
3. SWE-PRM 是首个把 PRM 用于 SWE agent **实时（inference-time）轨迹级错误预防、检测与纠偏**的工作；PRM 周期性以**有限滑窗**介入、由 taxonomy 锚定、输出可立即施用的自然语言反馈，且**不修改基础策略的架构或参数** [16: §1, §3.2, abstract]。
4. 在 SWE-bench Verified 上，闭源 PRM（C LAUDE -S ONNET-4）把 SWE- AGENT-LM-32B 的 resolution 从 40.0% 提升到 **50.6%（+10.6 pp）**，medium +10.7 pp、hard +4.4 pp，且轨迹更短或持平（37.99 vs 38.64 步）[16: §1, §5.1–5.2, Table 3]。
5. **开源 PRM 几乎全部无效甚至有害**：六个开源 PRM 变体的 resolution 落在 30.0–38.8% 之间，无一超过 40.0% base；最差的 SWE-PRM_S 把 32B 模型从 40.0% 拖到 **19.6%（-20.4 pp）**、patch generation 从 92.4% 跌到 67.6% [16: §5.1, Table 2]。
6. **反馈策略上 taxonomy-guided 最优**：PRM_D（taxonomy 引导、detailed、含示例、guidance+reasoning）+10.6 pp 且步数微降；unguided PRM_S +5.8 pp 但步数暴增（51.5 vs 38.6，更长更低效）；显式动作指令 PRM_DR 仅 +4.8 pp（步数最短 34.4 但成功率最低）[16: §5.3, Table 3]。
7. 闭源 PRM 几乎"逢窗必判 suboptimal"：PRM_D 每条轨迹平均 7.24 次 PRM 调用里有 7.21 次标记为 suboptimal（"optimal windows" 仅 0.03），论文将其解读为"频繁检测到轨迹级错误" [16: §5.3, Table 4]。
8. PRM 是一种**自觉的性能-成本权衡**：base 在 $2.77/100 instances 处停在 40%；PRM_D 把成本抬到 $25.98/100（其中 policy $3.61 + PRM 调用 $22.4）换来 50.6%，即每条增量约 $0.2 的 PRM 开销、$23.2 增量成本换 10.6 个新解 [16: §5.4, Table 3–4]。
9. taxonomy 本身被宣称为 **domain-agnostic**：它源自对执行 trace 的人工检视，组织为三族——Specification Errors（任务/角色规约违反、步骤重复、终止条件无感知）、Reasoning Errors（问题误识、工具选择错、幻觉、信息处理失败）、Coordination Errors（任务脱轨、目标偏移、上下文处理失败、验证失败），每类配一个对应的 recovery action [16: §3.2.1]。
10. MCTS 式 PRM（对每步打分多条未来 rollout [1, 2]）对 SWE agent **不可行**：代码编辑动作多为不可逆，难以 spin up 并行环境实例或重置到任意中间状态 [16: §1, §2.2]。

## Assumptions

- **轨迹级低效可被"滑窗 + taxonomy"在中途可靠识别。** 方法每 n=5 步喂入最近 k=8 步给 PRM；这假设关键低效在 8 步窗口内可见。但跨越 >8 步的长程漂移（例如第 3 步进入错误子目标、第 20 步才显现）在任意单窗口里都不完整，论文未讨论窗口尺寸对"宏观 task derailment"这类长程错误的检测盲区。
- **PRM 与 policy 能力非对称是可接受的实验设定。** 强结果全部来自 C LAUDE -S ONNET-4（PRM）监督 SWE- AGENT-LM-32B（弱 policy）。这意味着 +10.6 pp 既可归因于"过程奖励建模"也可归因于"用强模型把能力经 NL guidance 灌进弱策略"——论文没做"PRM 与 policy 同模型同规模"的对照（unified 设定里同模型的开源 PRM 全部失败，恰恰暗示增益主要来自 PRM 的绝对能力而非机制）。
- **"$0.2 added cost" 是有意义的成本叙事。** abstract 用"as low as $0.2"（最低值）描述开销，但这是相对 base policy 成本（$0.028/instance）近 **10×** 的增幅；把它包装成"acceptable"假设了读者关心绝对值而非相对倍数。
- **temperature=0 的"deterministic outputs"足以让 PRM 判定稳定。** 全程 T=0；但 LLM 在 T=0 下仍非逐 token 确定（采样实现、并发、版本漂移），论文把 T=0 等同于"判定可复现"，未做多次重复测同一 PRM 输入的方差。
- **SWE-bench Verified 的 easy/medium/hard 分层 [7] 是 PRM 增益的有意义切分。** hard 只有 45 个实例（9.0%），PRM_D 在 hard 的 +4.4 pp 对应约 2 个任务翻转——把"hard 上也有增益"作为卖点建立在极小样本上。
- **taxonomy 是 domain-agnostic 的。** §3.2.1 断言三族 12 类"广泛适用于其它长程 agentic 推理领域"，但全部验证只在 SWE-bench 一个 benchmark、一个 policy backbone 上完成；domain-agnostic 是宣称而非证据。

## Method

**任务框架（§3.1）。** repository-level issue resolution [10]：给定问题描述 d、工具说明 i、仓库快照 R，agent 产出 patch p̂，当且仅当所有 pass-to-pass 测试 S_pp 保持通过且所有 fail-to-pass 测试 S_fp 由失败转通过时 p̂ 被接受。base agent 用 SWE-agent 框架 [24] 跑 ReAct 式循环 [26]，transcript H_t = (u_1,a_1,o_1,...,u_t,a_t,o_t)，其中 u=思考、a=动作（bash / str_replace_editor / submit）、o=观测；策略 π_θ 在 H_t 上条件生成下一步，严格顺序直到 submit 或耗尽 step 预算（max 75 步）。

**PRM 作为 course-corrector（§3.2）。** PRM 不替换策略、不下达流程改写，而是周期性插入自然语言 guidance，把轨迹引向"下一个最优动作"。guidance 两个属性：(1) 自然语言、按 taxonomy 分节；(2) 锚定当前上下文 H_t，供策略并入自身推理。

**Guidance 生成（§3.2.2）。** 每 n 步触发一次，输入 (i) 原问题 d、(ii) 最近 k 步转录 H_t^(k)，序列化为结构化 prompt x_t = serialize(d, H_t^(k))；PRM 产出 g_t = f_φ(x_t, T)，T 为失误 taxonomy。taxonomy 把推理"锚定"到具体失误类别（looping / redundant backtracking / subgoal drift）而非无约束 critique；g_t 是策略可直接吸收的自然语言。

**变体网格（§3.2.3, Table 1）。** unified 设定（PRM 与 policy 同模型）下沿三轴变化：(i) 简洁度（Concise / Detailed）、(ii) 是否含示例、(iii) 是否把 PRM 的 taxonomy 推理也交给策略（Guidance+Reasoning / Guidance-only），得 8 个条件（PRM_S/C/CG/D/DN/DG/DNG/DR）。canonical 变体 = **PRM_D**（taxonomy-guided、detailed、含示例、guidance+reasoning）。另设 PRM_S（不用 taxonomy、靠模型自身对轨迹级错误的理解做简单 critique）与 PRM_DR（在 guidance 中显式声明下一步该做的动作）。expert PRM 设定（强闭源 PRM 监督弱开源 policy）因成本只跑 PRM_S / PRM_D / PRM_DR 三个关键对照。

**prompt 实现（附录 A.1）。** PRM_D / PRM_DG 的 prompt（Listing 4）是一棵**结构化判定树**：对 12 个失误类别逐条输出 `DETECTED: Yes/No` + `EVIDENCE` + `RECOVERY_ACTION`，再汇总 `TASK_STATUS`（On track / Needs correction / Critical intervention required）+ `OVERALL_GUIDANCE`；PRM_DR（Listing 6）在此基础上追加 `RECOMMENDED_ACTION`（必须从 agent 可用函数集中选具体动作）；PRM_S（Listing 2）则是 7 条 free-form 低效模式提示、要求 3–5 句"detailed actionable feedback"，无逐类 yes/no 结构。

**超参（§4.2）。** policy = SWE- AGENT-LM-32B（Qwen2.5-Coder-32B-Instruct 微调 [25]）；另测 Devstral-Small-2505/2507 与 C LAUDE -S ONNET-4。T=0、top_p=1、max 75 步、超时自动提交。PRM 滑窗 k=8、触发间隔 n=5。两张 A100。

**总结。** SWE-PRM 把"事后错误分析的 taxonomy"重写为"执行中途的监督信号"：核心机制是一个**周期触发、滑窗输入、taxonomy 锚定**的 LLM critique 注入循环；它不动策略权重，靠把强模型的 NL guidance 喂回弱策略的推理上下文来纠偏。

## Eval

- **闭源 PRM 主结果（§5.1, Table 3，SWE-bench Verified 500 任务，policy 固定 SWE- AGENT-LM-32B，PRM = C LAUDE -S ONNET-4）。** base 40.0% / patch gen 92.4% / 38.64 步 / $2.77；PRM_S 45.8%（+5.8）/ 98.2% / 51.54 步（变长）/ $28.42；**PRM_D 50.6%（+10.6）/ 98.2% / 37.99 步（微降）/ $25.98**；PRM_DR 44.8%（+4.8）/ 98.2% / 34.38 步（最短）/ $24.53。对照：C LAUDE -S ONNET-4 单独做 policy = 66.6% / 100% / 61.72 步 / $121.66。
- **难度分层（§5.2, Figure 2，194 easy / 261 medium / 45 hard）。** base：easy 57.2% / medium 32.6% / hard 8.9%。PRM_D（Claude）：easy 69.1% / medium 43.3% / hard 13.3%——三层全涨，medium 与 hard 相对增益最大。开源 PRM（Figure 2a）无一改善分布，PRM_DN / PRM_DGN 进一步拖低 hard。
- **开源 PRM（§5.1, Table 2）。** 六变体 resolution 全 ≤ 40.0% base：PRM_S 19.6%（-20.4，patch gen 67.6%、灾难性）、PRM_DNG 30.0%（patch gen 54.8%）、PRM_D 38.8%（-1.2，最接近 base）。Devstral-2505/2507 同样几乎无收益（多在 ±0.2–0.4 pp 浮动）。
- **反馈策略对比（§5.3）。** PRM_S +5.8 pp 但步数 38.6→51.5（未显式标记 suboptimal，缺乏具体低效信号，runs 更长）；PRM_D +10.6 pp 步数微降（"结构化信号帮助 agent 截断低效探索而非延长"）；PRM_DR +4.8 pp 步数最短但成功率最低（"刚性处方约束太紧反而有害"）。
- **suboptimal flag 率（§5.3, Table 4）。** 闭源 PRM 几乎逢窗必判 suboptimal：PRM_D 7.21/7.24、PRM_DR 6.37/6.39；开源 PRM 标记率更低、与其更弱的整体效果一致。
- **成本拆分（§5.4, Table 4）。** PRM_D 增量主要来自 PRM 调用：policy $3.61/100 + PRM $22.4/100；以"每个新增成功的增量成本"衡量 PRM_D 最优（$23.2 换 10.6 个解）。PRM_S / PRM_DR 较差但仍胜 base 绝对性能。
- **缺失的对照。** (1) 无"PRM 与 policy 同模型同规模"的有效对照（unified 开源 PRM 全失败，但那是弱 PRM 监督弱 policy，无法隔离"PRM 机制 vs PRM 绝对能力"）；(2) **无任何对 PRM 反馈本身命中率的度量**——只报告 suboptimal 标记频率（detection 频率），从不报告这些 RECOVERY_ACTION / OVERALL_GUIDANCE 是否真的对（fix-prediction accuracy）、是否误导（regression-prediction accuracy）；(3) 无 ORM / post-hoc LLM-judge（TRAIL / MAST）的同台 baseline——论文用"事后/昂贵"作为不跑的理由，但没给出 SWE-PRM 相对它们的实际数字对比；(4) 无 n（触发间隔）/ k（窗口）的敏感性曲线——两个核心超参被宣称"固定且平衡"但无消融。

## Weaknesses

1. **强增益与"强模型监督弱策略"的混淆未被隔离。** +10.6 pp 全来自 C LAUDE -S ONNET-4 监督 SWE- AGENT-LM-32B；而 Claude 单独做 policy 就有 66.6%。"PRM 把弱策略从 40% 推到 50.6%"完全可能是强模型的能力经 guidance 泄漏进弱策略，而非"过程奖励建模"这一机制的功劳。unified 设定里同模型开源 PRM **全部失败**，这条证据本身就强烈指向"增益 = PRM 的绝对能力"而非 PRM 范式——但论文把它叙述为"开源模型不适合做 PRM"，回避了"那闭源增益还剩多少属于机制"这一问题。
2. **逢窗必判 suboptimal（7.21/7.24）= 几乎零特异性的检测器。** 一个对 99.6% 的窗口都喊"你在低效"的 course-corrector 不提供判别性信号；它更像对策略持续施加"再想想"的通用压力，而非精确的错误定位。论文把这解读为"频繁检测到错误"，但等价解读是"检测器的 precision/specificity 接近 0、optimal-window 召回 ≈ 0.03"。若增益主要来自"周期性强制反思"而非"准确归因哪一步错"，那 taxonomy 的具体内容可能远不如论文宣称重要——而这恰恰可以用"把 taxonomy 替换为通用 'review your last 8 steps' 提示"的对照来检验，论文没做。
3. **auto-fix 闭环不审计自身反馈可靠性（thesis 明确反对的 anti-pattern）。** 论文只报告 end-to-end resolution delta 与 suboptimal 标记频率，从不度量 PRM 反馈的 fix-prediction / regression-prediction 准确率。于是 +10.6 pp 无法区分"反馈是对的"与"强 base PRM 的通用纠偏压力让策略碰巧恢复"。对比 AHE [12: §4.4.2] 显式量化了 evolve agent 的 fix（5× random）/ regression（2× random）预测准确率——SWE-PRM 与 AgentDebug [13] 同属"reports neither"那一类。
4. **滑窗 k=8 与"宏观 task derailment"的检测尺度冲突。** taxonomy 把 Coordination Errors 的 task derailment 定义为"宏观漂移、放弃主任务"，但 8 步窗口天然看不到 20+ 步尺度的漂移；论文未讨论"哪些 taxonomy 类别在 8 步窗口里根本不可观测"，使 12 类里相当一部分（尤其长程协调类）的"检测"实际无从谈起。
5. **abstract 的"$0.2 added cost" 是 cherry-pick 的最小值，掩盖 ~10× 相对增幅。** base policy $0.028/instance，PRM_D 总成本 $0.26/instance；"as low as $0.2"既挑了最低、又用绝对值叙事掩盖了"为 +10.6 pp 付出近 10 倍单条成本"这一对生产更相关的事实。在 thesis "LLM-judging 在 production scale 经济不可行"的论证方向上，这正是被低估的成本结构。
6. **in-flight 介入是高方差且可净负的——但论文未把它作为风险点。** 开源 PRM_S 把 32B 从 40.0% 拖到 19.6%、patch gen 92.4%→67.6%：一个不够强的 in-flight critique 不只是"没帮上"，而是**主动把 agent 带偏**。这恰是 thesis "LLM variance is itself harmful to production-grade judgment"的鲜活证据；论文把它当作"开源模型不行"轻轻带过，没上升为"实时 LLM 介入的部署风险"这一更重要的结论。
7. **PRM_DR（显式动作指令）反而更差——这条结果削弱"determinism 越深越好"的朴素读法，需谨慎吸收。** 三档对比是 free-form（PRM_S，+5.8）< 结构化 taxonomy（PRM_D，+10.6）> 硬动作处方（PRM_DR，+4.8）。结构化判定树胜出印证 thesis 的"结构化优于自由式"，但"最确定的 PRM_DR 反而最差"提示：把决定论推到"直接规定下一个动作"会过度约束策略——这与 thesis "drive determinism as deep into the tree as possible" 表面张力，实则不矛盾（thesis 主张的是判定规则的确定化，不是替策略做动作选择），但论文自身没有这层区分，吸收时要小心别误用为"反对结构化判断"的证据。
8. **domain-agnostic 的宣称无跨域证据。** §3.2.1 称 taxonomy 广泛适用，但全部实验只在 SWE-bench Verified、一个 policy backbone 上；taxonomy 的"通用性"是断言。
9. **无代码、无检测触发逻辑细节，复现性止于 prompt 级。** prompt 全公开（这点优于 Signals [1]），但滑窗序列化的确切格式、submit 后评测脚本、PRM 触发与策略上下文拼接的实现都未释出；"逢窗必判 suboptimal"这类关键行为无法被独立验证。

## Relations

- **competes-with `13_where_llm_agents_fail_and_how` [high]**：两篇是"用 LLM-judge + 失误 taxonomy 做自动纠错"的同一范式在不同生命周期上的镜像。AgentDebug 是 deployment-time **post-hoc**：轨迹失败后回放、定位 root-cause、从 critical step 重 rollout；SWE-PRM 是 **online/in-flight**：执行中途每 5 步注入 NL guidance、不重跑、不改策略。二者都把 detector/纠错器交给 LLM（AgentDebug 强依赖 GPT-4.1，SWE-PRM 强依赖 C LAUDE -S ONNET-4），且都展示了**强烈的 base-model 依赖**（AgentDebug 换弱模型 All-Correct 32%→2–6%；SWE-PRM 换开源 PRM resolution 全部 ≤ base，最差 -20.4 pp）——这条 base-model 脆弱性在两篇独立工作（不同 lab、不同 benchmark、不同任务）上各自出现，故标 [high]。最关键的共同缺陷：两篇都**不度量纠错反馈本身的命中率**，只报 end-to-end delta，落入 thesis anti-patterns 里 "auto-feedback loops that do not audit their own reliability" 那一类。
- **competes-with `07_agent_as_a_judge` [high]**：SWE-PRM 的 PRM 本质是 Agent-as-a-Judge / LLM-as-Judge 范式被部署到"执行中途的过程监督"角色——每 n 步对滑窗做一次结构化语义判定（12 类 DETECTED:Yes/No + 证据 + recovery）。差别仅在 (a) 输出是 actionable NL guidance 而非对错评分；(b) 时机在轨迹未结束时。cost、base-model 依赖、不可复现性等问题完全继承自 AaaJ 范式。在 thesis "把 LLM-judging 留给 triaged 后的小子集、永不做 front-line"的判断方向上，SWE-PRM 把 LLM-judge 用在了**每条轨迹的每个 5 步窗口**（无前置 triage 门控），是 thesis 反对的部署模式的又一实例。
- **orthogonal `01_signals_trajectory_triage` [high]**：Signals 在 deployment-time 用**无模型调用的规则信号**做 post-hoc 轨迹分诊（哪条值得看），明确坚持"signals 不是 quality scores、不开药方"；SWE-PRM 在 online 用 **LLM-judge** 既判定又开药方（recovery action）。两者是 thesis 范式光谱的两极：Signals = lightweight rule-based front-line filter（thesis 标准实例），SWE-PRM = LLM-judge in-flight corrector（thesis 反例）。但二者可在工程上互补串联：Signals/L1 信号可作为 SWE-PRM 触发条件（仅当检测到 Loop/Stagnation 才调 PRM），把"每 5 步无条件触发"换成"信号触发的稀疏介入"——这正是 thesis 主张的"L1 信号作为升级触发器"在 in-flight 纠错上的自然落点；论文用固定 n=5 的密集触发，没走这条更省的路。
- **competes-with `08_tide_trace_diagnostics` [high]**：TIDE/TRACE 是 deployment-time post-hoc 轨迹诊断、交付给人类工程师；SWE-PRM 在 §1/§2.2 显式把自己**对立**于这类 post-mortem 方法（"cannot prevent wasted computation that has already occurred"），主张实时介入的优越性。二者方法论同源（LLM 解读轨迹找问题），分歧在时机（事后 vs 中途）与消费方（人 vs agent 自身）。SWE-PRM 的"实时优于事后"论点成立的前提是"中途介入的反馈是对的"——而这点它恰恰没度量（见 Weakness 3），所以相对 TIDE 的优越性论证不完整。
- **orthogonal `12_agentic_harness_engineering` [med]**：AHE 在 training-loop-time 用 LLM 读 trace 自动演化 harness，SWE-PRM 在 inference-time 用 LLM 读滑窗自动纠偏单条轨迹——都是"失败/低效 trace → LLM 解读 → 输出修复指令"的自动化闭环，只是时间尺度不同（AHE 持久化改 harness 文件、SWE-PRM 临时改单轨迹推理上下文）。**关键差别正是 thesis anti-pattern 的判据**：AHE 用 fix/regression precision-recall 显式 self-audit 了 evolve agent 的可靠性，SWE-PRM 完全没做。这提示在设计任何 in-flight auto-fix 时应借鉴 AHE 的 manifest/falsifiable-contract 模式，而非 SWE-PRM 的 free-form guidance——后者无法事后判决"这次纠偏到底有没有用"。
- **contradicts `11_near_miss_latent_policy_failure_detection` [high]**：Near-Miss 主张轨迹级判定应当机制化、可审、不依赖 LLM 推理（ToolGuard 生成的 Python guard code + 结构化历史搜索）；SWE-PRM 的纠偏判定完全是 LLM 调用、且强绑 C LAUDE -S ONNET-4（开源 PRM 全失败）。两篇对"trajectory-level 自动化判定该不该用 LLM"给出对立答案。在 thesis "mechanistic explanations over correlation studies"与"reproducibility-of-method matters more than reproducibility-of-numbers"的方向上 Near-Miss 受偏好；SWE-PRM 同一方法换 base 即崩（开源 PRM 甚至 -20.4 pp），机制可移植性几乎为零——这条对立应进入 contradictions.md，与 `11 ↔ 12`、`11 ↔ 13` 的对立同源。
- **competes-with `03_tsr_trajectory_search_rollouts` [low]**：SWE-PRM 在 §1/§2.2 显式拒绝了 search-based（MCTS-with-PRM [1,2] / 一步前瞻 [27]）路线，理由是 SWE 代码编辑不可逆、并行环境重置开销过高——它把"对未来多条 rollout 打分"换成"对过去滑窗打分并 NL 纠偏"。TSR 是 training-time rollout selection，与 SWE-PRM 的 inference-time process supervision 生命周期不同，但二者都涉及"用某种 reward/value 信号在 rollout 上做取舍"，构成"训练期选 rollout vs 部署期纠 rollout"的对照。对 thesis 的相关性低于上面几条。
- **orthogonal `02_agenther_hindsight_relabeling` [low]**：AgentHER 把失败轨迹 hindsight relabel 成 SFT/DPO 数据（offline 改模型），SWE-PRM 在 online 纠偏且**显式不改策略参数**。二者位于 thesis 栈的不同点（L2 数据 vs 不入栈的过程监督），但共享"taxonomy of failure modes"这一入口：AgentHER 6 类 outcome 失败、SWE-PRM 12 类 process 低效——前者面向"这条轨迹能否被重打标成训练数据"，后者面向"这条轨迹此刻该如何纠偏"。SWE-PRM 的 12 类 taxonomy 可作为 AgentHER stage-1 更细的失败模式词表的候选，但二者下游消费方完全不同。
