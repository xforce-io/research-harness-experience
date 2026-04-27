# Research-Agent-Triage

这个目录用于研究、设计和实验 KWeaver TraceAI 相关的核心能力，包括 Agent 轨迹可观测性（Observability）、信号过滤（Signal Triage）以及数据飞轮（Data Flywheel）建设。

## 目录结构 (Directory Structure)

*   `papers/`: 论文索引与本地存放目录。**PDF 原文件不纳入 Git**（见 `.gitignore`），通过 `papers/README.md` 索引管理。
*   `notes/`: 论文阅读笔记、方案设计思路和调研总结。`00_research_landscape.md` 是全局研究地图。
*   `experiments/`: 用于概念验证（PoC）的代码片段、脚本和探针示例。
*   `data/`: 脱敏后的 Agent 交互轨迹样本、Mock 数据和评测日志。

## 核心研究方向 (Core Research Topics)

1. **轻量级信号分流 (Lightweight Signal Triage)**
   * 研究如何不依赖昂贵的 LLM 评估，仅通过行为模式、正则匹配和拓扑图分析，低成本找出"高复盘价值"的轨迹。
   * 构建交互层（Interaction）、执行层（Execution）和环境层（Environment）的三维探针体系。
2. **后见之明重打标 (Hindsight Relabeling & Data Flywheel)**
   * 研究如何将失败或带有隐性摩擦的真实轨迹，通过重写转化为高质量的偏好数据（Preference Data），服务于后续的模型对齐（RLHF/DPO）。
3. **低成本遥测 (Topology-Aware Telemetry)**
   * 探索动态采样率和拓扑感知监控，降低 Agent 系统的 Observability Tax。

## 论文收录 (Paper Collection)

> 详见 [`papers/README.md`](papers/README.md) 和 [`notes/00_research_landscape.md`](notes/00_research_landscape.md)  
> **Last Updated:** 2026-04-27

| # | 论文 | 研究层 | 优先级 | 笔记状态 |
|---|------|--------|--------|---------|
| 1 | Signals: Trajectory Sampling and Triage | 筛选分诊 | P0 | ✅ 已读 |
| 2 | AgentHER: Hindsight Experience Replay | 数据转化 | P0 | ✅ 已读 |
| 3 | TSR: Trajectory-Search Rollouts | 数据转化 | P2 | ✅ 已读 |
| 4 | AgentTrace: Structured Logging | 基础设施 | P0 | ✅ 已读 |
| 5 | Breaking the Observability Tax | 基础设施 | P1 | 🟡 摘要已读 |
| 6 | AgentSeer: Agentic Vulnerabilities | 筛选分诊 | P1 | 🔲 待读 |
| 7 | Agent-as-a-Judge | 评估范式 | P2 | 🟡 综述已读 |
| 8 | TIDE / TRACE | 评估范式 | P2 | 🟡 摘要已读 |
| 9 | Trajectory Guard: Sequence-Aware Anomaly Detection | 筛选分诊 | P1 | ✅ 已读 |
| 10 | Sentinel / PhantomPolicy: Counterfactual KG Verifier | 筛选分诊 | P1 | ✅ 已读 |
