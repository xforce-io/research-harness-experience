# Experience 闭环：Research Report

> **Version:** v21 (23 papers)
> **Last Updated:** 2026-08-23
> **Papers:** [01](notes/active/01_signals_trajectory_triage.md), [02](notes/active/02_agenther_hindsight_relabeling.md), [03](notes/active/03_tsr_trajectory_search_rollouts.md), [04](notes/active/04_agenttrace_structured_logging.md), [05](notes/active/05_breaking_observability_tax.md), [06](notes/active/06_agentseer_agentic_vulnerabilities.md), [07](notes/active/07_agent_as_a_judge.md), [08](notes/active/08_tide_trace_diagnostics.md), [09](notes/active/09_trajectory_guard_a_lightweight_sequence_aware.md), [10](notes/active/10_policy_invisible_violations_in_llm_based.md), [11](notes/active/11_near_miss_latent_policy_failure_detection.md), [12](notes/active/12_agentic_harness_engineering_observability_driven_automatic.md), [13](notes/active/13_where_llm_agents_fail_and_how.md), [14](notes/active/14_autodata.md), [15](notes/active/15_aevo_harnessing_agentic_evolution.md), [16](notes/active/16_when_agents_go_astray_course_correcting.md), [17](notes/active/17_masprism_lightweight_failure_attribution_for_multi.md), [18](notes/active/18_agentracer_who_is_inducing_failure_in.md), [19](notes/active/19_harness_r1_learning_to_edit_executable.md), [20](notes/active/20_ai4ai_at_test_time_strong_to.md), [21](notes/active/21_rethinking_the_evaluation_of_harness_evolution.md), [22](notes/active/22_sample_efficient_learning_from_agent_experience.md), [23](notes/active/23_trace_turn_level_reward_assignment_via.md)
> **Thesis:** [.researcher/thesis.md](.researcher/thesis.md)

---

本报告是 `.researcher/thesis.md` 的证据装置。CHARTER 三词是支柱边界；章节按 Design Context 的 G1–G5 展开。论文以证据身份出现，一篇可在多节出现。旧 L0–L3 只覆盖更新 A 的层间消费；harness 编辑是并列的更新 B，不是 off-axis。

## 开篇定位

Silver & Sutton 把改进介质换成 agent 与环境的交互流；姚顺雨把下半场瓶颈换成任务与评价。闭环内部是：L0 schema → L1 分诊 →（可选归因）后**先分流**，再走更新 A（改权重）、更新 B（改 harness）、两者都走、或非更新（in-flight）。评价横切 A 与 B，不是第三段管子。判断尽量机制化，而不是逐轨迹 LLM-judge。

## G1 捕获：schema 是门槛，前线用轻量信号

**文献现状。** L0：AgentTrace [4] 给出 operational + cognitive + contextual 三类 surface；thesis 还要用户话语与系统资源状态，否则 L1 的 Interaction/Environment 信号无法计算。沉默门控被反复坐实——Sentinel [10] Coverage 里 scope 缺失让 recall 100%→40%；Near-Miss [11] 的 history search 依赖每次 tool call 的 name+args+return，缺字段方法即失效；AgentDebug [13] 强制四模块 tag 的 Modular rollout 比 ReAct 高 +12 pp，schema 在 detector 介入前就拿走 31% 相对增益。[12] 的 7 类组件文件把同一约束延伸到 harness mutation：[19] 的 failure packet / 四 hook 同样要求任务约束与动作-观测摘录在场。

L1：Signals [1] 在 τ-bench 上以非语义规则达 82% informativeness、~1.5× 采样，且明确 signals 不是 quality scores、不开药方。旁证同向但方法分叉——AgentSeer [6] 图拓扑、Trajectory Guard [9] 序列小代理（F1 ~0.92，宣称比 LLM-judge 快 17–27×）、Sentinel [10] 声明式 KG 不变量（acc/F1 0.93）、Near-Miss [11] 成功轨迹上的 guard-code oracle。SWE-PRM [16] 的低效 taxonomy 与 MASPrism [17] 的 prefill 内部信号可作捕获门控，而不是只做事后评判。

归因（可选，不是分诊也不是评价）：MASPrism [17] 用 token-NLL 找 symptom、step-attention 找 source，两次 prefill、0 output token、6.69× 延迟降（2.66s vs A2P 17.82s），Who&When-HC Top-1 27.59%、TRAIL-GAIA Loc.Acc 0.591。AgenTracer [18] 同任务同基准走相反实现：训练 8B reasoner，agent-level 最高 +18.18%，但「轻量」只在推理端、数据工厂依赖 oracle G + 可重跑环境，输出仍是 free-form `⟨think⟩`。同基准：短 trace AG 上 [18] 37.30 略优 [17] 36.51，长 trace HC 上 [17] 27.59 明显优于 [18] 20.68。

成功轨迹：thesis 主张约 2/3「任务完成」仍含隐性摩擦。Near-Miss [11] 在 τ²-Airlines 上检出 8.6%–17.3% mutating 轨迹的 latent failure，补上 Sentinel [10] 在线 block-only 不覆盖的「outcome=correct 但 process=non-compliant」。[16] 的冗余探索/动作循环/解出不终止是同一现象的低效切面，但它只在线纠偏、不沉淀。

观测成本：Breaking the Observability Tax [5] 主张信号触发的 sentinel 升档，而非均匀下采样；[10] 的 O(|M|) 反事实模拟给动作时低成本判定下界。

判断纪律：规则可枚举处机制化——[11] 确定性 Python guard，[10] policy-in-prompt 对照下违规仅 95.3%→40.7%、跨模型 25%–85% 不一致。[17] 是中间态：LLM 内部量进确定性聚合，无 free-form，但建在有争议的 attention-as-explanation 上。[16] 给出结构化 vs 自由式最干净对照：taxonomy-guided PRM_D +10.6 pp 且步数微降，优于 free-form PRM_S（+5.8、步数 38.6→51.5）也优于替策略选动作的 PRM_DR（+4.8）。[7][13][16] 都展现换弱模型即崩（[13] All-Correct 32%→6%；[16] 开源 PRM 全部 ≤ base、最差 −20.4 pp）。[13] 单条 ALFWorld 仅 detection 即 40–60 次 GPT-4.1。[18] 证明「不需要 frontier judge」，不证明「应当机制化」。

**残留张力。** Signals 的 82% 未在非 τ-bench 复现（可证伪点「流」）。[9] 学习型小代理 vs [10][11] 声明型 oracle 的选边仍在。尚无最小充分 schema 字段集。[5] 仅摘要级，旧「>5× 降本」不可引用为已证。[17] 只在全失败基准评测，HC Top-1 <30%，不是端到端 triage。

**对我们系统的含义。** schema 先行，再选 L1/更新方法。前线轻量信号；LLM-judge 只留分诊后小子集。能枚举规则处全机制化；必须用 LLM 处走 [16] PRM_D 式 yes/no 树，确定性止于判定、不替策略选动作。归因接在 L1 之后。成功轨迹的隐性摩擦是给更新 A 的原料，不是单独一章进化。

## G2 更新 A：能力或偏好错则改权重

**文献现状。** 场景是能力/偏好错：要内化、要跨环境迁移、有可配对正负样本。AgentHER [2] 把失败重打标成 DPO/RLHF；TSR [3] 是训练期镜像（选已有 rollout，不造题）。Near-Miss [11] 的「绕过 vs 合规」对是可执行的 DPO 原料，论文自己没接到训练。Experience Distillation [22] 表明已收集经验上直接 SFT 几乎恢复不了 ICL 增益——监督来源比「有数据」更关键。TRACE [23] 用 turn-level TD 混合 outcome GRPO，在搜索域打过纯轨迹 advantage；SWE-bench 类等预算对照仍缺。过程监督若出现，须能对照 outcome-only；[23] 与 thesis「显式过程监督」仍有 scope 差（信号来自 gold 参考 log-prob，不是 PRM 标签）。

**残留张力。** hindsight 后的下游 win rate **未测**（可证伪点「更新 A」）。[12] 外置 long-term memory +5.6 pp、无需 SFT，是对「必须改权重」的旁证压力，不是否证——那是更新 B 的证据。

**对我们系统的含义。** 走 A 的条件：失败可配对纠正，或成功轨迹上有隐性摩擦对。不要 raw dump 当 SFT。[16] 的低效 taxonomy 可给 [2] 当更细的 stage-1 词表，但 [16] 自己不沉淀，沉淀才算 A。

## G3 更新 B：基板或 runtime 错则改 harness

**文献现状。** 场景是基板/runtime 错：权重冻住、要可回滚、一条 patch 能泛化、失败能定位到组件或 hook。Harness-R1 [19] 把编辑形式化为可训练 9B engineer：failure packet → 四 hook 可执行 patch → 冻结 target 重跑 Δ reward → GRPO。vanilla 9B 44.3%→53.6%（+9.3 pp），超最强 frontier prompted editor（GLM-5.2 48.8%）；outcome-trained 比 supervised-only 高 7.1 pp；跨 20 未见目标平均 +7.06 pp；仅 10 条失败生成的 patch 泛化到 1,270 held-out 任务 +8.9±1.5 pp。AI4AI [20] 在冻结弱模型上用强 builder 编译推理时 harness，宏平均 0.488→0.763。AHE [12] 改文件级 substrate，component ablation 里 memory 外置 +5.6 pp、单换 system prompt −2.3 pp。AEVO [15]、Autodata [14] 同族但 proposer 参数不更新；[15] 有 evaluator 只读隔离（去掉会 reward-hacking），[14] val pass 12.8%→42.4% 且无 per-edit 预言。

**残留张力。** [19] 无 [12] 式 predicted_fixes/risk_tasks 合同，reward 绑同批任务（transductive），GPU-hour 未披露。[20] 头条 best-of 会夸大代表性。B 的表观涨点必须过 G5： [21] 同预算下 evolved harness held-out 平均仅 +0.6。飞轮初期成本高（AEVO ~3×，[19] K=8×全批重跑未量化）。

**对我们系统的含义。** 走 B 的条件：失败能落到组件/hook，或权重根本不能动。接受信号尽量 outcome-grounded（[19]），部署仍要 falsifiable 合同（[12]）；[15] 的 evaluator 隔离必要不充分。用 [1] 对失败子集做前置 triage，再喂 [19] engineer，摊薄重跑——论文未做此门控。

## G4 双轨分流：A 与 B 并列，可叠，先分流再更新

**文献现状。** A 回答「策略会不会这件事」，B 回答「runtime 让不让它做成这件事」。层间接口已有范例：Signals 筛出的高价值失败是 AgentHER 的原料 [1][2]；同一筛后子集也可喂 [19] 的 failure packet。Near-Miss [11] 同时给 A 提供 DPO 对。Harness-R1 [19] 与 AgentHER [2] **并列**：仅 B 即可 +9.3 pp，target SFT 后 engineer 仍 +5.0 pp（59.2%→64.2%）——co-evolve 是互补，不是替代。没有一篇端到端跑通「分诊 → 分流 → A 和/或 B → 评价门」。

分诊后四路（thesis G4）：

1. **失败 + 可配对纠正** → A（[2] + [11] 绕过 vs 合规）。
2. **系统性、可定位到组件/hook** → B（[19] 四 hook，[12] 文件级 substrate）。
3. **成功但有隐性摩擦** → 优先作 A 的原料（[11]）；不是 B 的主场。
4. **单次可救、不该沉淀** → **非更新**。SWE-PRM [16]、AgentDebug [13] 在执行中途或单次 re-rollout 纠偏，不改权重也不落盘 harness。弱 PRM 会主动带偏（开源 PRM_S 把 32B 从 40.0% 拖到 19.6%、patch gen 92.4%→67.6%）。若仍要 in-flight，用 L1 信号（Loop/Stagnation）做稀疏门控，替代 [16] 的固定 n=5。[16] 的 +10.6 pp 全来自 Claude 监督弱 32B，而 Claude 单独做 policy 即 66.6%——未隔离「过程奖励」与「强模型泄漏」。

**残留张力。** 整环有效仍是假设。可证伪点「更新 A」的 win rate 与「更新 B」的同预算 held-out 都未在同一实验里对照第三臂。in-flight 不是第三条进化。

**对我们系统的含义。** 先分流再动手。同一条轨迹可以既出偏好对又出 harness patch。禁止把 [16][13] 的端到端 Δ 写成更新成功。禁止再把 B 叫做旁路或 off-axis。

## G5 评价是门：两条更新共用同一套纪律

**文献现状。** Rethinking Harness Evolution Eval [21] 把 evolution 与 parallel/sequential sampling 锁进同一预算，held-out 上 evolved harness 平均仅 +0.6，表观收益常可被多采样解释。[20] 的 holdout 迁移与 [21] 的 disjoint 测量互补，但任务族不同，不能互否。这扇门卡住 B，同样卡住 A 的「训练涨点」。

self-audit 两轴：(i) 推理期 falsifiable 合同——AHE [12] fix-prediction 5× random、regression-prediction 2× random；AEVO [15] 有隔离无 per-edit 预言；Autodata [14] 两者均弱。(ii) 训练期 outcome 接地——[19] 用重跑 Δ reward，9B 超 397B 级 prompted editor，但无 [12] 式合同。AgentDebug [13]、SWE-PRM [16]、AgenTracer [18] 落在两轴最差端，只报端到端 Δ（+26% relative / +10.6 pp / +4.8∼14.2%），不度量反馈命中率。TIDE [8] 是事后诊断交付人类，[16] 自称优于 post-mortem，却没度量中途反馈是否对。[18] 与 [19] 独立坐实固定 Self-Refine 有害（−4.9%/−5.5%；平均 −2.5 pp）。Agent-as-a-Judge [7] 是重量级语义终评样板，作对照组而非生产前门。

**残留张力。** 下一代通用模型不改 env 是否压过 harness **与** 权重侧改进——未测。人类偏好 vs 接地 outcome 的成本——未测（可证伪点「评价」）。[19] 的 held-out 泛化与 [21] 的「进化≈多样本」尚未在同一协议对打。

**对我们系统的含义。** 没有同预算采样对照和 held-out，就不把 A 或 B 的涨点写成改进。生产 auto-fix 两轴过线。Self-Refine 不当默认策略。

## 可证伪点追踪

- **流**：轻量信号在非 τ-bench 上 >70% informativeness，或未分诊端到端 RL 打赢闭环。状态：**未测**。唯一正面数字仍是 [1] 的 τ-bench 82%，未独立复现。[9][10][11][17] 是旁证、指标不可折算；[18] 是训练化 judge，几乎不构成正面证据。待决：非 τ-bench 生产风格语料上跑非语义信号并报 informativeness。
- **更新 A**：L1 子集 hindsight 相对随机偏好有下游胜率；或 SWE-bench 类等预算过程监督优于 outcome-only。状态：**未测 / 张力**。[2][11] 给出管线，[23] 在搜索域支持密集信用，无目标 falsifier 上的 win rate。[22] 反对 raw dump SFT。待决：L1-triaged relabel vs 随机采样的同条件 DPO；过程监督对照须能对上 SWE-bench 类预算。
- **更新 B**：同预算 + held-out 分离后仅改 harness 仍有稳定增量（[21] 的 +0.6 不算）；或冻结 target 时系统性压过「只多样本」。状态：**张力**。[19][20][12] 显示不改权重能涨点，但 [21] 表明评价协议可把「进化」解释成采样。[19] 有 held-out 泛化，reward 仍 transductive。待决：同一预算框架下 harness-edit vs parallel/sequential sampling。
- **评价**：下一代通用模型不改 env/eval 即系统性压过 harness **与** 权重侧改进；或人类偏好持续更便宜更稳。状态：**张力 / 未测**。[21] 可翻转「进化是否有效」；[20] 冻结模型+harness 可超更强裸模型，支持「评价/系统重于 scale」，但无参数 scale-only 头对头。第二条未测。

旧四层栈口径下的 sentinel「>5× 降本」已退出 thesis 可证伪列表，降为 G1 残留（[5] 未深读，不能当已证）。

## 版本更新日志
| 版本 | 日期 | 新增论文 | 关键变化 |
|------|------|---------|---------|
| v21 | 2026-08-23 | — | 按 thesis Design Context 重脊骨：G1 捕获 / G2 更新 A / G3 更新 B / G4 分流 / G5 评价门。A/B 并列；in-flight 标非更新；废止 off-axis 桶。可证伪点改为四条。 |
| v20 | 2026-08-22 | [20] AI4AI · [21] Rethinking harness eval · [22] Experience distillation · [23] TRACE turn-credit | 报告骨架改为闭环三节（流 / 更新 / 评价）。[20][21] 进评价（及 [20] 的 harness 更新）；[22][23] 进更新。§4 按新三条可证伪点给 未测/张力 状态。 |
| v16 | 2026-06-02 | [16] SWE-PRM / When Agents go Astray | 报告首次创建（thesis-anchored 骨架）。[16] 引入"判断层 lifecycle = in-flight/online"新维度，进入 §6（结构化 > 自由式的最干净对照：PRM_D +10.6 严格优于 free-form PRM_S +5.8）与 §7（self-audit 谱最差端，与 [13] 同属 report-neither）；记录与 [11] 的 cross-paper contradiction、一条 taxonomy 扩展提案、一条 charter tension（trace 轴是否覆盖 in-flight course-correction）。 |
| v17 | 2026-06-02 | [17] MASPrism / Lightweight Failure Attribution | [17] 是 thesis "lightweight signal beats LLM-judge" 在 **failure attribution**（"哪一步坏"）任务上的高质量正例：prefill-only SLM 内部信号（token-NLL 找 symptom + step-attention 路由 source），两次 prefill / 0 output token / 6.69× 延迟降，与 AgentDebug [13] 同任务同基准（Who&When）方法论尖锐对立。进入 §2（轻量信号侧新增 attribution 维度证据）、§6（确定性谱**中间形态**——确定性聚合 over LLM-internal 信号、无 free-form，但 mechanism 建在有争议的 attention-as-explanation 上，是该路线代价边界的实例）、§8(a)（非 τ-bench 语料旁证，但指标维度不同且仅全失败基准）。提一条 taxonomy 扩展提案（within-trace attribution 作 triage/evaluation 之外的独立子任务）入 contradictions.md。无 vs-thesis 矛盾。 |
| v18 | 2026-06-03 | [18] AgenTracer / Who Is Inducing Failure | [18] 与 [17] 同任务同基准（Who&When）但**相反实现**：trained 8B tracer（Qwen3-8B + 多粒度 GRPO，数据靠 DeepSeek-R1 + counterfactual replay + fault injection 在沙盒造）vs [17] 零训练确定性聚合。**结构暧昧样本**：表面是"小模型胜 frontier judge"（agent-level +18.18%），但"轻量"只在推理端、且替代物仍是 free-form `⟨think⟩` 的随机 LLM judge。进入 §2（lightweight 路线内部 trained-judge vs frozen-signal 分叉 + inference-only 轻量的 caveat）、§6（同任务走相反方向——确定性取决于判定形式不取决于模型尺寸，[18] 是退一步）、§7（self-audit 谱最差端，与 [13][16] 同列：只报 +4.8∼14.2% 不审计 feedback 命中率）、§8(a)（非 τ-bench attribution 数据点，但作训练化 judge 对"非语义信号 >70% informativeness"几乎非正面证据）。无 vs-thesis 矛盾；提一条 taxonomy 扩展提案（强化 [17] 的 within-trace attribution 子桶 + learned-judge vs frozen-mechanistic 子轴）入 contradictions.md。 |
| v19 | 2026-08-05 | [19] Harness-R1 / Learning to Edit Executable Runtime Harnesses | [19] 把 harness 编辑形式化为**可训练 9B engineer + outcome GRPO**（冻结 target 重跑 Δ reward），与 [12]/[14]/[15] 的"proposer 不更新"分叉。进入 §1（桥的旁路：不改 target 权重也可 +9.3 pp）、§7（self-audit 谱拆两轴——训练期 outcome 接地 vs 推理期 falsifiable 合同；[19] 强于 [14]/[15] 但仍弱于 [12] 的 manifest）、§8(b)（model-side relabel 的必要性再受旁证压力）。与 [18] 独立复现 Self-Refine 有害（−2.5 pp）。两条 contradiction：cross-paper manifest vs outcome-RL；vs-thesis 主路径 model relabel vs harness edit。无 taxonomy 扩展（归入既有 off-axis harness 自演化桶）。 |
