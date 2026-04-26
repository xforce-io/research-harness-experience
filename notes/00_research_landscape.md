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

## 与 KWeaver TraceAI 的映射

| 研究层 | KWeaver 落地组件 | 当前状态 |
|--------|-----------------|---------|
| Layer 0 基础设施 | TraceAI Collector / Schema | 已有基础，需参考 AgentTrace 补全 Schema |
| Layer 1 信号分诊 | TraceAI Triage Center (待建) | 需实现正则探针 + 状态机探针；[9] Trajectory Guard 作"学习型小代理"备选路线（黑箱、二元输出，与可解释规则路线互斥） |
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

3. 🟢 **P2 — 参考，拓宽理论视野**
   - [3] TSR — 训练期轨迹优化（与部署后场景互补）
   - [7] Agent-as-a-Judge — 评估范式对照
   - [8] TIDE / TRACE — 理论延伸
