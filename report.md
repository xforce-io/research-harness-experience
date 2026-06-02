# Post-Deployment Agent 工程基建：Research Report

> **Version:** v16 (16 papers)
> **Last Updated:** 2026-06-02
> **Papers:** [01](notes/01_signals_trajectory_triage.md), [02](notes/02_agenther_hindsight_relabeling.md), [03](notes/03_tsr_trajectory_search_rollouts.md), [04](notes/04_agenttrace_structured_logging.md), [05](notes/05_breaking_observability_tax.md), [06](notes/06_agentseer_agentic_vulnerabilities.md), [07](notes/07_agent_as_a_judge.md), [08](notes/08_tide_trace_diagnostics.md), [09](notes/09_trajectory_guard_a_lightweight_sequence_aware.md), [10](notes/10_policy_invisible_violations_in_llm_based.md), [11](notes/11_near_miss_latent_policy_failure_detection.md), [12](notes/12_agentic_harness_engineering_observability_driven_automatic.md), [13](notes/13_where_llm_agents_fail_and_how.md), [14](notes/14_autodata.md), [15](notes/15_aevo_harnessing_agentic_evolution.md), [16](notes/16_when_agents_go_astray_course_correcting.md)
> **Thesis:** [.researcher/thesis.md](.researcher/thesis.md)

---

本报告是 `.researcher/thesis.md` 的证据与论证装置：thesis 是 spec，本报告把 thesis 的定位、设计决策与可证伪点放到迄今所有论文证据下，使之可检验、可挑战。它不是逐篇笔记、不是学术综述。每节由 thesis 的一条主张或一个可证伪点驱动，论文以**证据**身份出现在节内，一篇论文可在多节出现。

## 开篇定位

部署后 agent 改进的瓶颈不是模型能力或评估精度，而是"生产轨迹流 → 偏好/SFT 数据"之间缺失的桥。我们主张这座桥按四层栈搭建——结构化 tracing (L0) → 轻量信号分诊 (L1) → 后见之明重打标 (L2) → 模型迭代 (L3)，每个上层消费下层产物而不重做其工作。中心张力贯穿全部 16 篇：**判断（哪条轨迹/哪一步值得动作）应当尽量机制化、确定化，而非交给逐轨迹的 LLM-judge**——这既是 L1 的成本立场，也是判断层一以贯之的纪律。迄今证据在多数维度支持该立场，但 off-axis 的 LLM-driven 自动化方法（[12]–[16]）持续提供反例，且它们的真实可靠性几乎从不被自审——这正是本报告要追踪的战场。

## §1 桥的形状：四层栈 L0→L1→L2→L3 是否成立？

**文献现状。** 四层各有锚点：L0 由 AgentTrace 的结构化日志 schema [4] 与 Breaking the Observability Tax 的低成本遥测 [5] 支撑；L1 由 Signals 的三层信号分类学 [1] 领衔；L2 由 AgentHER 的失败→DPO/RLHF 重打标 [2] 与训练侧的 TSR rollout 选择 [3] 占位；L3 是对齐→部署→新轨迹的闭环。层间消费关系是真实的而非拼凑：Signals 筛出的高价值失败轨迹正是 AgentHER 的天然原料 [1][2]；Near-Miss [11] 检出的 latent failure 自带"绕过策略 vs 合规版"对，可直接成 DPO 负正样本，是 L1→L2 的可执行接口。

**残留张力。** 没有一篇论文端到端验证整条桥；每篇只夯实一层或一段接口。L2→L3 的 hindsight relabel 能否真正跑出下游 win rate 仍是 thesis 可证伪点 (b)（见 §8），当前语料无直接证据。

**对我们系统的含义。** 四层划分作为工程骨架成立，且层间接口已有论文级范例可抄；但"整桥有效"仍是假设，落地需自建端到端度量，不能从单层论文外推。

## §2 L1 的核心立场：front-line 该用轻量信号还是 LLM-judge？

这是 thesis 最强、最可被挑战的主张，也是语料里证据最密集的一节。

**支持轻量信号一侧。** Signals [1] 在 τ-bench 上以非语义规则信号达 82% informativeness、~1.5× 采样效率，且明确"signals 不是 quality scores、不开药方"。非语义检测的可行性被多条独立路线佐证：AgentSeer [6] 走 action-component 图拓扑、Trajectory Guard [9] 走序列 Siamese RNN 小代理（F1 ~0.92，宣称比 LLM-judge 快 17–27×）、Sentinel [10] 走声明式 KG 不变量（acc/F1 0.93）。这些方法各异，但共享"front-line 不调 per-trajectory LLM"的成本立场。

**LLM-judge 一侧的成本与脆弱性证据（thesis 反例，但恰好支撑 thesis 的成本论证）。** Agent-as-a-Judge [7] 是重量级语义评估的范式样板；其问题在 off-axis 论文里被反复坐实：AgentDebug [13] 把 LLM-judge 用在全部失败轨迹的全部步全部模块，单条 ALFWorld trace 仅 detection 即 40–60 次 GPT-4.1 调用，且换 base 即崩（Llama-3.3-70B All-Correct 32%→6%）；SWE-PRM [16] 把 LLM-judge 搬到执行中途每 5 步无前置门控地触发，开源 PRM 六变体**全部 ≤ base、最差 -20.4 pp**。这些不是 thesis 的反驳，而是 thesis"LLM-judging 在 production scale 经济不可行 / LLM variance 本身有害"的鲜活实证。

**残留张力。** Signals 的 82% 未被独立复现（thesis 可证伪点 a，见 §8）；学习型小代理 [9] 与声明型 oracle [10][11] 之间存在"把世界知识压进权重 vs 让它住在可审 guard code 里"的真实立场分叉，生产分诊层须显式选边。

**对我们系统的含义。** front-line 用轻量信号、把 LLM-judge 严格留给信号 triage 后的小子集——立场维持。一个具体落点：用 L1 信号（Loop/Stagnation）作 SWE-PRM 式 in-flight 介入的**稀疏触发门控**，替代 [16] 的固定 n=5 密集触发，把"每 5 步无条件 LLM 调用"换成"信号触发的稀疏介入" [1][16]。

## §3 L0 schema：silent gating constraint

**文献现状。** AgentTrace [4] 给出 operational + cognitive + contextual 三类 surface；thesis 补充 user-interaction discourse 与 system-resource state 为必要项，否则 L1 的 Interaction/Environment 信号无从计算。schema 的"沉默门控"被多篇从不同角度坐实：Sentinel [10] 的 Coverage 实验里 scope 标注缺失直接让 recall 100%→40%；Near-Miss [11] 的 history search 强依赖每次 tool call 完整保留 name+args+return value，缺字段方法即失效；AgentDebug [13] 的 Modular rollout（强制 `<memory><reflection><plan><action>` tag）比 ReAct 一体输出高 +12 pp 成功率——schema 缺失在 detector 介入前就拿走 31% 相对增益。

**残留张力。** 这些证据来自 deployment-time、training-loop-time（[12] 的组件文件 schema）、inference-time 三个不同尺度，但都指向同一结论：schema 是上游一切方法的前提。尚无一篇系统化给出"最小充分 schema 字段集"。

**对我们系统的含义。** schema 设计应被当作一等公民先行；任何 L1/L2 方法选型前先核对它依赖的字段是否在 trace 中存在。这是落地顺序问题，不是事后补救问题。

## §4 成功轨迹的 hidden friction：mature 系统最大可学习面

**文献现状。** thesis 主张约 2/3"task-completed"轨迹仍含可学习隐性摩擦——这恰是 gross failure 稀少后生产系统的主战场。Near-Miss [11] 给出可执行入口：用 guard-code-as-oracle 反查"成功轨迹中应读未读的 RO"，在 τ²-Airlines 上检出 8.6%–17.3% mutating 轨迹的 latent failure，正好覆盖 Sentinel [10] 在线 block-only 不覆盖的"outcome=correct 但 process=non-compliant"子集。SWE-PRM [16] 的 trajectory-level inefficiency taxonomy（冗余探索/动作循环/解出不终止）是同一现象在"低效"而非"违规"维度的另一切面，但它只做在线纠偏、不沉淀为训练数据。

**残留张力。** 成功轨迹挖掘目前只有"检出"证据（[11]），没有"重打标后下游训练增益"证据——直接关联可证伪点 (b)。

**对我们系统的含义。** 把成功轨迹的隐性摩擦检测（[11] 式 oracle + [16] 式低效 taxonomy 词表）接到 AgentHER [2] 的 relabel 管线，是 L1→L2 最有杠杆的具体工程，且区别于只挖失败的主流路线。

## §5 观测成本：sentinel sampling 而非 uniform downsampling

**文献现状。** Breaking the Observability Tax [5] 主张拓扑感知 sentinel 采样、以信号为升级触发器控制成本，而非均匀下采样。该原则与 L1 自然耦合：信号既是 triage 输出，也是观测升级的触发条件。Sentinel [10] 的 O(|M|) 反事实模拟给出"动作时低成本判定"的可行性下界。

**残留张力。** [5] 为 IEEE 付费、当前仅摘要级阅读（🟡），其降本倍数与 thesis 可证伪点 (c)（>5× 降本不损训练数据质量）尚无法对齐核验。

**对我们系统的含义。** 观测成本控制采"信号触发的稀疏升级"而非全量下采样；但量化目标（5×）需自建实验，不能引 [5] 数字。

## §6 判断层的确定性纪律：mechanistic oracle / 结构化决策树 / 拒绝 free-form

thesis 的方法论约束：判断链应把确定性尽量推深；规则可枚举时机制化 oracle 严格优于 LLM-judge（理由是 LLM variance 本身有害，而非成本）；判断不可避免依赖 LLM 时，须包成 narrow yes/no predicates + 显式 aggregation/threshold/abstain 的决策树，**拒绝 free-form"这条轨迹好不好？"prompting**。

**支持机制化一侧。** Near-Miss [11] 把判断委托给确定性 Python guard code + 结构化历史搜索，论证"判断准确性本身也可从语义模型转移到结构化代码"。Sentinel [10] 用声明式不变量在动作时判定，并以 policy-in-prompt 对照证明"决定性事实未进上下文 → 任何模型类判定结构性失败"（违规率仅 95.3%→40.7%、跨模型 25%–85% 不一致）。

**结构化 > 自由式的直接实证（本期关键新增）。** SWE-PRM [16] 提供了 thesis"结构化决策树优于自由式 prompt"迄今最干净的对照：taxonomy-guided 的 PRM_D（12 类 DETECTED:Yes/No + 证据 + recovery + TASK_STATUS 汇总）+10.6 pp 且步数微降，严格优于 unguided free-form 的 PRM_S（+5.8 pp 但步数 38.6→51.5 暴增）。但同一论文也给出须谨慎吸收的边界：把确定性推到"显式规定下一个动作"的 PRM_DR 反而最差（+4.8 pp）——这与 thesis"drive determinism as deep as possible"是**表面张力而非矛盾**：thesis 主张确定化的是**判定规则**，不是替策略做动作选择；[16] 自身没做这层区分，吸收时不可误用为"反对结构化判断"的证据。

**反例与脆弱性。** Agent-as-a-Judge [7]、AgentDebug [13]、SWE-PRM [16] 都是 free-form / LLM-only 判断的实例，且都展现强 base-model 依赖（换弱模型即崩）。TIDE/TRACE [8] 是 post-hoc 诊断交付人类，SWE-PRM [16] §1/§2.2 显式把自己对立于此类 post-mortem——但其"实时优于事后"成立的前提是"中途反馈是对的"，而它恰恰没度量（见 §7）。SWE-PRM 闭源 PRM 几乎逢窗必判 suboptimal（7.21/7.24，optimal-window ≈0.03）= 近零特异性检测器，提示增益可能来自"周期性强制反思"而非精确归因——这反过来削弱"taxonomy 内容很重要"的强读法，须靠"taxonomy 替换为通用 review 提示"的对照检验（论文未做）。

**对我们系统的含义。** 判定规则可枚举处一律机制化（[10][11] 路线）；不可避免用 LLM 处，强制 [16] PRM_D 式结构化 yes/no 决策树而非 free-form；但 determinism 止于判定，不替策略选动作（PRM_DR 教训）。

## §7 off-axis：harness meta-optimization 与 in-flight 过程监督的 self-audit 谱

这组论文不在四层栈内，但持续压力测试 thesis 的 anti-pattern 判据——"auto-fix loops that do not audit their own reliability"。

**self-audit 严格度谱。** AHE [12]（training-loop-time 改 harness 文件）显式量化 evolve agent 的 fix-prediction（5× random）与 regression-prediction（2× random）准确率；AEVO [15] 有 harness evaluator 只读隔离防 reward hacking、但无 per-edit 预言核对；Autodata [14] 两者均弱、spec gaming 部分绕过。三篇构成 **AHE > AEVO > Autodata** 的自审严格度谱。AgentDebug [13]（inference-time re-rollout）与 SWE-PRM [16]（inference-time in-flight 纠偏）落在谱的最差端：两者都只报 end-to-end delta（[13] +26% relative，[16] +10.6 pp），**从不度量纠错反馈本身的命中率**——无法区分"反馈对了"与"强 base 的通用纠偏压力让策略碰巧恢复"。

**SWE-PRM [16] 的独特贡献——一个新的生命周期位置。** 它把失误 taxonomy 从事后诊断搬到执行中途实时纠偏，是 thesis 四层栈未显式覆盖的"判断层放在哪一时刻"对照（online/in-flight，不改策略参数）。它同时暴露：in-flight 弱 LLM 介入不止"没帮上"而是**主动带偏**（开源 PRM_S 把 32B 从 40.0% 拖到 19.6%、patch gen 92.4%→67.6%），把 thesis"LLM variance is harmful"从离线评估推广到在线纠偏。该 lifecycle 维度不属现有 off-axis"harness 自演化"桶——已在本期 contradictions.md 提 taxonomy 扩展供人裁决。

**残留张力（强增益的归因）。** SWE-PRM 的 +10.6 pp 全来自 C LAUDE -S ONNET-4 监督弱 SWE-AGENT-LM-32B，而 Claude 单独做 policy 即 66.6%——增益究竟是"过程奖励建模"机制还是"强模型能力经 NL guidance 泄漏进弱策略"未被隔离；unified 设定下同模型开源 PRM 全失败，恰指向后者。

**对我们系统的含义。** 任何 auto-fix（无论 training-loop、inference-time 还是 in-flight）必须以 AHE [12] 的 falsifiable-manifest（predicted_fixes/risk_tasks → 下一轮核对）为最低自证起点；AEVO [15] 的 evaluator 隔离是必要但不充分。off-axis 演化回路成本（AEVO ~3× baseline、SWE-PRM PRM 开销近 10× 单条成本）使其在飞轮初期不引入。

## §8 可证伪点追踪

thesis 在三条主张上自陈可证伪。逐条追踪当前证据与下一步可解争议的观察。

**(a) 轻量信号在真实非 τ-bench 语料上达 >70% informativeness。** 当前唯一正面数字是 Signals [1] 的 82%，但限于 τ-bench 且未独立复现。旁证：Trajectory Guard [9] F1 ~0.92、Sentinel [10] F1 0.93、Near-Miss [11] code-gen 路径 P=R=1.00（但单标注者 ground truth 偏弱）——指标不同维度、不可直接折算为 informativeness。**待决观察：** 在一个非 τ-bench 生产风格语料上跑非语义信号并报 informativeness。状态：未证。

**(b) L1-triaged 轨迹的 hindsight relabel 跑出对随机采样偏好数据的下游 win rate。** AgentHER [2] 给出 relabel 管线、Near-Miss [11] 给出成功轨迹的可执行 relabel 入口，但**无任何下游训练 win rate 证据**。AHE [12] 的 component ablation（long-term memory externalize +5.6 pp，无需 SFT）甚至从侧面质疑"必须 model-side relabel"。**待决观察：** L1-triaged relabel 数据 vs 随机采样数据的同条件 DPO 对比。状态：未证，且有反向工程对照点。

**(c) 信号驱动 sentinel 采样 >5× 降本不损训练数据质量。** Breaking the Observability Tax [5] 主张该方向但仅摘要级、无可核数字；Sentinel [10] 给出 O(|M|) 低成本判定的可行性下界。**待决观察：** 自建端到端实验测降本倍数与下游数据质量的联合曲线。状态：未证。

三条全部未证——thesis 当前是**有证据支撑方向、无端到端验证**的工作论点。任一条的反向证据都应触发重构。

## 版本更新日志
| 版本 | 日期 | 新增论文 | 关键变化 |
|------|------|---------|---------|
| v16 | 2026-06-02 | [16] SWE-PRM / When Agents go Astray | 报告首次创建（thesis-anchored 骨架）。[16] 引入"判断层 lifecycle = in-flight/online"新维度，进入 §6（结构化 > 自由式的最干净对照：PRM_D +10.6 严格优于 free-form PRM_S +5.8）与 §7（self-audit 谱最差端，与 [13] 同属 report-neither）；记录与 [11] 的 cross-paper contradiction、一条 taxonomy 扩展提案、一条 charter tension（trace 轴是否覆盖 in-flight course-correction）。 |
