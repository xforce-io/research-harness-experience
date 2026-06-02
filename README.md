# Agent Harness Research

聚焦 **agent harness / post-deployment agent 工程** 的学术研究仓:论文索引 + 深读笔记 + 研究地图。关注生产部署后的 agent 轨迹可观测性(Observability)、信号分诊(Signal Triage)、后见之明重打标(Hindsight Relabeling)与数据飞轮(Data Flywheel),不绑定任何具体产品或厂商。

## 目录结构 (Directory Structure)

*   `papers/`: 论文索引(`papers/README.md`,按 arXiv ID 维护)。**PDF 原文件不纳入 Git**(见 `.gitignore`)。
*   `notes/`: 论文 7-Block 深读笔记。`00_research_landscape.md` 是全局研究地图(论文间关系 + 四层栈定位 + 阅读优先级)。
*   `.researcher/`: 自动化 researcher 的配置——`thesis.md`(工作论点)与 `project.yaml`(研究问题、纳入/排除标准、检索源)。

## 研究主轴:Post-Deployment Agent Engineering Stack

一条四层栈,外加横切的评估范式对照组:

| 层 | 关注点 | 代表论文 |
|----|--------|---------|
| **L0 基础设施** | 结构化 trace schema、低成本遥测 | AgentTrace [4]、Breaking the Observability Tax [5] |
| **L1 信号分诊** | 不靠昂贵 LLM 评估,用行为模式 / 正则 / 拓扑 / 小代理低成本筛出高复盘价值轨迹 | Signals [1]、AgentSeer [6]、Trajectory Guard [9]、Sentinel [10]、Near-Miss [11] |
| **L2 数据转化** | 把失败 / 带摩擦轨迹重写为偏好数据(DPO/RLHF) | AgentHER [2]、TSR [3] |
| **L3 模型迭代** | 对齐 → 部署 → 产生新轨迹 → 回到 L0 | (闭环) |
| **横切·评估范式** | 重量级语义评估对照组(留给分诊后的小子集) | Agent-as-a-Judge [7]、TIDE/TRACE [8] |
| **off-axis·Harness 自演化** | 训练 / 推理期的 harness 自动演化方法论(不在四层栈内,作对照) | AHE [12]、AgentDebug [13]、Autodata [14]、AEVO [15] |

> **工作论点(详见 `.researcher/thesis.md`)**:部署后 agent 改进的瓶颈不是模型能力或评估精度,而是"轨迹流 → 偏好数据"之间缺失的桥;这座桥最好按上述四层栈搭建,且 L1 分诊应以非语义、规则化 / 小代理检测器为主,而非逐轨迹 LLM-as-Judge。

## 论文收录 (Paper Collection)

> 完整索引见 [`papers/README.md`](papers/README.md);论文间关系与阅读优先级见 [`notes/00_research_landscape.md`](notes/00_research_landscape.md)。

| # | 论文 | 研究层 | 优先级 | 笔记 |
|---|------|--------|--------|------|
| 1 | Signals: Trajectory Sampling and Triage | L1 筛选分诊 | P0 | ✅ |
| 2 | AgentHER: Hindsight Experience Replay | L2 数据转化 | P0 | ✅ |
| 3 | TSR: Trajectory-Search Rollouts | L2 数据转化 | P2 | ✅ |
| 4 | AgentTrace: Structured Logging | L0 基础设施 | P0 | ✅ |
| 5 | Breaking the Observability Tax | L0 基础设施 | P1 | 🟡 |
| 6 | AgentSeer: Agentic Vulnerabilities | L1 筛选分诊 | P1 | 🔲 |
| 7 | Agent-as-a-Judge | 评估范式 | P2 | 🟡 |
| 8 | TIDE / TRACE | 评估范式 | P2 | 🟡 |
| 9 | Trajectory Guard: Sequence-Aware Anomaly Detection | L1 筛选分诊 | P1 | ✅ |
| 10 | Sentinel / PhantomPolicy: Counterfactual KG Verifier | L1 筛选分诊 | P1 | ✅ |
| 11 | Near-Miss: Latent Policy Failure Detection | 评估 / L1 筛选 | P1 | ✅ |
| 12 | Agentic Harness Engineering (observability-driven evolution) | off-axis | P2 | ✅ |
| 13 | Where LLM Agents Fail and How They Learn (AgentDebug) | off-axis | P2 | ✅ |
| 14 | Autodata: automatic data scientist | off-axis | P2 | ✅ |
| 15 | AEVO: Harnessing Agentic Evolution | off-axis | P2 | ✅ |
| 16 | SWE-PRM: Course-Correcting SWE Agents with PRMs | off-axis (in-flight 过程监督) | P2 | ✅ |

> **图例:** ✅ 全文已读 | 🟡 摘要/综述已读 | 🔲 待读
