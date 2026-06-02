# 论文阅读笔记：《TSR: Trajectory-Search Rollouts for Multi-Turn RL of LLM Agents》

> **Created:** 2026-04-26  
> **Last Updated:** 2026-04-26  
> **状态：** ✅ 已读  
> **arXiv:** [2602.11767](https://arxiv.org/abs/2602.11767)  
> **作者:** Aladin Djuhera, Swanand R. Kadhe, Farhan Ahmed, Heiko Ludwig, Holger Boche  
> **优先级：** 🟢 P2 — 数据转化层 (Layer 2)，训练期侧  
> **与 Signals 的关系：** 互补 — Signals 解决部署后轨迹筛选，TSR 解决训练期轨迹筛选。

---

## 1. 核心问题

> 多轮 RL 训练中的三大挑战：
> 1. **稀疏/延迟奖励**：Agent 做对了前 N-1 步但最后一步失败 → 全程奖励为 0
> 2. **不可逆陷阱**：早期错误导致后续无法挽回（如 Sokoban 推箱子推到死角）
> 3. **多样性不足**：Agent 反复采样相同成功轨迹 → 优势估计归零 → 梯度停滞

> **核心思路**：将测试时缩放（test-time scaling）的搜索思想搬到训练时的 rollout 生成阶段。

## 2. 关键方法

### 2.1 POMDP 建模

多轮 Agent 交互建模为 POMDP：ℳ = ⟨U, S, A, O, P, R⟩
- 每轮产生 (action, observation, reward)
- 轨迹 τ = 完整的交互序列
- 目标：最大化累积奖励 R(τ) = Σr_t

### 2.2 TSR 框架

**不改变优化目标，只改变 rollout 的生成方式**（optimizer-agnostic）：

```
对每个 prompt/task:
  1. 在每轮 t，采样 M 个候选动作 A_t（而非仅 1 个）
  2. 用评分函数 S(τ<t, a_t, o_t) 评估每个候选
  3. 按搜索策略选择最优动作/保留最优分支
  4. 生成的高质量轨迹用于标准 PPO/GRPO 更新
```

### 2.3 三种搜索策略

| 策略 | 原理 | 优劣 |
|------|------|------|
| **Best-of-N** | 独立采样 N 条完整轨迹，取最高分 | 简单但不影响中间决策 |
| **Beam Search** | 每轮保留 B 条最优前缀，逐轮扩展 | **最强**，能从早期错误恢复 |
| **Shallow Lookahead** | 每轮向前看 D 步后再选择 | 比 greedy 强但不如 beam |

### 2.4 Instance Filtering（任务多样性保障）

- 采样 P=16 个不同任务 × L=16 条 rollout/任务
- 按奖励标准差排序，只保留 top-25% 不确定性最高的任务组
- **与 TSR 互补**：TSR 保证 rollout 质量（exploitation），Filtering 保证任务多样性（exploration）

## 3. 实验结果

### 3.1 主要成绩

| 环境 | 基线→TSR(Beam) | 提升 |
|------|---|---|
| Sokoban (3B) | 43.9% → 52.3% | +8.4% |
| FrozenLake (3B) | 48.7% → 60.7% | +12.0% |
| WebShop (3B) | 32.0% → 47.0% | **+15.0%** |

- **0.5B 模型 + TSR 超越 GPT-4o**：Sokoban +10.6%，FrozenLake +3.4%
- 搜索预算边际收益递减：B=1→2 提升最大（+1.7%），之后收益递减

### 3.2 训练稳定性

- 梯度范数无异常尖峰 → 有效防止 mode collapse（Echo Trap）
- 熵平滑衰减 → 探索→利用转换正常
- 推理效率提升：更短回复 + 更少交互轮次

## 4. 当前阶段定位 ⚠️

> **TSR 是训练侧 RL 方法（rollout 阶段加搜索），与以推理侧观测/分诊为主的 agentic harness 定位距离最远。**

- TSR 优化的是**训练时**轨迹质量，不影响部署模型的推理行为
- 若工程主线短期不自建 RL 训练流水线 → TSR 不在落地范围内
- **可借鉴的薄概念**：
  - Beam Search "保留 B 条最优前缀逐轮扩展"的思想 → 可类比为推理侧的编排式多步执行（多分支评估），但本质完全不同
  - Instance Filtering 的"按奖励标准差排序"→ 可借鉴为分诊环节中"用历史成功率方差给任务优先级"的简单启发式
- **不要照搬**：评分函数 S(·)、search budget、PPO/GRPO 集成——这些是训练管道概念，推理侧 harness 现阶段无对应模块

## 5. 批判性阅读（紧凑版）

> 鉴于 TSR 当前不是落地优先级，仅记录在写综述时需要引用的关键质疑：

1. **评分函数 S(·) 的通用性极弱** — 论文中 Sokoban 用距离启发式、FrozenLake 用网格距离、WebShop 用进度评分。这些都是**任务特定的可观察 proxy**，在通用 Agent 场景（CRM、代码、自由 web 自动化）几乎不可得。论文未讨论 reward-model-based S 的实施
2. **"0.5B + TSR > GPT-4o" 是不公平对比** — 0.5B 模型经过任务专门 RL 训练，GPT-4o 是 zero-shot。这是把"专家小模型 vs 通才大模型"包装成"TSR 让小模型超越大模型"
3. **One-time compute increase 措辞误导** — Beam B=8 意味着每轮 8× rollout 计算，乘以平均 turn 数后实际开销并不"one-time"。论文未给训练 wall-clock 对比
4. **3 个环境都是 toy benchmark** — Sokoban / FrozenLake 是 grid puzzle，WebShop 是受限购物。代表不了现代复杂 agentic harness（多模态、长 horizon、模糊 reward）
5. **与 MCTS 的关系刻意回避** — TSR 本质上是"无 value backup 的简化 MCTS"，但论文不把 MCTS 列为 baseline。**跳过最强对照组**
6. **众多并行工作未实证对比** — AgentLightning、RAGEN、GFPO、StarPO、RAFT 都被引用为相关工作，但**没有一个跑在同一 benchmark**上做端到端对比。所谓"互补"是声明而非验证
7. **Echo Trap / mode collapse 的"防止"主张** — 论文用"梯度范数无尖峰 + 熵平滑衰减"作为证据，但**没有定量定义 mode collapse 的发生阈值**。这种定性观察等于说"看起来稳定"

## 6. 与其他论文的关系

- **vs AgentHER [2]**：训练期 vs 部署后，理论互补（"in-situ 优化 + post-hoc 回收"）。但**两篇都是训练侧** —— 对以推理侧观测为主的定位都不直接落地
- **vs Signals [1]**：Signals 是部署后的 model-free 信号筛选；TSR 是训练时的 model-heavy 搜索。**几乎正交**——综述时把它们放在不同象限
- **vs AgentTrace [4] / Obs Tax [5]**：基本无关——TSR 是训练阶段方法，不依赖运行时 trace

**综述引用指引：** TSR 作为**"训练时 rollout 优化"代表论文**讨论，定位为"未来训练扩展时可参考"，不参与综述主线（推理/观测/分诊/评估）。

## 7. 开放问题（保留）

- [ ] 评分函数 S(·) 通用化：能否用 reward model 替代 task-specific heuristic？
- [ ] 与现代 RL 框架（AgentLightning、RAGEN）的端到端对比缺失
- [ ] 真实 wall-clock 训练成本未刻画
