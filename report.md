# KWeaver TraceAI：服务于 AI 工程飞轮的双轨 Trace 与 Triage Agent: Research Report

> **Version:** v2.5 (11 papers)
> **Last Updated:** 2026-05-04
> **Papers:** [01](notes/01_signals_trajectory_triage.md), [02](notes/02_agenther_hindsight_relabeling.md), [03](notes/03_tsr_trajectory_search_rollouts.md), [04](notes/04_agenttrace_structured_logging.md), [07](notes/07_agent_as_a_judge.md), [08](notes/08_tide_trace_diagnostics.md), [09](notes/09_trajectory_guard_a_lightweight_sequence_aware.md), [10](notes/10_policy_invisible_violations_in_llm_based.md), [11](notes/11_near_miss_latent_policy_failure_detection.md), [12](notes/12_agentic_harness_engineering_observability_driven_automatic.md), [13](notes/13_where_llm_agents_fail_and_how.md)
> **Thesis:** [.researcher/thesis.md](.researcher/thesis.md)

> **形式：** 内部技术报告 / Position Paper
> **作者：** xupeng
> **基于：**
> - 11 篇深读论文（Signals、AgentHER、TSR、AgentTrace、Agent-as-a-Judge、TIDE、Trajectory Guard、Sentinel、Near-Miss、AHE、AgentDebug）+ 2 篇待补（Breaking Obs Tax、AgentSeer）— 索引见 [`papers/README.md`](papers/README.md)
> - **KWeaver 项目资料**（脱敏后入 [`references/`](references/)）：纲领（[01](references/01_overcoming_ontology.md)）、路线图（[02](references/02_engineering_roadmap.md)）、产品概览（[03](references/03_kweaver_core_overview.md)）、技术深度（[04](references/04_kweaver_core_deep_analysis.md)）、Harness Engineering 位置论（[05](references/05_harness_engineering_position.md)）
> **目标读者：** KWeaver TraceAI / Decision Agent / Triage Agent 设计与工程团队
> **目标：** 把学术研究映射到 KWeaver 飞轮的具体差距，特别是为 **Triage Agent prototype 设计**提供可操作的输入

---

## 0. v1 → v2 的关键校准

> **v1 报告的根本错误**：把 KWeaver 简化为"agentic harness（推理侧）"，把 TraceAI 当作"观测层"。这是**严重低估了 KWeaver 的产品定位**。

| v1 的判断 | v2 的修正 |
|----------|---------|
| KWeaver = agentic harness（推理侧）| **KWeaver = Harness 运行时 + AI 工程飞轮**——飞轮才是护城河 |
| TraceAI 是观测层 | **TraceAI 双轨 Trace（调用链 + 证据链）是飞轮的物理底座**，是核心数据资产 |
| 数据飞轮 12+ 个月不要建 | **飞轮是 KWeaver 的核心产品形态**，从 day 1 就在建——只是消费者不是模型微调，是 BKN 演化和执行策略修正 |
| 不要建质量评分系统 | "不打分"立场依然正确，但要校准——**Triage Agent 不是 Judge，是飞轮反馈通道** |
| 短期目标是"Triage Center MVP" | **短期目标是 Triage Agent prototype**——这是 KWeaver 路线图明确的当前差距 |

**保留 v1 中仍然正确的部分**：5-surface schema 设计原则、"信号不是质量分"立场、Agent-as-Judge 的"Memory + Planning 模块有害"发现、TIDE 的 LR 是 Signals.Loop 理论化版本。

---

## 1. KWeaver 的真实战略坐标

### 1.1 三阶段演进

```
2020-2023        2024-2025          2026+
Prompt Eng.   →  Context Eng.    →  Harness Eng.
人-模型连接      Agent-模型连接      Agent-业务连接
```

KWeaver 在 **Harness Engineering** 阶段——通过编排、校验、追踪与审批闭环，把智能体能力沉淀为稳定、可控、可审计的企业执行系统。

### 1.2 飞轮闭环

```
                ┌──────────────────────────┐
                │      业务信号采集层         │
                │  （用户/Agent 真实执行）    │
                └────────────┬─────────────┘
                             │
                             ▼
       ┌─────────────────────────────────────┐
       │   双轨 TraceAI（调用链 + 证据链）     │ ← 飞轮的物理底座
       │   ─────────────────────────────     │
       │   • 调用链：执行拓扑（谁调谁、耗时）    │
       │   • 证据链：决策依据（推理、引用、guard 触发）│
       └─────────────┬───────────────────────┘
                     │
                     ▼
       ┌─────────────────────────────────────┐
       │  Triage Agent（异步、不参与实时决策）  │ ← 当前最大 gap
       │   输入：双轨 Trace + ISF 记录         │
       │   输出：BKN patch 建议 / 策略调优 /    │
       │         失败归因 / 人工 review 队列    │
       └─────────────┬───────────────────────┘
                     │
                     ▼
       ┌─────────────────────────────────────┐
       │   反哺：BKN 自演化 + Context 策略学习 │
       │   + Guardrails 自动生成             │
       └─────────────┬───────────────────────┘
                     │
                     ▼
       （下一轮执行使用更准的 BKN + 更优的 Context 策略）
```

### 1.3 Scaling vs 脚手架的判据

> 引自路线图设计稿："系统如果不能随着模型、数据、算力的增长自动变强，其长期价值必然被绕过。"

**KWeaver 自身组件的 Scaling 审视**：

| 组件 | 属性 | 是否飞轮 |
|------|------|---------|
| VEGA 数据虚拟化底座 | 基础设施 | "门票的门票"，不可或缺但非飞轮 |
| 人工编写的 BKN 文件 | 脚手架 | ❌（必须自动化） |
| Context Loader 硬编码规则 | 脚手架 | ❌（必须可学） |
| 静态 Guardrails / Risk 描述 | 脚手架 | ❌（必须自动生成） |
| **TraceAI 数据与平台** | **Scaling** | ✅ **核心数据资产** |
| **BKN 自构建 / 自演化循环** | **Scaling** | ✅ |
| **Triage Agent** | **Scaling** | ✅ **吃 Trace 产出反馈** |
| **Context Loader 策略学习** | **Scaling** | ✅ |
| **整体闭环飞轮** | **Scaling**（最硬一轴）| ✅ |

**对学术研究的判据**：每篇论文的方法是被 KWeaver 借鉴为脚手架（一次性设计）还是 Scaling 组件（自动产出 / 演化）？这决定了它进入飞轮的资格。

### 1.4 三层结构性壁垒

模型变强反而让飞轮转得更快，三层壁垒确保飞轮不被吞并：

- **数据壁垒**：企业私有执行数据（TraceAI 调用链/证据链/决策依据/guardrail 触发记录）通用模型无法获取
- **制度壁垒**：合规审计是法规硬约束，不是模型推理能力可替代的
- **时间壁垒**：历时性业务记忆是数据存量问题——不是模型能力问题

### 1.5 不可替代性的 4 个交叉点

> KWeaver 的不可替代性不在 Trace 平台本身（LangSmith 也有），不在知识建模本身（Palantir 也有），**而在四者交叉**：

```
语义层（BKN）  ×  双轨 Trace（调用链 + 证据链）
        ×  执行约束（ISF + Guardrails）
        ×  自动演化闭环（Triage Agent）
```

→ **TraceAI 团队的工作必须服务这四者交叉**，单独优化其中任一项都不构成护城河。

---

## 2. 当前差距（来自路线图设计稿）

> 飞轮的终局图景已经清晰，但今天离"转起来"还有明显距离。

| 飞轮环节 | 当前状态 | 主要差距 |
|---------|---------|---------|
| **TraceAI 数据采集** | 调用链基本完整，证据链已接入 | 调用链-证据链的**关联查询能力**待加强 |
| **BKN 构建** | kweaver-dip 已有 bkn-creator 初版，kweaver-sdk 支持从部分资源自动构建 | 自动构建**覆盖面 + 生成质量**待提升；从 Trace 反向矫正 BKN 的**闭环未打通** |
| **BKN 自演化** | 概念阶段 | 缺少从 Trace 信号到 BKN patch 的**自动闭环** |
| **Triage Agent** | **概念阶段，尚无 prototype** | 输入信号格式、输出动作定义、与 Decision Agent 的关系**均待设计** |
| **Context Loader 策略学习** | 硬编码规则 | 策略参数化、反馈信号采集机制**均未实装** |

→ **Triage Agent 是当前最具体、最 actionable 的 gap，也是学术研究最能直接服务的对象。**

---

## 3. 学术研究按"飞轮环节"重新组织

> v1 按 4-Layer Stack 组织（学术友好）；v2 按"服务飞轮的哪个环节"组织（决策友好）。

### 3.1 燃料采集：双轨 Trace 字段补全

**主要论文：AgentTrace（schema 命名学）+ Signals（信号需要的数据）**

KWeaver 双轨 Trace 当前已有：
- ✅ 调用链：方法调用拓扑、耗时、上下文传播路径
- ✅ 证据链：每步决策依据、推理、引用

**学术研究启示**——AgentTrace 的三 surface ∪ Signals 的三组 = 5 个独立 surface，KWeaver 应映射如下：

| Surface | 出处 | KWeaver 现状 | 对应 KWeaver 组件 |
|---------|------|-------------|-------------------|
| Operational（方法调用）| AgentTrace | ✅ 调用链已覆盖 | Decision Agent / Execution Factory 调用栈 |
| Cognitive（plan/reasoning/reflection）| AgentTrace | 🟡 部分（混在 raw completion）| Decision Agent 的推理上下文 |
| Contextual（外部 I/O） | AgentTrace | ✅ 证据链已覆盖 | VEGA 引擎的数据访问记录 |
| **Interaction（用户↔Agent 对话）** | **Signals** | ❓ 取决于场景 | Decision Agent 的用户交互层（如有）|
| **Environment-as-resource（限流、context cap、外部 500）** | **Signals** | 🟡 部分（系统监控未与 trace 关联）| 系统监控 → 需要与 trace_id 关联 |

**具体改进建议**：
- [ ] **Cognitive surface 结构化**：在 Decision Agent 中要求 LLM 输出包含显式 `plan` / `reflection` / `thought` 字段，避免依赖 marker 解析（参考 AgentTrace §2.3 的 marker 脆弱性问题）
- [ ] **调用链 ↔ 证据链关联查询**（路线图明确的 gap）：建议用 OTel `trace_id + span_id` 嵌套作为关联主键
- [ ] **Environment surface 关联**：把限流、上游 500、context overflow 等系统级事件按 trace_id 写入 Trace，标记为 `environment` 类型——**关键**：这类事件与"agent 决策错误"必须严格区分（Signals 论文的 Environment.Exhaustion 立场）

### 3.2 燃料质量门控：什么 trace 值得反哺飞轮

**主要论文：Signals（信号分类法 + 实证 82% informativeness）**

> Signals 论文的核心实证：在 τ-bench 上信号采样达到 82% informativeness（vs 随机 54%），**且对成功轨迹仍有 66.7% 命中率**——后者是该论文最被低估也最有价值的发现。

**对 KWeaver 的启示**：

KWeaver 的飞轮反哺**不能是"看 trace 全部喂回 BKN"**——那是噪声，会污染本体。需要先做信号分诊。Signals 提供的 7 类信号在 KWeaver 场景的映射：

| Signals 信号 | KWeaver 场景对应 | 反哺价值 |
|-------------|----------------|---------|
| Interaction.Misalignment | 用户重述/澄清，与 Agent 输出语义偏移 | **高**：提示 BKN 概念定义不准 |
| Interaction.Stagnation | 多轮无进展 | **高**：提示 Context 策略需调优 |
| Interaction.Disengagement | 用户切人工或退出 | 中：可作为人工 review 入口 |
| Interaction.Satisfaction | 明确成功表达 | **高**：作为 exemplar，反向验证 BKN/路径 |
| Execution.Failure | 工具调用失败、no-op | 中：分情况——algorithm 错 vs 工具/数据错 |
| Execution.Loop | DPH 步骤循环、Agent 卡住 | **极高**：提示 BKN 行动定义有缺陷 |
| Environment.Exhaustion | 限流、context 溢出、500 | **诊断用，不进飞轮**——会引入虚假相关性 |

**重要立场（来自 Signals 论文）**：信号**不是质量分**——它们只标"这条 trace 值得 Triage Agent 处理"，不评对错、不开药方。**Triage Agent 的输入是带信号标签的 trace 流，不是分数排名**。

#### 3.2.1 成功轨迹中的"侥幸"信号（Near-Miss / Latent Failure）

> **核心补充（Near-Miss 论文 [11]）**：在 outcome=success 的子集上提供一个**确定性、非 LLM-judge** 的可计算 friction 信号——agent 跳过策略要求的前置检查、却因为客户事实恰好正确而到达正确终态。该信号是 thesis"成功轨迹的 2/3 仍含可学习 friction"在 mutating-tool 场景下的精确化实现。

| 维度 | Near-Miss 信号定位 |
|------|------------------|
| 触发条件 | trace 含 ≥1 mutating tool call（MTC）+ 终态匹配 gold |
| 检测机制 | 用 ToolGuard 自动生成的 `guard_<MTC>(args)` 作为 oracle，重放执行；guard 内引用的每个 read-only 工具都被解读为"该 MTC 的策略前提"；在 trace 历史中反向搜索是否存在等价 RO 调用 |
| 输出 | 二元标签 `latent_failure ∈ {TRUE, FALSE}` + 漏读的 RO 列表 + 对应的 guard 引用 |
| 实证表现 | τ²-Airlines 上含 MTC 子集 8.6%–17.3% latent failure 率（GPT-oss-120b 最低，GPT5-chat 最高）；code-gen + Claude-Sonnet4 实现路径 P=R=1.00（单标注者 ground truth，~400 sims）[11: §4.2–§4.3] |
| 工具级偏置 | `update_reservation_flights()` 最易被无前置检查直接调用；`get_flight_status()` 最常被绕过 [11: §4.4] |

**对 KWeaver 的两点直接含义**：

1. **Near-Miss 是 Signals.Execution 信号在 mutating-tool 场景的领域特化版本**——把通用 phrase pattern 换成"BKN.Action 的前置约束 → 必读 RO 集合"。在 KWeaver 中，BKN 已经为每个 Action 定义了前置约束（PreCondition），可以**反向自动生成 guard code**：扫 BKN.Action 的 PreCondition → 列出"满足该约束所需的最小 RO 集合"→ 在 trace 历史中检索是否被读取。这条路径**避开了 ToolGuard 论文中需要 LLM 离线生成 guard code 的成本**——KWeaver 的 BKN 已经携带了等价信息。

2. **Near-Miss 与 Sentinel [10] 形成 in-line / post-hoc 互补**：Sentinel 在动作时反事实拦截显式违规（block-only），明确不覆盖"outcome=correct 但 process=non-compliant"子集；Near-Miss 在轨迹后用 guard code 重放检测侥幸绕过。两者共享"executable rules > LLM-judge"哲学，可联合部署：Sentinel 在线、Near-Miss 离线。

**Triage Agent 输入 schema 扩充**（在 §4.2.1 yaml 上加一个 `latent_failures` 字段）：

```yaml
signals:
  ...
  latent_failures:                       # ← 新增（Near-Miss 信号）
    - mtc: "update_reservation_flights"  # 哪次 mutating call 漏检
      args: { ... }
      missing_ro_set:                    # 应读但未读的等价 RO 集合
        - "get_flight_status"
      guard_ref: "guard_update_reservation_flights:within_24_hours"
```

**重要边界（论文 §6 + 自批判）**：
- 该信号**只在 outcome=success 的子集上有意义**——失败轨迹的策略遵从度由 Sentinel / Signals.Execution.Failure 处理。
- 论文的"危害论"建立在 non-adversarial 假设上；KWeaver 在敌对场景（如对话中用户故意提供不实事实）需要把 Near-Miss 升级为实时拦截信号——这正好回到 Sentinel [10] 的范式。
- ToolGuard guard 自身的质量构成上界：guard 漏写一个前置约束就会产生 latent_failure 的 false negative；KWeaver 用 BKN.PreCondition 作为生成源时，要求 BKN 完备性——这又把负担推回到"BKN 自演化"环节，形成飞轮的反身性。

### 3.3 燃料消化：Trace 如何转为 BKN/Context patch

**主要论文：AgentHER（思想借鉴）+ TIDE（动力学指标）**

> ⚠️ 这里需要做关键的概念转换：**AgentHER 原本面向训练数据生成（SFT/DPO），但其"Stage 1 失败分类 + Stage 2 Outcome 提取 + Stage 3 替代目标合成"的思想可以重定向为本体演化**。这是 v1 报告完全没意识到的应用方式。

**重定向 AgentHER 思想到 BKN 自演化**：

```
原 AgentHER（训练侧）              重定向到 BKN 自演化（KWeaver 侧）
─────────────────────────────────────────────────────────
Stage 1: Failure Detector       → 失败分类作为 BKN.Risk 类型扩充输入
Stage 2: Outcome Extractor      → 提取 trace 实际达成 → 用于校准 BKN.Action 后果
Stage 3: Prompt Relabeler       → 不重打 prompt，但生成 "BKN patch 候选"
                                  （例："此 Action 的前置约束应包含 X"）
Stage 4: Data Augmenter         → 不打包训练数据，而是生成 BKN PR
                                  （供人工或自动 merge）
```

**TIDE 的 AUV / LR / MI 在 KWeaver 的应用**：

- **AUV**（成功率随交互轮次的曲线下面积）→ 直接作为 Decision Agent 任务效率指标
- **LR**（Loop Ratio）→ 与 Signals.Execution.Loop 形成"标签 + 量化"组合：标签由信号检测，量化由 LR 给出，**用 DPH 编排定义本身作为 cycle 检测的 ground-truth baseline**（比 TIDE 的 embedding 阈值方法更可靠，KWeaver 独有优势）
- **MI**（Memory Index）→ 当前阶段语义模糊，**暂不实装**

#### 3.3.1 Near-Miss 标注 → DPO 对的零成本生成

> 当 §3.2.1 的 Near-Miss 信号产出 `(原 trace, missing_ro_set)` 二元标注时，**hindsight relabel 几乎不需要 LLM 推理**——纠正轨迹的结构是机械可生成的。

**重定向 AgentHER 思想到 Near-Miss-driven 偏好对生成**：

```
Near-Miss 标注                        机械生成的偏好对（无需 LLM 重写整段轨迹）
─────────────────────────────────────────────────────────
原 trace（latent_failure=TRUE）  →   负样本：[..., MTC(args), ...]
+ missing_ro_set                     正样本：[..., RO_i(args_i), ..., MTC(args), ...]
+ guard_ref（可解释来源）              其中 RO_i 注入位置 = MTC 之前最近的合法点
```

这条路径的**经济性**优于 AgentHER 完整管道：

| 阶段 | AgentHER 完整管道 | Near-Miss-driven 路径 |
|------|------------------|---------------------|
| Stage 1 失败检测 | 通用 LLM 失败分类器 | guard code 重放（确定性，无 LLM）|
| Stage 2 Outcome 提取 | LLM 抽取实际达成 | 已知（trace 终态匹配 gold）|
| Stage 3 Prompt Relabel | LLM 改写 task description | **不需要**——任务不变，只是动作序列要先调 RO |
| Stage 4 数据增广 | LLM 合成对照样本 | **机械注入**：在 MTC 前插一次 RO 调用即可 |

→ 这是 thesis 可证伪命题 (b) "hindsight relabeling of L1-triaged traces produces measurable downstream win rates over random-sampled preference data" 的**最低成本可验证路径**。在 KWeaver 飞轮初期（数据量小、人工预算紧）尤其适合作为第一个 end-to-end 闭环实验。

**与 §3.3 主路径的关系**：完整 AgentHER 重定向（BKN patch 候选生成）覆盖**所有失败类型**；Near-Miss 路径只覆盖**MTC 漏前置检查**这一窄类，但产出的偏好对**质量更高、来源可审计、无需 LLM 二次创作**。两路并行，覆盖率与质量互补。

### 3.4 燃料经济学：成本控制（待补）

> 待 Breaking the Observability Tax 论文 PDF 补到位后回填。当前可用的 placeholder 思路：
> - 信号触发的分级采样（Level 0：仅元数据；Level 1：完整 payload；Level 2：完整 5-surface）
> - Triage Agent 异步运行，不参与实时决策路径——这与 KWeaver 路线图的设计完全一致

### 3.5 评估范式（横切）

**主要论文：Agent-as-a-Judge（4 模块发现）+ TIDE（已纳入 §3.3）+ AgentDebug（root-cause 归因 LLM-judge 实例）**

Agent-as-a-Judge 的最重要工程发现（论文 Table 4）：**最优 4 模块 = Graph + Locate + Read + Ask**；Memory + Planning 模块对评估**有害**。

**对 Triage Agent 内部架构的直接启示**——见 §4.2。

#### 3.5.1 AgentDebug：root-cause 归因的 LLM-judge 路线（off-axis）

> [13] AgentDebug 是 [7] AaaJ 路线在 root-cause 归因任务上的具体实现：5 模块 × 17 类细粒度 error taxonomy + AgentErrorBench（200 标注轨迹，Cohen's κ=0.55——论文叙事 "substantial" 实为 Landis & Koch "moderate"）+ 三阶段 LLM-cascade（fine-grained detector → critical-error finder → re-rollout 循环 ≤5 次）。

**为什么放在评估范式而不在 §3.7 AHE off-axis 之外**：[13] 与 [12] AHE 同属"LLM-as-judge 自动修复"家族，但时间尺度对偶——[12] 改 harness 文件持久化（training-loop-time），[13] 改单 trajectory re-rollout 临时化（inference-time）。两者方法论同向，**自我审查严格度差距明显**：[12] 显式量化 evolve agent 的 fix/regression 预测准确率（5×/2× random，§4.4.2）作为 self-audit；[13] 完全没做这种 self-audit——corrective_guidance 是否真 actionable / re-rollout 失败到底是因为 feedback 错还是 base model 弱，论文从未拆解。意味着 [13] 的"+26% relative"包含大量盲试成分。

**对 KWeaver 的三处可借鉴 + 三处警惕**：

| 借鉴 | 内容 | 警惕 |
|------|------|------|
| **5 模块 × 17 类 taxonomy 作对照词表** | Memory（hallucination / over-simplification / retrieval failure）/ Reflection（progress misassessment / outcome misinterpretation / causal misattribution / hallucination）/ Planning（constraint ignorance / impossible action / inefficient planning / planning–action disconnect）/ Action（format / parameter / misalignment / invalid）/ System（step limit / tool execution / LLM limit / environment）。可与 [2] AgentHER 6 类、Signals 7 类、AHE 7 组件并列作 KWeaver Triage Agent 失败标签词表的对照来源 [13: §2.1] | 17 类边界模糊（论文自承 retrieval failure vs constraint ignorance 需"多轮校准"区分），κ=0.55 即标注者一致性天花板；KWeaver 取词表时**取模块切分而非细类标签**更稳 |
| **强制四模块 tag 化的 schema 收益** | Rollout strategy ablation（Figure 7c）：Modular 0.38 vs ReAct 0.26 = +12 pp 成功率，且这部分增益**不需要任何 detector 介入**就能拿到。直接坐实 thesis"L0 schema 是 silent gating constraint"——schema 缺失直接拿走 31% 相对增益 [13: §5.1] | KWeaver Decision Agent 已强制结构化输出（plan / reflection / thought 字段建议见 §3.1），与 [13] 该结论同向 |
| **Memory + Reflection 占根因 ~38% 的分布** | §2.2 Figure 3：失败多在 step 6–15 中段聚集，Memory + Reflection 是最主要的根因来源 [13: §2.2] | 跨 benchmark family 不可迁移：[13] 测试床（ALFWorld / WebShop / GAIA）强依赖 long-context 记忆与状态跟踪；KWeaver 关心的 DPH 编排（弱用户对话 + 强 tool/API 链）场景下，Memory 错误频度可能远低，Action / System 错误占比应远高 |

**严格不做**：不把 AgentDebug 三阶段 cascade 直接搬进 KWeaver Triage Agent。原因——单条 ALFWorld trace（中位 ~10–15 步）的 detection 阶段就要 4 模块 × 步数 ≈ 40–60 次 GPT-4.1 调用 + critical finder + 最多 5 次 re-rollout 的 detector 再调用；论文从未拆 detection / critical-finder / re-rollout 三段 token 占比，"matched by total token usage" 的"匹配"无从核验。在 thesis"front-line filtering 用轻量信号"判断方向上，[13] 把 LLM-judging 用在**全部失败轨迹的全部步全部模块**——thesis 明确反对的部署模式。如借鉴，**严格限定在 Signals/Near-Miss/Sentinel 已 triage 后的小子集**作为 §4.2.4 Read 模块的 backbone（详见 contradictions.md "front-line vs triaged-subset"）。

**Detector 强模型依赖警示**：§5.1 Figure 7b——detector 从 GPT-4.1 换成 Llama-3.3-70B，All-Correct 直接从 32% 跌到 6%；GPT-4o-mini 跌到 2%。意味方法的有效性几乎完全绑在最强闭源 base 上。KWeaver 设计若走 LLM-judge backbone 路线，必须**显式选模型 + 准备 fallback**，并定期跑 detector 端的 base-model 对照测试（这是 [13] 自己未做的可移植性检查）。

### 3.6 训练侧（远期）

**AgentHER（完整管道）+ TSR**：保留为远期参考。当 KWeaver 飞轮数据量级达到支撑领域微调时（路线图明确的 reassessment 条件），重新评估。当前阶段不做。

### 3.7 Off-axis：Harness 自演化方法论（AHE）

**主要论文：[12] Agentic Harness Engineering**

> AHE 不在 KWeaver 飞轮的四环节内——它优化的是**给定模型 + 给定 benchmark 下，harness 自身的组件配置**。但其方法论原语对 KWeaver 至少有三处可借鉴：

| AHE 原语 | KWeaver 借鉴方向 | 警惕点 |
|---------|----------------|-------|
| **Component observability**：把可编辑面切成 7 类正交文件级组件（system prompt / tool desc / tool impl / middleware / skill / sub-agent / long-term memory），每类失败模式映射单一组件类，每条 logical edit = 一次 git commit | KWeaver 的 BKN 演化、Triage Agent 自身组件、Context Loader 策略集都可遵循"文件级正交 + git diff 可观察"原则——这是 thesis"L0 schema 是 silent gating constraint"在工程层的等价投影：没有解耦的 substrate，归因不可能 [12: §3.1] | "正交"是工程契约不是效果属性——AHE 自己实证三个单组件增益相加 11.1 pp 远超 full AHE 的 7.3 pp，说明 component 之间在效果上实际相互干扰 [12: §4.4.1] |
| **Falsifiable change manifest**：每条 edit 自带 `predicted_fixes[]` + `risk_tasks[]`，下一轮的 task delta 自动判决，falsified 的 edit 文件级回滚 | 可直接移植为 KWeaver detector versioning / 信号阈值演化的治理样板：每次 detector 调整都附带"我会修哪些已知失败 / 我会引入哪些回归"的预言，下个评估窗口核对，未生效自动回滚——把 rationale-driven self-justification 换成 contract-by-next-evaluation [12: §3.3, Algorithm 1] | AHE 自己实证 fix-prediction precision 33.7%（5× random），但 regression-prediction 精度仅 11.8%（仅 2× random）——agent 大体能命中"我会修好哪些"，**几乎无法预言"我会弄坏哪些"** [12: §4.4.2 Figure 4]。这直接量化了 §4.2.5 中 Triage Agent 自我归因的可信边界 |
| **Agent Debugger（trajectory 文件系统化）**：把 trajectory 当 navigable file environment（每 message 一个文件），LLM-agent 用通用 shell + scripting 浏览，按 progressive disclosure 输出 per-task + benchmark-level 报告 | 形式上是把 raw trace → 分层证据语料的方法论原语，可作为 KWeaver Triage Agent §4.2.4 Read 模块的"如何让 LLM 读 trace"实现参考 | **这条与 thesis 立场对立**：Agent Debugger 是 LLM-as-judge 路线的具体实现，且 AHE 把它用在**全部 trajectory**（每任务 k=2 traces 全跑），是 thesis"front-line filtering 用轻量信号"明确反对的部署模式。引入须严格限定在 triage 后的小子集 [12: §3.2; 见 contradictions.md] |

**对 KWeaver 决策的净效应**：AHE 的 ablation 数据带来一个不愉快的拷问——其 component-level 实证显示**system prompt 单换入 −2.3 pp（唯一退化），long-term memory 单换入 +5.6 pp，tool 单换入 +3.3 pp**。即"把经验显式落到外部文件"比"把经验编进 system prompt"或"通过 SFT 编进权重"效果更稳定 [12: §4.4.1 Table 3]。这给 §3.3 / §3.6 的 model-side relabeling 路线提出工程对照点：在 KWeaver 飞轮初期（数据少、人工预算紧），把 Triage Agent 的输出落到 BKN / long-term memory 文件可能比落到 SFT 数据更划算——这与 §1.3 "BKN 自演化是 Scaling 组件" 的判断相互印证。

**严格不做**：不把 AHE 的 evolve-loop 直接搬进 KWeaver 飞轮。AHE 的演化对象是 harness（model 不变）；KWeaver 飞轮的演化对象是 BKN + Context 策略 + 可选 model 微调——任务结构不同。AHE 的 32 小时 + 数十亿 token 的成本曲线在 KWeaver 这种长期持续运行的飞轮场景下不可重复支付。

### 3.8 可证伪点追踪（Falsifiability tracking）

> 本节追踪 thesis "Working thesis" 中三条显式可证伪命题在当前已读语料下的证据状况。每条命题随飞轮闭环持续可验证；新论文进入时按"支持 / 反向 / 不可解"分类，落实"counter-evidence on any of these should force a reframing"的 thesis 自约束。

**可证伪命题 (a)**：lightweight signals 在真实非 τ-bench 语料上达到 >70% informativeness。
- 现有支持证据：[1] Signals 在 τ-bench 上 informativeness 82%（vs 随机 54%）——但**仍是 τ-bench**，命题严格意义上未被验证。
- 间接支持：[10] Sentinel 在 ToolEmu / OAgent 上 acc 0.93 / F1 0.93（声明式 KG 不变量，§6.1）；[11] Near-Miss 在 τ²-Airlines 含 MTC 子集 8.6%–17.3% latent failure 检出率，code-gen 路径 P=R=1.00。两条都属"轻量、规则化、轨迹级判定"家族但测的是判定准确性而非采样信息量，相关但不可直接折算。
- 反向 / 警示证据：[13] AgentDebug 在 ALFWorld/WebShop/GAIA 上 LLM-judge detector 路线 All-Correct 24.3%（GPT-4.1）/2-6%（小模型），与 informativeness 不可比但提示"任何 detector 在跨基准上数字会显著下滑"——KWeaver 业务语料独立复现前不能假设 82% 数字可移植。
- 下一步可解：在 KWeaver 现有 trace 池抽样 100 条做 §4.2.7 实验，测 Signals 检测器与 KWeaver 业务专家标注的 informativeness 一致率。

**可证伪命题 (b)**：L1-triaged 轨迹的 hindsight relabel 比随机采样产生可测量 downstream win rate。
- 现有支持证据：[2] AgentHER 端到端 SFT/DPO 提升幅度（论文报告，本仓 02 笔记紧凑版未落具体数字）；[11] Near-Miss 在 mutating-tool 场景给出**零成本 DPO 对生成路径**（机械注入 RO，§3.3.1）——这是命题 (b) 的最低成本可验证路径。
- 反向 / 警示证据：[12] AHE long-term-memory 单换入 +5.6 pp（§4.4.1 Table 3）暗示"externalize 比 SFT 更划算"——relabel + SFT 路线的边际收益相对"把同样信息落到 BKN / memory 文件"需独立核验。如果 externalize 路径胜出，命题 (b) 不必假，但 KWeaver 落地优先级要重排（§4.4 BKN 自演化 > §3.6 训练侧）。
- 下一步可解：Triage Agent prototype（§4.2）落地后，比较"Near-Miss-driven DPO 对 vs 随机 DPO 对"在同任务 hold-out 上的 win rate；同时跑 externalize-only baseline 作三方对照。

**可证伪命题 (c)**：signal-driven sentinel sampling 把观测成本降 >5× 而不损 downstream 训练数据质量。
- 现有支持证据：[5] Breaking the Observability Tax 概念性主张（PDF 阻塞，待补——见 §6.1）；[10] Sentinel 在动作时反事实模拟 O(|M|) 复杂度（§6.1），证明声明式不变量在 enforcement 侧成本可控。
- 反向 / 警示证据：[13] AgentDebug 走相反方向（每条失败 trace 全 detection，单条 ALFWorld trace ~40–60 次 GPT-4.1 调用），代表"无成本约束 / 全量 LLM-judge"对照组——其 +26% relative 收益的成本未拆，无法构成命题 (c) 的反例但是其规模对照。
- 下一步可解：补 [5] PDF 后落地分级采样实验（Level 0 元数据 / Level 1 完整 payload / Level 2 完整 5-surface），在 KWeaver trace 池上量化 cost vs 下游 Triage Agent 输出质量曲线。

**追踪规则**：每篇新论文进入 notes/ 时，必须在本节相应命题下追加一行（"支持 / 反向 / 不可解"标签 + 简短理由 + note 编号）。如某条命题积累 ≥3 条独立反向证据，触发 thesis reframing 提议（落到 contradictions.md 而非本节）。

---

## 4. 直接服务 KWeaver 当前差距的具体建议

按"差距优先级 + 学术研究覆盖度"排序：

### 4.1 双轨 Trace 字段补全 + 关联查询能力

**对应路线图差距**："调用链-证据链的关联查询能力待加强"

**学术依据**：AgentTrace 的 trace_id + span_id 嵌套机制；Signals 对 5-surface 数据基座的需求

**具体动作**：

- [ ] 双轨字段对齐到 OTel `gen_ai.*` semantic conventions
- [ ] 调用链 ↔ 证据链以 `trace_id` 为主键的反向查询 API
- [ ] Cognitive surface 字段结构化（plan / reflection / thought 显式字段）
- [ ] Environment 类事件以 trace_id 关联到调用链
- [ ] **写时 schema 校验 + 降级而非阻塞**

### 4.2 Triage Agent prototype 设计 ⭐ 最重头

**对应路线图差距**："Triage Agent 概念阶段，尚无 prototype；输入信号格式、输出动作定义、与 Decision Agent 的关系均待设计"

> 这是当前最大、最具体的 gap，也是学术研究最能直接服务的环节。本节给出可执行的 prototype 设计建议。

#### 4.2.1 输入信号格式

输入是**带信号标签的双轨 Trace 流**，不是裸 trace：

```yaml
# Triage Agent 输入：每条 trace 携带元数据
trace_record:
  trace_id: "..."
  ts: "..."
  outcome: success | failure | partial
  
  # 双轨数据
  call_chain: [...]       # 调用链（已有）
  evidence_chain: [...]   # 证据链（已有）
  
  # 信号标签（待新增 — Signals 论文 7 类）
  signals:
    interaction:
      - misalignment      # 检测到用户重述
    execution:
      - loop              # 检测到工具调用循环
    environment: []
  
  # ISF / Guardrails 触发记录（KWeaver 特有）
  guardrails:
    triggered: [...]       # 哪些 guardrail 拦截了
    risk_level: "high"
  
  # 动力学指标（来自 TIDE 思想）
  dynamics:
    auv: 0.42              # 成功率曲线下面积
    lr: 0.18               # Loop ratio
```

#### 4.2.2 输出动作定义

Triage Agent 输出**4 类结构化建议**，**不是分数排名**（坚持 Signals "信号不是质量分"立场）：

| 输出类型 | 内容 | 消费者 |
|---------|------|--------|
| **BKN patch 建议** | 概念定义/关系/行动/风险的修改候选 + 证据引用 | BKN 自演化通道 → 人工或自动 merge |
| **Context 策略调优参数** | limit / Schema 预加载 / 工具集 / 路径指引 的调整 | Context Loader 策略学习模块 |
| **失败归因报告** | 6 类失败标签（参考 AgentHER 分类法）+ 严重度权重 | 工程团队 + 人工 review |
| **人工 review 队列** | 高价值 trace 子集（高风险 + 高歧义）| 审计员 / 业务专家 |

> **关键设计原则**：每个输出都附带 trace_id 作为证据指针，可双向追溯。这与 KWeaver "可追溯证据链"的产品价值直接对齐。

#### 4.2.3 与 Decision Agent 的关系

```
Decision Agent（实时决策路径）
    │
    │ 持续产生 trace
    ▼
TraceAI 双轨 Trace（物理底座）
    │
    │ 异步消费（不阻塞 Decision Agent）
    ▼
Triage Agent（异步）
    │
    │ 产出：BKN patch / Context 策略 / 归因 / review
    ▼
反哺通道（BKN / Context Loader / 人工）
```

**严格约束**：
- ✅ Triage Agent **完全异步**，不参与实时决策
- ✅ Triage Agent **不修改正在执行的 trace**（read-only）
- ✅ Triage Agent **可以失败 / 延迟**——飞轮可以慢，但不能阻塞业务

#### 4.2.4 内部架构（基于 Agent-as-a-Judge 4 模块发现）

> Agent-as-a-Judge 论文 Table 4 的关键工程发现：**最优 4 模块 = Graph + Locate + Read + Ask**；**Memory + Planning 有害**。

直接借鉴到 Triage Agent prototype：

```
┌──────────────────────────────────────────┐
│ Triage Agent 内部架构（v0 prototype）     │
├──────────────────────────────────────────┤
│ 1. Graph    — 构建 trace 的执行图           │
│              （调用链 + 证据链合一视图）     │
│ 2. Locate   — 定位 trace 中的关键节点       │
│              （信号触发点 / guardrail 触发） │
│ 3. Read     — 读取节点的 cognitive +       │
│              contextual 数据               │
│ 4. Ask      — 给出 4 类输出之一            │
│              （prompt-engineered，非微调）  │
│                                          │
│ ❌ 不做 Memory（避免错误链）                │
│ ❌ 不做 Planning（避免决策不稳定）           │
└──────────────────────────────────────────┘
```

**Backbone 选择**：起步阶段用闭源 SOTA（Claude Sonnet / GPT-4 系列）保证质量；待飞轮成熟后切到开源/自研。

#### 4.2.5 反馈信号收集机制

Triage Agent 自己的输出也需要被评估，否则飞轮的"反哺通道"是单向开放的：

- [ ] **BKN patch 接受率**：人工审计 patch 中被 merge 的比例
- [ ] **策略调优有效性**：Context Loader 应用调优参数后的下一轮 trace 表现
- [ ] **失败归因准确率**：归因报告与人工最终判定的对齐率
- [ ] **review 队列价值密度**：人工 reviewer 在队列中找到真问题的比例（参考 Signals 的 informativeness rate）
- [ ] **自我预言的 fix vs regression 不对称性**：每条 patch 建议附带 `predicted_fixes[]` + `risk_tasks[]`（借鉴 [12] AHE change manifest 模式），下一窗口对照实际效果。AHE 实证：跨 9 轮均值 fix-prediction precision 33.7% / recall 51.4%（≈5× random），但 regression-prediction precision 11.8% / recall 11.1%（仅≈2× random）[12: §4.4.2 Figure 4]——即 evolve agent 大体能命中"我会修好哪些"，**几乎无法预言"我会弄坏哪些"**。KWeaver 设计意涵：Triage Agent 的"修复预测"可作为弱信号驱动 patch 自动 merge，但**回归预测必须独立核验**——不能依赖 Triage Agent 自己声明的 risk_tasks 做"安全"判断。这是 §4.4 BKN 自演化"自动 merge 分流策略"的可信边界依据。

#### 4.2.6 评估指标 — 不要打分

> 重申 Signals 论文的核心立场：**信号不是质量分**。Triage Agent 输出的"价值"应通过下游消费者的**接受率/有效性**来度量，而不是 Triage Agent 自己给自己打分。

**禁止**：Triage Agent 给 trace 打"质量分"或"价值分"。
**允许**：Triage Agent 给 trace 打"标签 + 推荐动作"。

#### 4.2.7 Prototype 验证实验设计

参考 Signals 论文的小规模标注实验：

- 从 KWeaver 现有 trace 池中抽样 100 条（含成功 + 失败）
- 3 名领域专家盲评：每条 trace 是否值得 Triage Agent 处理（informative）+ 应该输出哪类 patch
- 跑 Triage Agent prototype
- 对比指标：
  - 与人工的 informativeness 标签对齐率（目标参考 Signals 82%）
  - 输出 patch 与人工建议的语义接近度
  - 端到端处理延迟（指标）

→ **这是 v1 阶段最直接的"AI 工程闭环"自我验证**。

### 4.3 Context Loader 策略学习

**对应路线图差距**："硬编码规则 → 策略参数化 + 反馈信号采集机制"

**学术依据**：Signals 信号作为反馈输入；TIDE AUV 作为效率衡量

**具体动作**：

- [ ] **策略参数化**（替换硬编码）：
  - `retrieval_limit`（《深度解析》验证：10 → 20 准确率 +8.27pp）
  - `schema_preload`（验证：Token 减 8.29K + 准确率 +2pp）
  - `tool_set_active`（验证：精简工具 99.31% 准确率）
  - `path_guidance_template`（验证：99.31% + 延迟 -29% + Token -22%）
- [ ] **反馈信号通道**：把 Triage Agent 的"策略调优"输出作为 Context Loader 策略更新的输入
- [ ] **A/B 实验框架**：策略变更后跑控制实验，AUV / 准确率 / Token 是核心指标

### 4.4 BKN 自演化（从 Trace 反向矫正）

**对应路线图差距**："从 Trace 信号到 BKN patch 的自动闭环未打通"

**学术依据**：AgentHER 思想重定向（见 §3.3）

**具体动作**（依赖 Triage Agent prototype 先就位）：

- [ ] Triage Agent 产出的 `BKN patch 建议` 接入 BKN 版本管理
- [ ] **人工 + 自动 merge 的分流策略**：低风险 patch（如概念描述补全）可自动 merge；高风险 patch（如新增 Risk 类型）走人工审计
- [ ] **回滚机制**：BKN 自动 merge 后，下一轮 trace 表现下降时自动回滚

### 4.5 Guardrails 自动生成（轻提）

**对应路线图差距**："静态 Guardrails / Risk 描述是脚手架，必须自动生成"

**学术依据**：弱——这块当前论文研究覆盖少。**这是研究空白，KWeaver 自己探索**。

**初步思路**：从 Trace 中"应当被 guard 但未被 guard"的失败案例反向归纳新 Guardrail 候选——这是反向工程的研究问题。

---

## 5. 与生态对比的精确定位

> 引自路线图设计稿。这部分对外讲故事时极重要，对内决策也很重要。

| 维度 | KWeaver | LangSmith / LangFuse | Palantir Foundry / AIP |
|------|---------|---------------------|----------------------|
| 核心定位 | 企业决策智能体的 Harness 运行时 | Agent 开发者的可观测 + 评估工具链 | 一体化企业操作系统 |
| 语义层 | **BKN（AI 原生业务本体，Markdown 载体）** | 无内置语义层 | Ontology（平台绑定，重型配置）|
| Trace 体系 | **调用链 + 证据链双轨**，面向 Agent 决策审计 | 调用链为主，面向开发者调试 | 平台内置审计，绑定 Foundry 生态 |
| 执行约束 | **ISF 语义级授权 + Guardrails** | 无执行约束（观测侧工具）| 权限隔离强但绑定平台 |
| 飞轮闭环 | **Trace → Triage → BKN/策略自动演化** | Trace → 评估 → 人工优化 prompt | 闭环存在但绑定 Palantir 实施体系 |
| 部署模式 | 模块解耦，轻量部署，开源/国产化适配 | SaaS 为主 | 深度定制，实施周期长 |

**TraceAI 团队的工作必须强化"双轨 Trace"这个相对其他玩家的差异化点**——LangSmith 没有证据链，Palantir 没有面向 Agent 决策的双轨设计。

---

## 6. 阻塞与限制

### 6.1 学术资料阻塞（待补 PDF）

| 论文 | 状态 | 对本报告的影响 |
|------|------|--------------|
| Breaking the Observability Tax | ❌ 未下载（IEEE Access CC-BY） | §3.4 燃料经济学 + §4.2 异步采样章节是占位 |
| AgentSeer | ❌ 本地 PDF 是 Short-PHD（不相关） | 拓扑异常检测路线缺失（与 Triage Agent 的 Loop 检测可能互补）|
| TRACE | ❌ 本地 PDF 是数学论文（不相关） | §3.5 评估范式只有 TIDE 一半 |

### 6.2 KWeaver 内部信息盲点

写本报告时仍有不完全清楚的点（需要团队确认）：

- [ ] 当前调用链 / 证据链的具体 schema 字段（与建议的 5-surface 映射对齐情况）
- [ ] kweaver-dip 的 bkn-creator 当前的自动构建覆盖率与质量数据
- [ ] Decision Agent 的"用户交互层"在哪些场景存在（决定 Interaction 信号是否优先实装）
- [ ] ISF 的 guardrail 触发记录是否已经写入 trace（决定 Triage Agent 输入格式是否已就绪）

### 6.3 学术研究本身的局限

每篇论文笔记 §5（批判性阅读）已详述。报告层面归纳：
- **可复现性**：Signals 检测器实现细节、Agent-as-Judge 模块组合空间、TIDE cycle 阈值选择都不充分公开
- **测试床狭窄**：τ-bench / DevAI / 5 个 grid puzzle benchmark——KWeaver 业务的实际表现需独立复现
- **生产成本未量化**：所有论文都说"low cost"，没人给真实运维数字。AHE 是少数明确给出量级的——一次 10 轮演化 ~32 小时 + 估计 10⁹ 量级 token，但仍未拆"rollout / debug / evolve"三类调用的 token 占比 [12: §4.1, §Limitations]
- **LLM-driven trajectory 解读未做相对 raw-trace 的消融**：[12] AHE 的 Agent Debugger（LLM-as-judge for harness diagnosis）是 thesis 反对的"front-line LLM-judge"模式，且论文未做"evolve agent 直接读 cleaned raw trace"的消融——无法分清增益是"分层结构化"在起作用还是"又一次 LLM 重新解释"在起作用
- **Detector 端基模强依赖未被任何 LLM-judge 论文系统体检**：[13] AgentDebug §5.1 Figure 7b 是少数显式做的——detector 从 GPT-4.1 换成 Llama-3.3-70B 后 All-Correct 直接从 32% 跌到 6%，GPT-4o-mini 跌到 2%——意味方法的有效性几乎完全绑在最强闭源 base 上。同源问题在 [7] AaaJ / [12] AHE 上也存在但未独立量化。KWeaver 走 LLM-judge backbone 路线时，**必须**显式选模型 + 准备 fallback + 定期跑 base-model 对照测试
- **κ 指标的术语膨胀**：[13] AgentErrorBench 报告 Cohen's κ=0.55 并叙事为 "substantial agreement"，但 Landis & Koch 标准为 "moderate"（0.41–0.60）。其 strict All-Correct 24.3% 数字的天花板恰恰被 κ=0.55 限制——这是 detector 评测论文常见的"基础地基不稳但叙事乐观"模式，KWeaver 自评 Triage Agent 时应**自报 κ 严格档位**而非沿用论文叙事
- **同篇论文内部方法描述不一致**：[13] Algorithm 1 注释 "Critical Error Detection via LLM (no rollout/counterfactuals)" 与 §3.2 文字 "perform counterfactual testing step by step: at each point, we substitute a corrected action and test whether the rollout would succeed" 直接冲突；附录 A.5 prompts 证实是 LLM-only "想象式 counterfactual"。该论文在术语上沿用 "counterfactual" 一词但语义滑移成"在 prompt 里让 LLM 想象 corrected action"——下游引用须以 Algorithm 1 + 附录 A.5 为准（同条已落 contradictions.md）

### 6.4 当前阶段不做的事

> 与 KWeaver 路线图非目标对齐：

- ❌ **不做自研 / 蒸馏模型**：当前外部模型够用；飞轮数据量级支撑领域微调时重评
- ❌ **不做 Chat / Board UI 重大迭代**：UI 降级，预算留给 Agent-first 核心能力
- ❌ **不追求单纯学术 Benchmark 跑分**：飞轮内部评测服务于自演化，不对外做榜单

---

## 7. 一句话结论

> **TraceAI 团队当前的最高优先级，不是把可观测做得更全，而是把"双轨 Trace + Triage Agent"这条飞轮反哺通道打通**——这是 KWeaver 当前最大的具体差距，也是学术研究最能直接服务的环节。学术界 6 篇深读论文中，**Signals 直接定义了 Triage Agent 的输入信号格式**，**Agent-as-a-Judge 的 Memory+Planning 有害发现直接定义了 Triage Agent 的内部架构**，**AgentHER 的 Stage 1-2 思想可以重定向为 BKN 自演化的 patch 生成机制**。这构成了一条**从学术到 KWeaver prototype 的清晰映射**。

---

## 附录 A：KWeaver 组件 ↔ 论文术语速查

| KWeaver 组件 / 概念 | 学术对应 | 关系 |
|-------------------|---------|------|
| 双轨 TraceAI（调用链）| AgentTrace Operational + 部分 Contextual surface | 字段对齐 |
| 双轨 TraceAI（证据链）| AgentTrace Cognitive + 部分 Contextual | 需结构化 |
| Decision Agent | 论文中"agent under evaluation" | 被观测对象 |
| Triage Agent（待建）| Signals 信号 + Agent-as-Judge 4 模块架构 | 综合借鉴 |
| BKN（业务知识网络）| Ontology / domain model | KWeaver 特有 + Markdown 载体 |
| BKN Lang | DSL（Markdown 扩展） | KWeaver 特有 |
| ISF（信息安全编织）| 论文未涉及（学术空白）| KWeaver 特有 |
| Guardrails | 论文未深入；AgentHER 的 severity weighting 部分相关 | 大体属研究空白 |
| Context Loader | 论文中 "context engineering" 工具链 | 可借鉴 reranking / compression |
| VEGA 数据虚拟化 | 论文未涉及（基础设施层）| KWeaver 特有 |
| 信号（Triage 输入）| Signals 7 类 | 直接借鉴 + KWeaver 业务扩展 |

## 附录 B：v1 → v2 主要变更对照

| 章节 | v1 | v2 |
|------|-----|-----|
| KWeaver 定位 | 推理侧 agentic harness | Harness 运行时 + AI 工程飞轮 |
| TraceAI 角色 | 观测层（Layer 0 基础设施）| 飞轮物理底座（双轨 Trace 是核心数据资产）|
| 主要 deliverable 建议 | Triage Center MVP（5-surface schema + 7 信号检测器）| **Triage Agent prototype**（输入/输出/架构定义清楚）|
| 数据飞轮 | 12+ 个月不要建 | **核心产品形态**，从 day 1 就在建 |
| 评估范式策略 | 不要建质量评分系统 | 立场不变，但 Triage Agent ≠ Judge——是反馈通道 |
| 学术论文组织 | 4-Layer Stack（学术友好）| 飞轮 4 环节（决策友好）|
| AgentHER 角色 | "训练侧、当前不落地" | **思想重定向**到 BKN 自演化 |
| 与生态对比 | 缺失 | 新增（vs LangSmith / Palantir）|

## 附录 C：参考资料

### KWeaver 项目资料（脱敏后入 `references/`）

| 文件 | 内容 |
|------|------|
| [`references/01_overcoming_ontology.md`](references/01_overcoming_ontology.md) | KWeaver 的 AI 工程能力如何超越本体论（飞轮纲领）|
| [`references/02_engineering_roadmap.md`](references/02_engineering_roadmap.md) | 工程路线图设计稿（三层结构 + Scaling 审视 + 现状差距）|
| [`references/03_kweaver_core_overview.md`](references/03_kweaver_core_overview.md) | KWeaver Core 产品架构与组件概览 |
| [`references/04_kweaver_core_deep_analysis.md`](references/04_kweaver_core_deep_analysis.md) | 非结构化数据问答的可靠性深度分析（含 145 样本消融）|
| [`references/05_harness_engineering_position.md`](references/05_harness_engineering_position.md) | Harness Engineering 三阶段演进与生态位置 |

入库脱敏规则见 [`references/README.md`](references/README.md)。原始材料保留在本地 `~/Downloads/`，不入库。

### 学术论文索引

完整索引见 [`references/papers/INDEX.md`](references/papers/INDEX.md)。已深读 6 篇（Signals / AgentHER / TSR / AgentTrace / Agent-as-Judge / TIDE）；3 篇阻塞待补（Breaking Obs Tax / AgentSeer / TRACE）。

学术 PDF 不入 git（gitignored）。**PDF 拉取由外部 researcher 项目负责**，本项目只维护 `references/papers/INDEX.md` 中的 arXiv ID 与一句话定位。

### 详细笔记位置
- `notes/00_research_landscape.md` — 研究全景图（v1 时期版本，需要按 v2 重新组织）
- `notes/01_signals_trajectory_triage.md` — Signals 7-Block 深读
- `notes/04_agenttrace_structured_logging.md` — AgentTrace 7-Block 深读
- `notes/07_agent_as_a_judge.md` — Agent-as-Judge 7-Block 深读
- `notes/08_tide_trace_diagnostics.md` — TIDE 7-Block 深读（TRACE 半待补）
- `notes/02_agenther_hindsight_relabeling.md` — AgentHER（紧凑版）
- `notes/03_tsr_trajectory_search_rollouts.md` — TSR（紧凑版）

---

## 版本更新日志

| 版本 | 日期 | 新增论文 | 关键变化 |
|------|------|---------|---------|
| v2.0 | 2026-04-26 | — | 重大修正：基于 KWeaver 路线图设计稿，全面对齐 Harness Engineering 定位与飞轮中心论；从 4-Layer Stack 重组为飞轮 4 环节 |
| v2.1 | — | [09] Trajectory Guard | L1 分诊层"学习型小代理"对照组（待回填） |
| v2.2 | — | [10] Sentinel / PhantomPolicy | L1 enforcement 侧"声明式 KG 不变量"样板；为"schema 是 silent gating constraint"提供 Coverage 实证（待回填） |
| v2.3 | 2026-04-30 | [11] Near-Miss | §3.2.1 新增 Near-Miss 信号定义（成功轨迹中的 latent failure 检测），与 [10] Sentinel 形成 in-line/post-hoc 互补；§3.3.1 新增 Near-Miss-driven DPO 对零成本生成路径，作为 thesis 命题 (b) 的最低成本可验证路径；Triage Agent 输入 schema 扩充 `latent_failures` 字段 |
| v2.4 | 2026-05-01 | [12] AHE | 新增 §3.7 "Off-axis：Harness 自演化方法论（AHE）"——记录三处可借鉴原语（component observability / falsifiable change manifest / Agent Debugger）与各自警惕点；§4.2.5 新增 fix vs regression 自我预言不对称性条目（5× vs 2× random，§4.4.2 实证）作为 BKN 自动 merge 可信边界依据；§6.3 新增 LLM-driven trajectory 解读相对 raw-trace 未消融的局限；contradictions.md 记录 [12] vs [11] "LLM-driven exploration vs deterministic mechanistic oracle" 范式对立 + 提案"横切面 ─ Harness 自演化"分类轴扩展（待人工 review） |
| v2.5 | 2026-05-04 | [13] AgentDebug | §3.5 评估范式新增 §3.5.1 子节，定位 [13] 为 [7] AaaJ 路线在 root-cause 归因任务上的具体实现；与 [12] AHE 同属 LLM-as-judge 自动修复家族但时间尺度对偶（[12] training-loop 改 harness vs [13] inference-time 改 trajectory），且 [13] 缺 self-audit；新增 §3.8 "可证伪点追踪"——按 thesis 三条可证伪命题 (a)(b)(c) 分别落地"支持/反向/不可解"证据 + 下一步可解，关闭审计中"可证伪点追踪非 body 章节"差距；§6.3 新增四条限制（detector 强模型依赖未系统体检 / κ 术语膨胀 / 同篇内部方法描述不一致）；contradictions.md 记录 [13] vs [11] "trajectory root-cause judgment 机制 vs LLM-judge"（与 11↔12 同源第三例）+ [13] vs thesis "front-line vs triaged-subset 部署"+ [13] 同篇 Algorithm 1 与 §3.2 内部不一致 |

*Report version: v2.5（2026-05-04）。基于 KWeaver 路线图设计稿，全面对齐 Harness Engineering 定位与飞轮中心论；§3.8 落实 thesis 可证伪点追踪闭环。*
