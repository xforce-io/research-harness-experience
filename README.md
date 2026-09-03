# Experience

Silver & Sutton《Welcome to the Era of Experience》把改进介质从人类数据换成 agent 与环境的交互流；姚顺雨《The Second Half》把下半场的瓶颈从新 trainer 换成任务与评价。本仓沿这两点深耕 **experience 闭环**：捕获 → 分流后的更新（A 改权重 **或** B 改 harness，并列可叠）→ 评价门。

不绑定任何具体产品或厂商。

## 目录结构 (Directory Structure)

*   `papers/`: 论文索引(`papers/README.md`,按 arXiv ID 维护)。**PDF 原文件不纳入 Git**(见 `.gitignore`)。
*   `notes/`: 论文 7-Block 深读笔记。`00_research_landscape.md` 是全局研究地图(论文间关系 + 闭环定位 + 阅读优先级)。
*   `.researcher/`: 自动化 researcher 的配置——`thesis.md`(工作论点)与 `project.yaml`(研究问题、纳入/排除标准、检索源)。

## 研究主轴: experience 闭环

| 节 | 关注点 | 代表论文 |
|----|--------|---------|
| **G1 捕获** | L0 schema、L1 轻量分诊、成功轨迹隐性摩擦 | Signals [1]、AgentTrace [4]、Near-Miss [11] |
| **G2 更新 A** | 能力/偏好错 → 改权重 | AgentHER [2]、TRACE [23]、Experience distillation [22] |
| **G3 更新 B** | 基板/runtime 错 → 改 harness | Harness-R1 [19]、Recuris [24]、SKILLSTATE [25]、AHE [12] |
| **G4 分流** | A 与 B 并列可叠；in-flight 非更新 | [2] vs [19]；SWE-PRM [16] |
| **G5 评价门** | 搜索≠终评、同预算采样、反馈自审计 | Rethinking harness eval [21]、AHE [12]、Recuris [24] |

## 论题

生产 agent 的改进介质是自身交互流，不是更多人类数据或新 trainer。更新是并列的两条路径——A 改权重、B 改 harness——先分流再动手，可叠。评价横切两支，in-flight 纠偏不是进化。完整架构与可证伪条件见 [`.researcher/thesis.md`](.researcher/thesis.md)。

## 论文收录 (Paper Collection)

> 完整索引见 [`papers/README.md`](papers/README.md);论文间关系与阅读优先级见 [`notes/00_research_landscape.md`](notes/00_research_landscape.md)。  
> **Last Updated:** 2026-09-01

| # | 论文 | 研究层 | 优先级 | 笔记 |
|---|------|--------|--------|------|
| 25 | SKILLSTATE: Scalable Long-Horizon Agent Skills | 更新 B（执行底物）/ L0 | P1 | ✅ |
| 24 | Recuris: Recursive Experiential–Working Memory Evolution | 更新 B / 评价门 | P1 | ✅ |
| 23 | TRACE: Turn-level reward assignment | 更新 A | P1 | ✅ |
| 22 | Sample-efficient learning from agent experience | 更新 A | P2 | ✅ |
| 21 | Rethinking the Evaluation of Harness Evolution | 评价 | P0 | ✅ |
| 20 | AI4AI at Test-Time (strong-to-weak harness) | 更新 B / 评价门 | P1 | ✅ |
| 19 | Harness-R1: Learning to Edit Executable Runtime Harnesses | 更新 B | P0 | ✅ |
| 18 | AgenTracer: Who Is Inducing Failure in MAS (trained 8B tracer) | 评估范式 (within-trace attribution) | P2 | ✅ |
| 17 | MASPrism: Lightweight Failure Attribution (prefill-stage signals) | 评估范式 (within-trace attribution) | P2 | ✅ |
| 16 | SWE-PRM: Course-Correcting SWE Agents with PRMs | 非更新（in-flight） | P2 | ✅ |
| 15 | AEVO: Harnessing Agentic Evolution | 更新 B | P2 | ✅ |
| 14 | Autodata: automatic data scientist | 更新 B | P2 | ✅ |
| 13 | Where LLM Agents Fail and How They Learn (AgentDebug) | 非更新（in-flight） | P2 | ✅ |
| 12 | Agentic Harness Engineering (observability-driven evolution) | 更新 B / 评价门 | P2 | ✅ |
| 11 | Near-Miss: Latent Policy Failure Detection | 评估 / L1 筛选 | P1 | ✅ |
| 10 | Sentinel / PhantomPolicy: Counterfactual KG Verifier | L1 筛选分诊 | P1 | ✅ |
| 9 | Trajectory Guard: Sequence-Aware Anomaly Detection | L1 筛选分诊 | P1 | ✅ |
| 8 | TIDE / TRACE | 评估范式 | P2 | 🟡 |
| 7 | Agent-as-a-Judge | 评估范式 | P2 | 🟡 |
| 6 | AgentSeer: Agentic Vulnerabilities | L1 筛选分诊 | P1 | 🔲 |
| 5 | Breaking the Observability Tax | L0 基础设施 | P1 | 🟡 |
| 4 | AgentTrace: Structured Logging | L0 基础设施 | P0 | ✅ |
| 3 | TSR: Trajectory-Search Rollouts | 更新 A | P2 | ✅ |
| 2 | AgentHER: Hindsight Experience Replay | 更新 A | P0 | ✅ |
| 1 | Signals: Trajectory Sampling and Triage | L1 筛选分诊 | P0 | ✅ |

> **图例:** ✅ 全文已读 | 🟡 摘要/综述已读 | 🔲 待读

## 深度入口

- 综合地图与论文关系：[`notes/00_research_landscape.md`](notes/00_research_landscape.md)
- 源索引：[`papers/README.md`](papers/README.md)
- 工作论点与项目配置：[`.researcher/thesis.md`](.researcher/thesis.md)
