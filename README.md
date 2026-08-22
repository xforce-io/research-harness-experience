# Agent Harness Research — experience loop

本仓是 `research-harness-trace` 的**就地升级**（同一 GitHub 仓库对象，star / issue / 历史保留）。原 mandate 是部署后 tracing → 分诊 → 重打标；现升级为 **experience 闭环**：捕获交互流 → 更新（权重或 harness）→ 用对齐部署的评价验收。

不绑定任何具体产品或厂商。原 19 篇笔记仍是捕获章，并未作废。

## 目录结构 (Directory Structure)

*   `papers/`: 论文索引(`papers/README.md`,按 arXiv ID 维护)。**PDF 原文件不纳入 Git**(见 `.gitignore`)。
*   `notes/`: 论文 7-Block 深读笔记。`00_research_landscape.md` 是全局研究地图(论文间关系 + 四层栈定位 + 阅读优先级)。
*   `.researcher/`: 自动化 researcher 的配置——`thesis.md`(工作论点)与 `project.yaml`(研究问题、纳入/排除标准、检索源)。

## 研究主轴: experience 闭环

| 节 | 关注点 | 代表论文 |
|----|--------|---------|
| **流（捕获）** | schema、轻量分诊、成功轨迹隐性摩擦、哨兵采样 | Signals [1]、AgentTrace [4]、Near-Miss [11] |
| **更新** | hindsight / 过程信用 → 权重；或编辑 harness 不改 target | AgentHER [2]、Harness-R1 [19]、TRACE turn-credit [23] |
| **评价** | 搜索与终评分离、同预算采样对照、反馈自审计 | AHE [12]、AI4AI [20]、Rethinking harness eval [21] |

旧四层栈 L0–L3 是「流 + 更新」的展开，现收进闭环，不另立 evolution 仓。

## 论题

- 改进介质是自身交互流，不是更多人类数据或新 trainer。
- 前线轻量信号；更新可走权重或 harness；评价必须拆掉采样假象。
- 完整论点与可证伪条件见 [`.researcher/thesis.md`](.researcher/thesis.md)。

## 论文收录 (Paper Collection)

> 完整索引见 [`papers/README.md`](papers/README.md);论文间关系与阅读优先级见 [`notes/00_research_landscape.md`](notes/00_research_landscape.md)。  
> **Last Updated:** 2026-08-05

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
| 17 | MASPrism: Lightweight Failure Attribution (prefill-stage signals) | 评估范式 (within-trace attribution) | P2 | ✅ |
| 18 | AgenTracer: Who Is Inducing Failure in MAS (trained 8B tracer) | 评估范式 (within-trace attribution) | P2 | ✅ |
| 19 | Harness-R1: Learning to Edit Executable Runtime Harnesses | 更新（harness） | P2 | ✅ |
| 20 | AI4AI at Test-Time (strong-to-weak harness) | 评价 / 更新 | P1 | ✅ 迁入 |
| 21 | Rethinking the Evaluation of Harness Evolution | 评价 | P0 | ✅ 迁入 |
| 22 | Sample-efficient learning from agent experience | 更新（蒸馏） | P2 | ✅ 迁入 |
| 23 | TRACE: Turn-level reward assignment | 更新（过程信用） | P1 | ✅ 迁入 |

> **图例:** ✅ 全文已读 | 🟡 摘要/综述已读 | 🔲 待读 | 迁入 = 从重叠本地 topic 并入，非新开仓

## 深度入口

- 综合地图与论文关系：[`notes/00_research_landscape.md`](notes/00_research_landscape.md)
- 源索引：[`papers/README.md`](papers/README.md)
- 工作论点与项目配置：[`.researcher/thesis.md`](.researcher/thesis.md)
