# KWeaver TraceAI：服务于 AI 工程飞轮的双轨 Trace 与 Triage Agent

> **形式：** 内部技术报告 / Position Paper（v2.0）
> **作者：** xupeng
> **日期：** 2026-04-26
> **基于：**
> - 6 篇深读论文（Signals、AgentTrace、Agent-as-a-Judge、TIDE、AgentHER、TSR）+ 3 篇待补 — 索引见 [`references/papers/INDEX.md`](references/papers/INDEX.md)
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

### 3.4 燃料经济学：成本控制（待补）

> 待 Breaking the Observability Tax 论文 PDF 补到位后回填。当前可用的 placeholder 思路：
> - 信号触发的分级采样（Level 0：仅元数据；Level 1：完整 payload；Level 2：完整 5-surface）
> - Triage Agent 异步运行，不参与实时决策路径——这与 KWeaver 路线图的设计完全一致

### 3.5 评估范式（横切）

**主要论文：Agent-as-a-Judge（4 模块发现）+ TIDE（已纳入 §3.3）**

Agent-as-a-Judge 的最重要工程发现（论文 Table 4）：**最优 4 模块 = Graph + Locate + Read + Ask**；Memory + Planning 模块对评估**有害**。

**对 Triage Agent 内部架构的直接启示**——见 §4.2。

### 3.6 训练侧（远期）

**AgentHER（完整管道）+ TSR**：保留为远期参考。当 KWeaver 飞轮数据量级达到支撑领域微调时（路线图明确的 reassessment 条件），重新评估。当前阶段不做。

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
- **生产成本未量化**：所有论文都说"low cost"，没人给真实运维数字

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

*Report version: v2.0（2026-04-26）。重大修正基于 KWeaver 路线图设计稿，全面对齐 Harness Engineering 定位与飞轮中心论。*
