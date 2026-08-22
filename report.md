# Experience 闭环：Research Report

> **Version:** v20 (23 papers)
> **Last Updated:** 2026-08-22
> **Papers:** [01](notes/active/01_signals_trajectory_triage.md), [02](notes/active/02_agenther_hindsight_relabeling.md), [03](notes/active/03_tsr_trajectory_search_rollouts.md), [04](notes/active/04_agenttrace_structured_logging.md), [05](notes/active/05_breaking_observability_tax.md), [06](notes/active/06_agentseer_agentic_vulnerabilities.md), [07](notes/active/07_agent_as_a_judge.md), [08](notes/active/08_tide_trace_diagnostics.md), [09](notes/active/09_trajectory_guard_a_lightweight_sequence_aware.md), [10](notes/active/10_policy_invisible_violations_in_llm_based.md), [11](notes/active/11_near_miss_latent_policy_failure_detection.md), [12](notes/active/12_agentic_harness_engineering_observability_driven_automatic.md), [13](notes/active/13_where_llm_agents_fail_and_how.md), [14](notes/active/14_autodata.md), [15](notes/active/15_aevo_harnessing_agentic_evolution.md), [16](notes/active/16_when_agents_go_astray_course_correcting.md), [17](notes/active/17_masprism_lightweight_failure_attribution_for_multi.md), [18](notes/active/18_agentracer_who_is_inducing_failure_in.md), [19](notes/active/19_harness_r1_learning_to_edit_executable.md), [20](notes/active/20_ai4ai_at_test_time_strong_to.md), [21](notes/active/21_rethinking_the_evaluation_of_harness_evolution.md), [22](notes/active/22_sample_efficient_learning_from_agent_experience.md), [23](notes/active/23_trace_turn_level_reward_assignment_via.md)
> **Thesis:** [.researcher/thesis.md](.researcher/thesis.md)

---

本报告是 `.researcher/thesis.md` 的证据与论证装置：thesis 是 spec，本报告把主张与可证伪点放到迄今所有论文证据下。每节由闭环的一条主张驱动，论文以**证据**身份出现，一篇可在多节出现。

## 开篇定位

Silver & Sutton 把改进介质换成 agent 与环境的交互流；姚顺雨把下半场瓶颈换成任务与评价。本仓主张把这两点收成闭环：**流（捕获）→ 更新（权重或 harness）→ 评价**。判断（哪条轨迹值得动作）尽量机制化，而不是逐轨迹 LLM-judge。下文先按三节给证据，再保留捕获纪律与 harness 对照的展开。

## §1 流（捕获）

**文献现状。** schema 与成本由 AgentTrace [4]、Observability Tax [5] 支撑；轻量分诊由 Signals [1] 领衔（τ-bench 82% informativeness、~1.5×），AgentSeer [6] / Trajectory Guard [9] / Sentinel [10] / Near-Miss [11] 从拓扑、序列、声明式不变量、成功轨迹 latent failure 补面。SWE-PRM [16] 的低效 taxonomy 与 MASPrism [17] 的 prefill 内部信号也可作捕获门控，而不是只做事后评判。

**残留张力。** Signals 的 82% 未在非 τ-bench 上复现。学习型小代理 [9] 与声明型 oracle [10][11] 的立场分叉仍在。

**对我们系统的含义。** 前线用轻量信号；LLM-judge 只留在分诊后的小子集。

## §2 更新（权重或 harness）

**文献现状。** 改权重：AgentHER [2] 把失败重打标成 DPO/RLHF；TSR [3] 是训练期镜像；Experience Distillation [22]（`22_sample_efficient_learning_from_agent_experience`）表明已收集经验上直接 SFT 几乎恢复不了 ICL 增益，监督来源比「有数据」更关键；TRACE turn-credit [23]（`23_trace_turn_level_reward_assignment_via`）用 turn-level TD 混合 outcome GRPO，在搜索域打过纯轨迹 advantage，但 SWE-bench 类等预算对照仍缺。改 harness：AHE [12]、Autodata [14]、AEVO [15]、Harness-R1 [19]、AI4AI [20]（`20_ai4ai_at_test_time_strong_to`）走不改 target 权重的路径——[20] 在冻结弱模型上用强 builder 编译推理时 harness，宏平均 0.488→0.763。

**残留张力。** hindsight 后的下游 win rate 未测；[23] 的过程信用与 thesis「显式过程监督」定义仍有 scope 差；[20] 头条 best-of 会夸大代表性。

**对我们系统的含义。** 两条更新路径平行：relabel 权重，或编辑 harness。不要把 raw dump 当训练。

## §3 评价

**文献现状。** Rethinking Harness Evolution Eval [21]（`21_rethinking_the_evaluation_of_harness_evolution`）把 evolution 与 parallel/sequential sampling 锁进同一预算，held-out 上 evolved harness 平均仅 +0.6，表观收益常可被多采样解释。[20] 的 holdout 迁移与 [21] 的 disjoint 测量互补，但任务族不同，不能互否。AHE [12] 量化 fix/regression 预测；Harness-R1 [19] 用重跑 Δ reward，但仍缺 [12] 式 falsifiable 合同。Agent-as-a-Judge [7]、TIDE [8]、AgentDebug [13]、AgenTracer [18] 是对照组：重量级 judge 或不审计反馈命中率。

**残留张力。** 下一代通用模型不改 env 是否压过 harness 改进——未测。人类偏好 vs 接地 outcome 的成本——未测。

**对我们系统的含义。** 没有同预算采样对照和 held-out，就不把 harness 涨点写成 SOTA。

## §4 可证伪点追踪

thesis 三条可证伪点的当前状态（支持 / 未测 / 张力）：

- **流**：轻量信号在非 τ-bench 上 >70% informativeness，或未分诊端到端 RL 打赢闭环。状态：**未测**（唯一正面数字仍是 Signals [1] 的 τ-bench 82%，未独立复现；[9][10][11][17] 是旁证、指标不可折算）。
- **更新**：L1 子集 hindsight 相对随机偏好有下游胜率；或 SWE-bench 类等预算过程监督优于 outcome-only。状态：**张力**（[2][11] 给出管线、[23] 在搜索域支持密集信用，但无目标 falsifier 上的 win rate；[19][20] 显示不改权重也能涨点，旁证压力而非否证）。
- **评价**：下一代通用模型不改 env/eval 即压过 harness 改进；或人类偏好持续更便宜更稳。状态：**张力**（[21] 表明评价协议可翻转「进化是否有效」；[20] 显示冻结模型+harness 可超更强裸模型——方向支持「评价/系统重于 scale」，但未做参数 scale-only 头对头，第二条未测）。

下文 §5 起是捕获纪律与 harness 对照的展开，不替代上面三节。

## §5 桥的形状：捕获如何接到更新

**文献现状。** 捕获与更新的层间消费是真实的：Signals 筛出的高价值失败轨迹正是 AgentHER 的天然原料 [1][2]；Near-Miss [11] 检出的 latent failure 自带"绕过策略 vs 合规版"对，可直接成 DPO 负正样本。Harness-R1 [19] 另开一条**不改 target 权重、只编 harness** 的旁路（+9.3 pp），与 model-side relabel 平行而非替代。

**残留张力。** 没有一篇论文端到端验证整条闭环；每篇只夯实一节或一段接口。L2→权重的 hindsight relabel 能否真正跑出下游 win rate 仍落在「更新」可证伪点上。

**对我们系统的含义。** 节间接口已有论文级范例；整环有效仍是假设，落地需自建端到端度量。

## §6 L1 的核心立场：front-line 该用轻量信号还是 LLM-judge？

这是 thesis 最强、最可被挑战的主张，也是语料里证据最密集的一节。

**支持轻量信号一侧。** Signals [1] 在 τ-bench 上以非语义规则信号达 82% informativeness、~1.5× 采样效率，且明确"signals 不是 quality scores、不开药方"。非语义检测的可行性被多条独立路线佐证：AgentSeer [6] 走 action-component 图拓扑、Trajectory Guard [9] 走序列 Siamese RNN 小代理（F1 ~0.92，宣称比 LLM-judge 快 17–27×）、Sentinel [10] 走声明式 KG 不变量（acc/F1 0.93）。这些方法各异，但共享"front-line 不调 per-trajectory LLM"的成本立场。同一对峙在 failure attribution（"哪一步坏"）任务上由 MASPrism [17] 推到极致：用 SLM 在 prefill 阶段读 trace 时天然算出的两个内部量（token-level NLL + step-to-step attention）定位根因，**两次 prefill、0 output token、6.69× 端到端延迟降**（2.66s vs LLM-judge 基线 A2P 17.82s），Qwen3-0.6B 即可用且跨模型族稳健，在 Who&When-HC / TRAIL-GAIA 上反超 GPT-4o / o3 / Gemini-2.5-Pro 基线——把"front-line 不调 per-trajectory LLM"的立场从 triage 延伸到 attribution。AgenTracer [18] 是同一对峙里**结构暧昧**的一个数据点：一个 8B 训练小模型在 Who&When 上 agent-level 击败 GPT-4.1/Claude-4-Sonnet/Gemini-2.5-Pro（最高 +18.18%），表面是"小模型胜 frontier judge"的强证据——但它的"轻量"只在**推理端**成立，训练集 TracerTraj 靠 DeepSeek-R1 analyzer + 可重跑环境的 counterfactual replay 在沙盒里造出来（且 analyzer 被喂 ground-truth G、生产 trace 既无 G 也通常不可逐步重放），且替代物仍是一个会输出 free-form `⟨think⟩` 的 trained LLM reasoner。它证明的是"不需要 frontier judge"，而非 thesis 更强主张的"应当机制化 / 不需要 LLM judge"——与 MASPrism [17] 的零训练、确定性聚合形成 lightweight 路线内部"trained small judge vs frozen prefill signal"的直接分叉（同基准落点：AG [18] 37.30 略优、长 trace HC [17] 27.59 明显优 [18] 20.68）。

**LLM-judge 一侧的成本与脆弱性证据（thesis 反例，但恰好支撑 thesis 的成本论证）。** Agent-as-a-Judge [7] 是重量级语义评估的范式样板；其问题在 off-axis 论文里被反复坐实：AgentDebug [13] 把 LLM-judge 用在全部失败轨迹的全部步全部模块，单条 ALFWorld trace 仅 detection 即 40–60 次 GPT-4.1 调用，且换 base 即崩（Llama-3.3-70B All-Correct 32%→6%）；SWE-PRM [16] 把 LLM-judge 搬到执行中途每 5 步无前置门控地触发，开源 PRM 六变体**全部 ≤ base、最差 -20.4 pp**。这些不是 thesis 的反驳，而是 thesis"LLM-judging 在 production scale 经济不可行 / LLM variance 本身有害"的鲜活实证。

**残留张力。** Signals 的 82% 未被独立复现（thesis 可证伪点 a，见 §8）；学习型小代理 [9] 与声明型 oracle [10][11] 之间存在"把世界知识压进权重 vs 让它住在可审 guard code 里"的真实立场分叉，生产分诊层须显式选边。

**对我们系统的含义。** front-line 用轻量信号、把 LLM-judge 严格留给信号 triage 后的小子集——立场维持。一个具体落点：用 L1 信号（Loop/Stagnation）作 SWE-PRM 式 in-flight 介入的**稀疏触发门控**，替代 [16] 的固定 n=5 密集触发，把"每 5 步无条件 LLM 调用"换成"信号触发的稀疏介入" [1][16]。

## §7 L0 schema：silent gating constraint

**文献现状。** AgentTrace [4] 给出 operational + cognitive + contextual 三类 surface；thesis 补充 user-interaction discourse 与 system-resource state 为必要项，否则 L1 的 Interaction/Environment 信号无从计算。schema 的"沉默门控"被多篇从不同角度坐实：Sentinel [10] 的 Coverage 实验里 scope 标注缺失直接让 recall 100%→40%；Near-Miss [11] 的 history search 强依赖每次 tool call 完整保留 name+args+return value，缺字段方法即失效；AgentDebug [13] 的 Modular rollout（强制 `<memory><reflection><plan><action>` tag）比 ReAct 一体输出高 +12 pp 成功率——schema 缺失在 detector 介入前就拿走 31% 相对增益。

**残留张力。** 这些证据来自 deployment-time、training-loop-time（[12] 的组件文件 schema）、inference-time 三个不同尺度，但都指向同一结论：schema 是上游一切方法的前提。尚无一篇系统化给出"最小充分 schema 字段集"。

**对我们系统的含义。** schema 设计应被当作一等公民先行；任何 L1/L2 方法选型前先核对它依赖的字段是否在 trace 中存在。这是落地顺序问题，不是事后补救问题。

## §8 成功轨迹的 hidden friction：mature 系统最大可学习面

**文献现状。** thesis 主张约 2/3"task-completed"轨迹仍含可学习隐性摩擦——这恰是 gross failure 稀少后生产系统的主战场。Near-Miss [11] 给出可执行入口：用 guard-code-as-oracle 反查"成功轨迹中应读未读的 RO"，在 τ²-Airlines 上检出 8.6%–17.3% mutating 轨迹的 latent failure，正好覆盖 Sentinel [10] 在线 block-only 不覆盖的"outcome=correct 但 process=non-compliant"子集。SWE-PRM [16] 的 trajectory-level inefficiency taxonomy（冗余探索/动作循环/解出不终止）是同一现象在"低效"而非"违规"维度的另一切面，但它只做在线纠偏、不沉淀为训练数据。

**残留张力。** 成功轨迹挖掘目前只有"检出"证据（[11]），没有"重打标后下游训练增益"证据——直接关联可证伪点 (b)。

**对我们系统的含义。** 把成功轨迹的隐性摩擦检测（[11] 式 oracle + [16] 式低效 taxonomy 词表）接到 AgentHER [2] 的 relabel 管线，是 L1→L2 最有杠杆的具体工程，且区别于只挖失败的主流路线。

## §9 观测成本：sentinel sampling 而非 uniform downsampling

**文献现状。** Breaking the Observability Tax [5] 主张拓扑感知 sentinel 采样、以信号为升级触发器控制成本，而非均匀下采样。该原则与 L1 自然耦合：信号既是 triage 输出，也是观测升级的触发条件。Sentinel [10] 的 O(|M|) 反事实模拟给出"动作时低成本判定"的可行性下界。

**残留张力。** [5] 为 IEEE 付费、当前仅摘要级阅读（🟡），其降本倍数与 thesis 可证伪点 (c)（>5× 降本不损训练数据质量）尚无法对齐核验。

**对我们系统的含义。** 观测成本控制采"信号触发的稀疏升级"而非全量下采样；但量化目标（5×）需自建实验，不能引 [5] 数字。

## §10 判断层的确定性纪律：mechanistic oracle / 结构化决策树 / 拒绝 free-form

thesis 的方法论约束：判断链应把确定性尽量推深；规则可枚举时机制化 oracle 严格优于 LLM-judge（理由是 LLM variance 本身有害，而非成本）；判断不可避免依赖 LLM 时，须包成 narrow yes/no predicates + 显式 aggregation/threshold/abstain 的决策树，**拒绝 free-form"这条轨迹好不好？"prompting**。

**支持机制化一侧。** Near-Miss [11] 把判断委托给确定性 Python guard code + 结构化历史搜索，论证"判断准确性本身也可从语义模型转移到结构化代码"。Sentinel [10] 用声明式不变量在动作时判定，并以 policy-in-prompt 对照证明"决定性事实未进上下文 → 任何模型类判定结构性失败"（违规率仅 95.3%→40.7%、跨模型 25%–85% 不一致）。

**确定性谱上的中间形态（本期新增）。** MASPrism [17] 给出 thesis 方法论约束里"判断不可避免依赖 LLM 时，用结构化决策规则包裹"这一中间态的干净实例：它不让 LLM 生成任何 free-form 判词，而是把 LLM 在 prefill 阶段天然算出的内部量（token-NLL、step-attention）喂进**确定性聚合公式**（candidate score = attention 相关性 × NLL 方向性 contrast × multi-symptom consensus），并显式分离两种度量职责——NLL 找 symptom（失败暴露处）、attention 找 source（失败引入处），不混为一谈（与 Signals "signals 不是 quality scores" 同纪律）。论文声称该流程确定（同模型/prompt/参数下重复运行排名不变），位置在 Near-Miss [11] 全规则 oracle 与 Agent-as-a-Judge [7] free-form LLM-judge **之间**。但它也暴露这条中间路线的代价边界：整条 routing 机制建在 attention-as-explanation 上，而该假设本身有争议（论文自引 Jain & Wallace 的"Attention is not Explanation"），§5.3 退守为"routing evidence, not causal estimators"——即 mechanism 越"软"，可审性与可移植性越弱。对我们系统的含义：能枚举规则处仍优先全机制化（[10][11]）；必须依赖 LLM 内部信号处，确定性聚合 + 职责分离是可取形态，但"确定性"应被实测（[17] 仅断言未量化跨 GPU/精度方差），且不能把可争议的内部量当因果证据。AgenTracer [18] 在同一 attribution 任务上走了相反方向：它把判定交给一个 GRPO 训练的 8B reasoner，输出自由 `⟨think⟩` 推理再抽 `agentID|stepID`，且**不报任何确定性/跨运行方差**——相对 [17] 的"零解码、确定性聚合、重复运行排名不变"，[18] 在 thesis"LLM variance 本身有害于生产判断"判据上是退一步而非进一步：它把 free-form judge 换成了"更小但仍随机"的 judge，而非把判定机制化。这说明"用小模型"并不自动满足确定性纪律——确定性取决于判定**形式**（机制聚合 vs 采样推理），不取决于模型**尺寸**。

**结构化 > 自由式的直接实证（本期关键新增）。** SWE-PRM [16] 提供了 thesis"结构化决策树优于自由式 prompt"迄今最干净的对照：taxonomy-guided 的 PRM_D（12 类 DETECTED:Yes/No + 证据 + recovery + TASK_STATUS 汇总）+10.6 pp 且步数微降，严格优于 unguided free-form 的 PRM_S（+5.8 pp 但步数 38.6→51.5 暴增）。但同一论文也给出须谨慎吸收的边界：把确定性推到"显式规定下一个动作"的 PRM_DR 反而最差（+4.8 pp）——这与 thesis"drive determinism as deep as possible"是**表面张力而非矛盾**：thesis 主张确定化的是**判定规则**，不是替策略做动作选择；[16] 自身没做这层区分，吸收时不可误用为"反对结构化判断"的证据。

**反例与脆弱性。** Agent-as-a-Judge [7]、AgentDebug [13]、SWE-PRM [16] 都是 free-form / LLM-only 判断的实例，且都展现强 base-model 依赖（换弱模型即崩）。TIDE/TRACE [8] 是 post-hoc 诊断交付人类，SWE-PRM [16] §1/§2.2 显式把自己对立于此类 post-mortem——但其"实时优于事后"成立的前提是"中途反馈是对的"，而它恰恰没度量（见 §7）。SWE-PRM 闭源 PRM 几乎逢窗必判 suboptimal（7.21/7.24，optimal-window ≈0.03）= 近零特异性检测器，提示增益可能来自"周期性强制反思"而非精确归因——这反过来削弱"taxonomy 内容很重要"的强读法，须靠"taxonomy 替换为通用 review 提示"的对照检验（论文未做）。

**对我们系统的含义。** 判定规则可枚举处一律机制化（[10][11] 路线）；不可避免用 LLM 处，强制 [16] PRM_D 式结构化 yes/no 决策树而非 free-form；但 determinism 止于判定，不替策略选动作（PRM_DR 教训）。

## §11 off-axis：harness meta-optimization 与 in-flight 过程监督的 self-audit 谱

这组论文不在四层栈内，但持续压力测试 thesis 的 anti-pattern 判据——"auto-fix loops that do not audit their own reliability"。

**self-audit 严格度谱（两轴）。** 原先单轴 **AHE > AEVO > Autodata** 已不够：[19] Harness-R1 引入第二条轴——**proposer 是否 outcome-RL 可训练**。轴 (i) 推理期 falsifiable 合同：AHE [12] 显式量化 evolve agent 的 fix-prediction（5× random）与 regression-prediction（2× random）；AEVO [15] 有 harness evaluator 只读隔离、无 per-edit 预言；Autodata [14] 两者均弱。轴 (ii) 训练期信号接地：Harness-R1 [19] 用冻结 target 重跑的 Δ reward 做 GRPO（K=8），9B outcome-trained engineer 平均 53.6% 超最强 frontier prompted editor（GLM-5.2 48.8%），且 outcome-trained 比 supervised-only 高 7.1 pp——直接支持"编辑是否有效只能由重跑判断，不能由文本合理性判断" [19: §4.2]。但 [19] **无** [12] 式 predicted_fixes/risk_tasks 合同，reward 绑定同批任务（transductive），总 GPU-hour 未披露。AgentDebug [13]、SWE-PRM [16] 与 AgenTracer [18] 仍落在两轴最差端：只报 end-to-end delta（[13] +26% relative，[16] +10.6 pp，[18] +4.8∼14.2%），从不度量纠错反馈命中率。AgenTracer [18] 与 Harness-R1 [19] 独立坐实固定 Self-Refine 有害（[18] CRITIC iter −4.9%/−5.5%；[19] Self-Refine 三 benchmark 平均 −2.5 pp）[18: §5.3; 19: §4.2]。

**Harness-R1 [19] 的独特贡献——可训练的 harness engineer。** 与 [12]/[14]/[15] 的"proposer 参数不更新"相反，[19] 把 failure-conditioned、lifecycle-wide 编辑形式化为在线 RL 问题：失败 packet → 四 hook 可执行 patch → 重跑奖励 → 更新 9B engineer。跨 20 未见目标平均 +7.06 pp；仅 10 条失败生成的 patch 泛化到 1,270 held-out 任务 +8.9±1.5 pp [19: §4.3–4.4]。与 AgentHER [2] 构成**平行改进路径**（改 harness vs relabel target），且可 co-evolve（target SFT 后仍 +5.0 pp）[19: §4.2]——对 thesis 主桥是旁证压力而非否证（见 contradictions.md）。

**SWE-PRM [16] 的独特贡献——一个新的生命周期位置。** 它把失误 taxonomy 从事后诊断搬到执行中途实时纠偏，是 thesis 四层栈未显式覆盖的"判断层放在哪一时刻"对照（online/in-flight，不改策略参数）。它同时暴露：in-flight 弱 LLM 介入不止"没帮上"而是**主动带偏**（开源 PRM_S 把 32B 从 40.0% 拖到 19.6%、patch gen 92.4%→67.6%），把 thesis"LLM variance is harmful"从离线评估推广到在线纠偏。

**残留张力（强增益的归因）。** SWE-PRM 的 +10.6 pp 全来自 C LAUDE -S ONNET-4 监督弱 SWE-AGENT-LM-32B，而 Claude 单独做 policy 即 66.6%——增益究竟是"过程奖励建模"机制还是"强模型能力经 NL guidance 泄漏进弱策略"未被隔离；unified 设定下同模型开源 PRM 全失败，恰指向后者。[19] 的 +9.3 pp 虽有 held-out 泛化，但 same-batch 训练 reward 与未披露的 RL 重跑成本使生产可移植性仍不确定。

**对我们系统的含义。** 生产 auto-fix 应两轴过线：训练/接受信号尽量 outcome-grounded（[19] 方向），推理期仍要 falsifiable 合同（[12] 方向）；AEVO [15] 的 evaluator 隔离是必要但不充分。off-axis 演化回路成本（AEVO ~3× baseline、SWE-PRM PRM 近 10×、[19] K=8×全批重跑未量化）使其在飞轮初期不引入；若引入 [19]，应用 [1] Signals 对失败子集做前置 triage 摊薄重跑。

## §12 历史可证伪点（v19 四层栈口径，已由 §4 取代）

thesis 在三条主张上自陈可证伪。逐条追踪当前证据与下一步可解争议的观察。

**(a) 轻量信号在真实非 τ-bench 语料上达 >70% informativeness。** 当前唯一正面数字是 Signals [1] 的 82%，但限于 τ-bench 且未独立复现。旁证：Trajectory Guard [9] F1 ~0.92、Sentinel [10] F1 0.93、Near-Miss [11] code-gen 路径 P=R=1.00（但单标注者 ground truth 偏弱）、MASPrism [17] 在非 τ-bench 的 Who&When / TRAIL 上以 prefill 内部信号做 attribution（HC Top-1 27.59%、GAIA Loc.Acc 0.591）——但这些都是**不同维度的指标**（detection F1 / latent-failure P-R / attribution accuracy），不可直接折算为 informativeness，且 MASPrism 仅在"全失败"基准评测、HC Top-1 绝对值仍 <30%。AgenTracer [18] 在同一非 τ-bench 的 Who&When 上 agent-level 达 63–69%、step-level 仅 20.68%（HC），但它是**训练化 LLM judge**而非非语义信号——对"轻量**信号**达 >70% informativeness"这一可证伪点几乎不构成正面证据，反而提示：在 attribution 这一维度，连训练专用模型的 step-level 精确命中都 <1/4，非语义信号要在生产语料上达到 informativeness 阈值的难度被进一步放大。**待决观察：** 在一个非 τ-bench 生产风格语料上跑非语义信号并报 informativeness。状态：未证。

**(b) L1-triaged 轨迹的 hindsight relabel 跑出对随机采样偏好数据的下游 win rate。** AgentHER [2] 给出 relabel 管线、Near-Miss [11] 给出成功轨迹的可执行 relabel 入口，但**无任何下游训练 win rate 证据**。AHE [12] 的 component ablation（long-term memory externalize +5.6 pp，无需 SFT）与 Harness-R1 [19] 的 harness-only 路径（+9.3 pp，SFT 后仍 +5.0 pp co-evolve）从侧面质疑"必须 model-side relabel"——两者都是**不改偏好数据飞轮**也能涨点的旁证，但都不替代 (b) 本身的 DPO 对照。**待决观察：** L1-triaged relabel 数据 vs 随机采样数据的同条件 DPO 对比；可选第三臂对照 harness-edit [19]。状态：未证，且反向工程对照点增多。

**(c) 信号驱动 sentinel 采样 >5× 降本不损训练数据质量。** Breaking the Observability Tax [5] 主张该方向但仅摘要级、无可核数字；Sentinel [10] 给出 O(|M|) 低成本判定的可行性下界。**待决观察：** 自建端到端实验测降本倍数与下游数据质量的联合曲线。状态：未证。

三条全部未证——thesis 当前是**有证据支撑方向、无端到端验证**的工作论点。任一条的反向证据都应触发重构。

## 版本更新日志
| 版本 | 日期 | 新增论文 | 关键变化 |
|------|------|---------|---------|
| v20 | 2026-08-22 | [20] AI4AI · [21] Rethinking harness eval · [22] Experience distillation · [23] TRACE turn-credit | 报告骨架改为闭环三节（流 / 更新 / 评价）。[20][21] 进评价（及 [20] 的 harness 更新）；[22][23] 进更新。§4 按新三条可证伪点给 未测/张力 状态。 |
| v16 | 2026-06-02 | [16] SWE-PRM / When Agents go Astray | 报告首次创建（thesis-anchored 骨架）。[16] 引入"判断层 lifecycle = in-flight/online"新维度，进入 §6（结构化 > 自由式的最干净对照：PRM_D +10.6 严格优于 free-form PRM_S +5.8）与 §7（self-audit 谱最差端，与 [13] 同属 report-neither）；记录与 [11] 的 cross-paper contradiction、一条 taxonomy 扩展提案、一条 charter tension（trace 轴是否覆盖 in-flight course-correction）。 |
| v17 | 2026-06-02 | [17] MASPrism / Lightweight Failure Attribution | [17] 是 thesis "lightweight signal beats LLM-judge" 在 **failure attribution**（"哪一步坏"）任务上的高质量正例：prefill-only SLM 内部信号（token-NLL 找 symptom + step-attention 路由 source），两次 prefill / 0 output token / 6.69× 延迟降，与 AgentDebug [13] 同任务同基准（Who&When）方法论尖锐对立。进入 §2（轻量信号侧新增 attribution 维度证据）、§6（确定性谱**中间形态**——确定性聚合 over LLM-internal 信号、无 free-form，但 mechanism 建在有争议的 attention-as-explanation 上，是该路线代价边界的实例）、§8(a)（非 τ-bench 语料旁证，但指标维度不同且仅全失败基准）。提一条 taxonomy 扩展提案（within-trace attribution 作 triage/evaluation 之外的独立子任务）入 contradictions.md。无 vs-thesis 矛盾。 |
| v18 | 2026-06-03 | [18] AgenTracer / Who Is Inducing Failure | [18] 与 [17] 同任务同基准（Who&When）但**相反实现**：trained 8B tracer（Qwen3-8B + 多粒度 GRPO，数据靠 DeepSeek-R1 + counterfactual replay + fault injection 在沙盒造）vs [17] 零训练确定性聚合。**结构暧昧样本**：表面是"小模型胜 frontier judge"（agent-level +18.18%），但"轻量"只在推理端、且替代物仍是 free-form `⟨think⟩` 的随机 LLM judge。进入 §2（lightweight 路线内部 trained-judge vs frozen-signal 分叉 + inference-only 轻量的 caveat）、§6（同任务走相反方向——确定性取决于判定形式不取决于模型尺寸，[18] 是退一步）、§7（self-audit 谱最差端，与 [13][16] 同列：只报 +4.8∼14.2% 不审计 feedback 命中率）、§8(a)（非 τ-bench attribution 数据点，但作训练化 judge 对"非语义信号 >70% informativeness"几乎非正面证据）。无 vs-thesis 矛盾；提一条 taxonomy 扩展提案（强化 [17] 的 within-trace attribution 子桶 + learned-judge vs frozen-mechanistic 子轴）入 contradictions.md。 |
| v19 | 2026-08-05 | [19] Harness-R1 / Learning to Edit Executable Runtime Harnesses | [19] 把 harness 编辑形式化为**可训练 9B engineer + outcome GRPO**（冻结 target 重跑 Δ reward），与 [12]/[14]/[15] 的"proposer 不更新"分叉。进入 §1（桥的旁路：不改 target 权重也可 +9.3 pp）、§7（self-audit 谱拆两轴——训练期 outcome 接地 vs 推理期 falsifiable 合同；[19] 强于 [14]/[15] 但仍弱于 [12] 的 manifest）、§8(b)（model-side relabel 的必要性再受旁证压力）。与 [18] 独立复现 Self-Refine 有害（−2.5 pp）。两条 contradiction：cross-paper manifest vs outcome-RL；vs-thesis 主路径 model relabel vs harness edit。无 taxonomy 扩展（归入既有 off-axis harness 自演化桶）。 |
