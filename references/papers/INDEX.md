# 学术论文索引

> 学术 PDF 不入 git（gitignored）。本文件维护 arXiv ID + 一句话定位，配套 `scripts/fetch-papers.sh`（待建）按 ID 重新下载到 `papers/`。

---

## 已深读论文（6 篇）

| # | 论文 | arXiv | 一句话定位 | 笔记 |
|---|------|-------|-----------|------|
| 1 | **Signals: Trajectory Sampling and Triage for Agentic Interactions** | [2604.00356](https://arxiv.org/abs/2604.00356) | 首次把"无 model call 的轨迹分诊"系统化的实证论文。τ-bench 上 82% informativeness、1.52× 效率，且对成功轨迹仍 66.7% 命中。**KWeaver Triage Agent 的输入信号格式直接借鉴这里** | `notes/01_signals_trajectory_triage.md` |
| 2 | **AgentHER: Hindsight Experience Replay for LLM Agent Trajectory Relabeling** | [2603.21357](https://arxiv.org/abs/2603.21357) | 把 RL 时代的 HER 思想首次系统化操作到自然语言 Agent。WebArena +7.1-11.7 pp。**思想可重定向到 BKN 自演化的 patch 生成机制** | `notes/02_agenther_hindsight_relabeling.md` |
| 3 | **TSR: Trajectory-Search Rollouts for Multi-Turn RL of LLM Agents** | [2602.11767](https://arxiv.org/abs/2602.11767) | 训练时 rollout 加搜索避免 mode collapse。**训练侧、当前 KWeaver 不落地** | `notes/03_tsr_trajectory_search_rollouts.md` |
| 4 | **AgentTrace: A Structured Logging Framework for Agent System Observability** | [2602.10133](https://arxiv.org/abs/2602.10133) | 把工业实践（Langfuse/Phoenix）学术化为三 surface schema。**借鉴命名学，不照搬实现** | `notes/04_agenttrace_structured_logging.md` |
| 5 | **Agent-as-a-Judge: Evaluate Agents with Agents** | [2410.10934](https://arxiv.org/abs/2410.10934) | 把 LLM-as-Judge 升级为带工具/记忆/规划的 agentic 评估器。DevAI 上 86-92% align with human。**关键发现：Memory + Planning 模块有害——直接定义了 Triage Agent 内部架构** | `notes/07_agent_as_a_judge.md` |
| 5b | **A Survey on Agent-as-a-Judge** | [2601.05111](https://arxiv.org/abs/2601.05111) | Agent-as-a-Judge 综述，系统化三阶段成熟度模型 | `notes/07_agent_as_a_judge.md` |
| 6 | **TIDE: Trajectory-based Diagnostic Evaluation of Test-Time Improvement in LLM Agents** | [2602.02196](https://arxiv.org/abs/2602.02196) | 把 SR 拆成 AUV/LR/MI 三维动力学指标。**LR 是 Signals.Loop 的理论化版本** | `notes/08_tide_trace_diagnostics.md` |

---

## 阻塞待补论文（3 篇）

| # | 论文 | 状态 | 期望 ID |
|---|------|------|---------|
| 7 | **Breaking the Observability Tax** | ❌ 未下载 | DOI [10.1109/ACCESS.2026.3675074](https://doi.org/10.1109/ACCESS.2026.3675074)（IEEE Access CC-BY，应可免费下载）|
| 8 | **AgentSeer: Evaluating Agentic Vulnerabilities** | ❌ 本地 PDF 是 Short-PHD（2504.02873），与 AgentSeer 无关 | 真实 arXiv ID 待确认 |
| 9 | **TRACE: Trajectory-Aware Comprehensive Evaluation for Deep Research Agents** | ❌ 本地 PDF 是 math.CV 论文（2602.05428），与 TRACE 无关 | 真实 arXiv ID 待确认 |

---

## 学术论文与 KWeaver 飞轮环节的对应

| 飞轮环节 | 主要服务论文 |
|---------|-----------|
| 燃料采集（双轨 Trace 字段）| AgentTrace [4]、AgentSeer [8]（待补）|
| 燃料质量门控（信号分诊）| **Signals [1]**、AgentSeer [8]（待补）|
| 燃料消化（trace → BKN/Context patch）| **AgentHER [2] 思想重定向**、TIDE [6] |
| 燃料经济学（成本控制）| Breaking Obs Tax [7]（待补）|
| 评估范式（精评后端）| **Agent-as-a-Judge [5]**、TIDE [6]、TRACE [9]（待补）|
| 训练侧（远期）| AgentHER [2]、TSR [3]——当前 KWeaver 不落地 |

---

## 重新获取 PDF

PDF 拉取由**外部 researcher 项目**负责，按本文件维护的 arXiv ID 自动同步到本项目的 `papers/`（gitignored）。本项目不实现 fetch 脚本，只维护索引。

参考 arXiv ID 列表：
```
2604.00356  Signals
2603.21357  AgentHER
2602.11767  TSR
2602.10133  AgentTrace
2410.10934  Agent-as-a-Judge (原始)
2601.05111  Agent-as-a-Judge (综述)
2602.02196  TIDE
```
