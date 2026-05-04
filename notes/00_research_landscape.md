# 研究地图：Post-Deployment Agent 工程基建

> 本文件是所有论文之间关系的全局鸟瞰图，随着阅读推进持续更新。

## 拼图全景 (The Big Picture)

```
┌─────────────────────────────────────────────────────────────────────────┐
│                   Post-Deployment Agent Engineering Stack               │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  ┌─────────────────────────────────────────────────────────────────┐    │
│  │ Layer 0 ─ 基础设施层 (Observability Infrastructure)             │    │
│  │                                                                 │    │
│  │   [4] AgentTrace          [5] Breaking the Observability Tax    │    │
│  │   结构化日志 Schema          Sentinel Sampling / 拓扑感知采样    │    │
│  └──────────────────────────┬──────────────────────────────────────┘    │
│                             │ 产出：结构化轨迹流                        │
│                             ▼                                           │
│  ┌─────────────────────────────────────────────────────────────────┐    │
│  │ Layer 1 ─ 筛选分诊层 (Signal Triage)                            │    │
│  │                                                                 │    │
│  │   [1] Signals ★           [6] AgentSeer                         │    │
│  │   三层信号分类学             Action-Component Graph / 拓扑检测    │    │
│  │   (交互/执行/环境)           (非语义漏洞发现)                      │    │
│  └──────────────────────────┬──────────────────────────────────────┘    │
│                             │ 产出：高价值 / 高风险轨迹子集              │
│                             ▼                                           │
│  ┌─────────────────────────────────────────────────────────────────┐    │
│  │ Layer 2 ─ 数据转化层 (Data Reconstruction / Relabeling)         │    │
│  │                                                                 │    │
│  │   [2] AgentHER             [3] TSR                              │    │
│  │   后见之明重打标             高价值 Rollout 动态筛选              │    │
│  │   失败→DPO/RLHF 数据        训练期噪声轨迹剔除                   │    │
│  └──────────────────────────┬──────────────────────────────────────┘    │
│                             │ 产出：清洗后的偏好数据                     │
│                             ▼                                           │
│  ┌─────────────────────────────────────────────────────────────────┐    │
│  │ Layer 3 ─ 模型迭代层 (Model Alignment & Iteration)              │    │
│  │                                                                 │    │
│  │   DPO / RLHF Fine-tuning                                       │    │
│  │   私有模型对齐 → 部署 → 产生新轨迹 → 回到 Layer 0               │    │
│  └─────────────────────────────────────────────────────────────────┘    │
│                                                                         │
│  ┌─────────────────────────────────────────────────────────────────┐    │
│  │ 横切面 ─ 评估范式 (Evaluation Paradigm)                          │    │
│  │                                                                 │    │
│  │   [7] Agent-as-a-Judge     [8] TIDE / TRACE                     │    │
│  │   主流 Agent 评估对照组      轨迹级诊断理论                       │    │
│  └─────────────────────────────────────────────────────────────────┘    │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

## 论文间核心关系

| 上游 | 下游 | 关系 |
|------|------|------|
| Signals [1] | AgentHER [2] | Signals 筛出高价值失败轨迹 → AgentHER 将其重写为训练数据 |
| Signals [1] | AgentSeer [6] | 都主张通过行为模式（非语义）检测 Agent 缺陷 |
| AgentTrace [4] | Signals [1] | AgentTrace 提供结构化日志基座 → Signals 在其上运行探针 |
| Breaking Obs. Tax [5] | Signals [1] | 动态采样 + 阈值触发 → 降低 Signals 探针的运行成本 |
| TSR [3] | AgentHER [2] | TSR 在训练期预筛轨迹分支 → AgentHER 在部署后重标轨迹 |
| Agent-as-a-Judge [7] | Signals [1] | 对照组：Judge 是"重量级语义评估"，Signals 是"轻量级信号分流" |
| Trajectory Guard [9] | Signals [1] | 同处 L1 分诊层的方法论分叉：[9] 用 Siamese RNN 黑箱小代理输出二元 anomaly score；[1] 坚持可解释规则 + 多类信号。F1 0.92 与 informativeness 82% 不同指标，不可直接比较 [9: §Methodology / Relations] |
| Trajectory Guard [9] | Agent-as-a-Judge [7] | [9] 显式以 Phi-3-mini / GPT-4o-Mini / Gemini Flash / Deepseek 作 Judge 基线，主张以 17–27× 推理速度优势替代之；属"评估 vs 采样"分歧的有监督小模型变体 [9: Table 3, Table 4] |
| Trajectory Guard [9] | AgentSeer [6] | 拒绝纯语义检测的两种结构视图：[9] 走序列（GRU 时序），[6] 走图（action-component 拓扑），可叠加 [9: Relations] |
| AgentTrace [4] | Trajectory Guard [9] | [9] 输入 AgentAlign 风格的结构化 tool call 序列，隐式假设 AgentTrace 类 Operational + Contextual 日志已就位（论文未引用，按数据格式反推）[9: §Method] |
| Sentinel / PhantomPolicy [10] | Trajectory Guard [9] | L1 邻近层的范式分叉：[10] 用声明式不变量 + 反事实 KG 模拟（O(\|M\|)，acc 0.93 / F1 0.93），显式拒绝把世界知识压入模型权重；[9] 用 Siamese RNN 黑箱小代理（F1 0.92）。指标接近但**学习型 vs 声明型**立场对立，KWeaver 设计中需立场选择 [10: §6.1, Theorem 1, Relations] |
| Sentinel [10] | Signals [1] | 同属"非 LLM、规则化、动作时"判定家族；目标不同：[1] 做轨迹采样信息量、[10] 做合规阻断。[10] 的 I1 ActiveRecipient / I2 ContextBoundary / I7 Liveness 可作为 Signals 三类信号中 Environment 类的具体不变量样板 [10: §4.6, Relations] |
| Sentinel [10] | AgentSeer [6] | 都把 agent 行为视为图变更并在图上做结构判定，但**图的语义不同**：AgentSeer 图 = 轨迹结构（action component / causal link）；Sentinel 图 = 组织世界图（entity / relation / flow）。前者抓 agent 内部行为异常，后者抓 agent 行为对外部世界的合规态扰动；KWeaver L1 实际两图都需要 [10: Relations] |
| AgentTrace [4] | Sentinel [10] | Sentinel 的 session context S（source_scope / accumulated data_sources / project context）等价于 AgentTrace Operational + Contextual + User-Interaction 三类 surface 在动作判定时的实时摘要；Coverage 实验中 scope 标注缺失即让 recall 100→40%，是 thesis"L0 schema 是 silent gating constraint"的具体例证 [10: §6.3, Table 10] |
| Sentinel [10] | Agent-as-a-Judge [7] | [10] 用 policy-in-prompt（在 system prompt 中注入高层规则但仍隐藏实体级元数据）作为 LLM-judge / prompt-level 防御的代理对照，实测违规率仅从 95.3% 降到 40.7% 且跨模型 25%–85% 不一致，论证"决定性事实未进上下文 → 任何模型类判定结构性失败"；与 thesis"sampling informativeness ≠ judgment accuracy"是同一观察的 enforcement 侧 [10: §5.2 Table 6, F3] |
| Sentinel [10] | Near-Miss [11] | 同一评估光谱的两端：[10] 在动作时反事实模拟拦截**显式**违规（block-only），明确不覆盖"outcome=correct 但 process=non-compliant"子集；[11] 在轨迹后用 guard code 重放检测**侥幸**绕过（latent failure），正好补上 [10] 不覆盖的子集。两者共享"executable rules > LLM-judge"哲学，在 KWeaver L1 设计中可联合部署：[10] 在线拦截、[11] 离线评估剩余成功流 [11: §1, §3.1; 10: §6.2] |
| Agent-as-a-Judge [7] | Near-Miss [11] | 实质性挑战 LLM-judge 范式：[11] 把判断委托给确定性 Python（ToolGuard guard code）+ 一次结构化历史搜索，论证"判断准确性也可以从语义模型转移到结构化代码"。在 thesis"sampling informativeness ≠ judgment accuracy"延长线上，本文进一步指出**判断准确性本身**也未必需要语义模型 [11: §1, §5; Table 1 P=R=1.00] |
| Signals [1] | Near-Miss [11] | [11] 是 Signals "Execution 类信号"的领域特化版本：把通用 phrase pattern 替换为"guard-code 引出的必读 RO 集合"。共享"非语义、规则化、轨迹级"判定家族；不同点：Signals 输出多类信号标签用于采样信息量，[11] 输出二元 latent failure 标签用于评估盲点 [11: §3.2, §4; 1: §2.1] |
| AgentHER [2] | Near-Miss [11] | [11] 检出的 latent failure 自带可纠正模式："原轨迹（绕过策略）"vs"先调 RO 再调 MTC（合规版）"——可直接形成 DPO 对的负正样本。[11] 论文未提此用法，但其方法输出物（latent failure 标注 + 漏读 RO 列表）即为 L2 hindsight relabel 可消费的结构化提示；这是该论文对 KWeaver 主线最直接的贡献 [11: §3.3 listing, Relations] |
| AgentTrace [4] | Near-Miss [11] | [11] 的 history search 阶段强依赖于 trace schema 中**完整保留每次 tool call 的 name + args + return value**——若日志只存 tool name 而省略 args/return，等价 RO 跨 schema 字段匹配即失效。该论文为 thesis"L0 schema 是 silent gating constraint"提供又一具体例证：缺字段直接让方法不可用 [11: Appendix A.1; 4: §3 Operational surface] |
| Trajectory Guard [9] | Near-Miss [11] | "learned small surrogate vs declarative oracle"的明确分叉：[9] 把世界知识压进 Siamese RNN 权重输出 anomaly score；[11] 要求世界知识住在 guard code 与历史结构中并给出"缺少哪个 RO"的可解释解释。F1 数字落点接近（[9] ~0.92，[11] code-gen 路径 P=R=1.00 但单标注者 ground truth 偏弱），范式对立 [11: §4.2; 9: §Methodology] |
| Near-Miss [11] | AHE [12] | "判定该不该用 LLM"的范式对立：[11] 主张确定性 guard code + 结构化历史搜索代替 LLM-judge；[12] Agent Debugger 把 trajectory 解读完全交给 LLM-agent 在文件系统上自由探索，三层 observability（Component / Experience / Decision）全部依赖 LLM 调用。AHE +7.3 pp 但代价 32 小时 + 数十亿 token；Near-Miss code-gen 路径 P=R=1.00（单标注者 ground truth 偏弱）。立场对立——同条 contradiction 已落 contradictions.md [12: §3.2, §5.2; 11: §1, §5] |
| Agent-as-a-Judge [7] | AHE [12] | [12] 的 Agent Debugger 与 [7] 同源——都用 LLM-agent 读 trajectory 做语义判定，区别仅在输出消费方：[7] 给 outcome-evaluator，[12] 给 evolve agent。cost / 可解释性 / 可复现性问题完全继承自 [7]，论文未做"raw cleaned trace 直接喂 evolve agent"的消融。在 thesis"LLM-judging 留给 triaged 后的小子集"判断方向上，[12] 把 LLM-judging 用在**全部 trajectory**（每任务 k=2 traces 全跑）——thesis 明确反对的部署模式 [12: §3.2; 7: §4] |
| AgentTrace [4] | AHE [12] | [12] 的 7 类组件文件 + git history + change manifest 实质上是为"harness mutation"设计的一套 trace schema，与 [4] 的 deployment-time trace schema 同构。在 thesis"L0 schema 是 silent gating constraint"语境下，[12] 提供新例证：**没有解耦的文件级 harness substrate，evolve agent 没法定位每个失败模式的归属组件**——schema 必要性论据从 deployment-time 扩展到 training-loop-time [12: §3.1; 4: §2] |
| Signals [1] | AHE [12] | 同享"raw trace 不可消费、分层结构化才能可消费"的工程直觉，但 cost 立场对立：[1] rule-based detector（轻量），[12] Agent Debugger（LLM-driven 重组件 / 每条 trace 都进 LLM）。生命周期不同：[1] deployment-time L1 triage，[12] training-loop-time harness self-improvement。AHE 在 thesis"lightweight signal beats LLM-judge"语境下是反例的具体实现，且未对 Agent Debugger 相对 raw trace 做消融 [12: §3.2; 1: §2.1] |
| AHE [12] | AgentHER [2] | [12] 的 component ablation（system prompt 单换入 −2.3 pp，long-term memory 单换入 +5.6 pp，§4.4.1 Table 3）拷问 [2] 路线：把"经验"显式落到外部 memory 文件比落在 prompt 里效果更稳定，且不需要 SFT 训练。给 hindsight relabel + SFT 的"必要性"提出工程对照点——若 externalize-to-artifacts 就能 +5.6 pp，model-side relabeling 的成本-收益比需重估 [12: §4.4.1; 2: §3] |
| Agent-as-a-Judge [7] | AgentDebug [13] | [13] 是 [7] 在 root-cause 归因任务上的具体实现：detector + critical-finder 都是 LLM 调用（GPT-4.1 T=0），输出粒度比 [7] 更细（step×module×17 类 error type）。GPT-4.1 上 All-Correct 32%，detector 换成 Llama-3.3-70B 后跌到 6%、GPT-4o-mini 跌到 2%（§5.1 Figure 7b）——把 [7] "judging accuracy varies across base models" 论点在 root-cause 任务上再次坐实。在 thesis"LLM-judging 留给 triaged 后小子集"判断方向上，[13] 把 LLM-judging 用在**全部失败轨迹的全部步全部模块**，是 thesis 反对的部署模式 [13: §3, §5.1] |
| Near-Miss [11] | AgentDebug [13] | "trajectory-level 判定该不该用 LLM"的范式对立（与 11↔12 一脉同源）：[11] 用 ToolGuard guard code + 结构化历史搜索（确定性、可审、机制可移植）；[13] 用 LLM detector + LLM critical-finder + LLM feedback writer（强依赖 GPT-4.1，换 base 即崩）。同任务（trace 后置判定）反向结论：[11] code-gen 路径 P=R=1.00（单标注者 ground truth），[13] All-Correct 24.3% 但被 κ=0.55 标注一致性天花板限制（论文叙事为 "substantial" 实为 Landis & Koch "moderate"）。同条 contradiction 已落 contradictions.md [13: §2.2, §3.2; 11: §1, §5] |
| AHE [12] | AgentDebug [13] | 同源 LLM-as-judge 自动修复方法论的两个时间尺度：[12] training-loop-time 改 harness 文件持久化，[13] inference-time 改单 trajectory re-rollout 临时化。两者都把"失败 trace → LLM 解读 → 输出修复指令"做成自动化闭环。**关键差别**：[12] 显式量化 evolve agent fix/regression 预测准确率（5×/2× random）作为 self-audit；[13] 完全没做这种 self-audit——corrective_guidance 是否真 actionable / re-rollout 失败是因为 feedback 错还是 base model 弱，论文从未拆解。意味 [13] "+26% relative" 包含大量盲试成分，KWeaver 做 inference-time auto-fix 应当借鉴 [12] manifest 模式而非 [13] free-form feedback [13: §4.2; 12: §4.4.2] |
| Signals [1] | AgentDebug [13] | 同栈不同层可串联（Signals → AgentDebug 把 detector cost 摊薄到小子集），但 cost 范式对立：[1] rule-based detector 轻量；[13] 单条 ALFWorld trace（中位 ~10–15 步）仅 detection 阶段就 40–60 次 GPT-4.1 调用 + critical finder + 最多 5 次 re-rollout，论文从未拆 detection / critical-finder / re-rollout 三段 token 占比。在 thesis "lightweight signals beat LLM-judge for front-line filtering" 论证方向上，[13] 是 thesis 反例的具体实现 [13: §3, §4] |
| AgentTrace [4] | AgentDebug [13] | "L0 schema 是 silent gating constraint" 的新极端例证：[13] detector 之所以能逐 step×模块 判错，**前提**是 agent 已被强制在 `<memory>` `<reflection>` `<plan>` `<action>` tag 内输出（Appendix A.6 三个 environment 的 rollout prompts 强约束）。Rollout strategy ablation（Figure 7c）：仅强制四模块 tag 化（Modular 0.38）就比 ReAct 一体输出（0.26）高 +12 pp 成功率——schema 缺失直接拿走 31% 相对增益，detector / critical-finder 都还没介入 [13: §5.1, Appendix A.6; 4: §3] |
| AgentHER [2] | AgentDebug [13] | 同向 taxonomy 工作的两次切法：[2] 给 6 类失败（Incomplete / Constraint_Violation / Wrong_Result / Tool_Error / Hallucination / Off_Topic），[13] 给 5 模块 × 17 子类（Memory / Reflection / Planning / Action / System，含 hallucination / over-simplification / progress misassessment / constraint ignorance 等细粒度子类）。下游消费方完全不同：[2] taxonomy 做训练数据筛选与 DPO 偏好对（offline），[13] taxonomy 做 inference-time re-rollout 反馈（online）。[13] 的细粒度 critical-error 标注方法可反向供给 [2] 路线作为更细的 stage-1 detector——任一论文均未写出的组合点 [13: §2.1, §3; 2: §3] |

## 与 KWeaver TraceAI 的映射

| 研究层 | KWeaver 落地组件 | 当前状态 |
|--------|-----------------|---------|
| Layer 0 基础设施 | TraceAI Collector / Schema | 已有基础，需参考 AgentTrace 补全 Schema |
| Layer 1 信号分诊 | TraceAI Triage Center (待建) | 需实现正则探针 + 状态机探针；[9] Trajectory Guard 作"学习型小代理"备选路线（黑箱、二元输出，与可解释规则路线互斥）；[10] Sentinel 提供"声明式 KG 不变量"样板（环境类信号），与 [9] 立场对立；[11] Near-Miss 提供"成功轨迹隐藏 friction"的可计算指标（guard-code-as-oracle 反查必读 RO），与 [10] 形成 in-line / post-hoc 互补 |
| Layer 2 数据转化 | Data Flywheel Pipeline (待建) | 需参考 AgentHER 设计重标管道 |
| Layer 3 模型迭代 | 私有模型微调 (未来) | 依赖 Layer 2 产出 |

## 阅读优先级建议

1. 🔴 **P0 — 必读，直接影响架构设计**
   - [1] Signals ✅ 已读
   - [2] AgentHER — 数据飞轮的核心方法论
   - [4] AgentTrace — 日志 Schema 的工程蓝图

2. 🟡 **P1 — 重要，补全设计视角**
   - [5] Breaking the Observability Tax — 成本控制策略
   - [6] AgentSeer — 非语义检测验证
   - [9] Trajectory Guard ✅ 已读 — L1 分诊层"学习型小代理"对照组，方法论与 [1] 互斥
   - [10] Sentinel / PhantomPolicy ✅ 已读 — 动作时声明式 KG 不变量验证；L1 邻居（enforcement 侧），与 [9] 学习型路线分叉，并为 thesis"schema 是 silent gating constraint"提供 Coverage 退化实证
   - [11] Near-Miss / Latent Policy Failure ✅ 已读 — 评估范式横切 + L1 后置补盲：guard-code-as-oracle 反查"成功轨迹中应读未读的 RO"，τ²-Airlines 上 8.6%–17.3% mutating 轨迹被检出；与 [10] 形成 in-line/post-hoc 互补，并为"成功轨迹的 hindsight relabel"提供可执行入口（[2] AgentHER 的天然原料）

3. 🟢 **P2 — 参考，拓宽理论视野**
   - [3] TSR — 训练期轨迹优化（与部署后场景互补）
   - [7] Agent-as-a-Judge — 评估范式对照
   - [8] TIDE / TRACE — 理论延伸
   - [12] Agentic Harness Engineering ✅ 已读 — **off-axis（harness 自演化，不在四层栈内）**；其 Agent Debugger 是 [7] LLM-judge 路线在 training-loop-time 的具体落地，与 thesis"front-line filtering 用轻量信号"立场对立；但其**falsifiable change manifest**（edit 自带 predicted_fixes / risk_tasks，下一轮 task delta 自动判决并文件级回滚）作为 detector versioning / 信号阈值演化的治理样板可借鉴；fix-prediction 5× random 但 regression-prediction 仅 2× random（§4.4.2）的非对称是 KWeaver Triage Agent 自我归因可靠性的直接量化警示（详见 contradictions.md 中的 taxonomy 提议）
   - [13] AgentDebug / Where LLM Agents Fail ✅ 已读 — **off-axis（root-cause 归因的 LLM-judge 路线，inference-time）**：5 模块 × 17 类 error taxonomy + AgentErrorBench（200 标注轨迹，Cohen's κ=0.55 论文叙事 "substantial" 实为 Landis & Koch "moderate"）+ AgentDebug 三阶段 LLM-cascade（fine-grained detector → critical-error finder → re-rollout 循环 ≤5 次）；与 [12] AHE 同属 LLM-as-judge 自动修复家族但时间尺度不同（[12] training-loop 改 harness 持久化，[13] inference-time 改单 trajectory 临时化）；与 [11] Near-Miss 在"trajectory-level 判定该不该用 LLM"上立场对立——同条 contradiction 已落 contradictions.md。其 5 模块 × 17 类 taxonomy 可作 KWeaver Triage Agent 失败标签词表的对照来源；其 detector 强依赖 GPT-4.1（换 Llama-3.3-70B 后 All-Correct 从 32% 跌到 6%，§5.1 Figure 7b）警示 LLM-judge 路线的机制不可移植性；其 Modular rollout 0.38 vs ReAct 0.26 的 +12 pp（Figure 7c）则为"L0 schema 是 silent gating constraint"提供新极端例证
