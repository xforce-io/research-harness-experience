---
zone: active
tags: [capture_stream]
pin: false
score: 0.20305997552019583
dwell: 1
---
# 论文阅读笔记：《AgentSeer: Evaluating Agentic Vulnerabilities》

**分类：** 🟡 P1 — 筛选分诊层 (Layer 1)，非语义检测  
**状态：** 🔲 待读  
**与 Signals 的关系：** 验证了 Signals 中"执行层信号优于语义层分析"的前瞻性主张。

---

## 1. 核心问题

> 传统的对话安全评测（如 adversarial prompting）只在语义层工作，无法发现 Agent 独有的漏洞——那些通过工具调用组合触发的隐性安全问题。如何检测这类"仅靠语言测不出来"的缺陷？

## 2. 关键方法

* **动作-组件图（Action-Component Graphs）**：
  - 将 Agent 的执行过程分解为图结构
  - 节点 = 工具调用 / 组件
  - 边 = 调用关系 / 数据流
  - [ ] 待补充：图的具体构建方法

* **拓扑结构异常检测**：
  - 通过分析执行流的拓扑特征（而非语义内容）来发现漏洞
  - 例：异常的工具调用组合、不符合预期的执行路径
  - [ ] 待补充：具体使用了哪些图分析算法？

## 3. 工程落地启示

* **call-chain 就是天然的 Action-Component Graph**：
  - 一个 observability 双链（call-chain + evidence-chain）平台天然记录了 Action 之间的调用关系（call-chain）
  - 可以直接在 call-chain 数据上运行 AgentSeer 的拓扑分析
* **安全审计用例**：
  - 在分诊环节中增加"拓扑异常检测"维度
  - 不仅检测"失败"，还检测"成功但路径异常"的轨迹（潜在安全隐患）
* **待确认的工程前提**：
  - [ ] call-chain 粒度是否足够细，能否区分不同的工具调用组合？
  - [ ] Action-Component Graph 的构建是实时的还是批处理的？
  - [ ] 拓扑异常的 false positive 率如何？

## 4. 关键摘录 & 笔记

> （阅读后补充）

## 5. 开放问题

- [ ] AgentSeer 的 Action-Component Graph 是否可以与 Signals 的三层信号分类学合并？
- [ ] 在编排式多步执行场景下，"预期拓扑"可以从编排定义中自动推导，用作 baseline
- [ ] 拓扑检测的计算复杂度是否适合在线场景？还是只能做离线批量分析？
