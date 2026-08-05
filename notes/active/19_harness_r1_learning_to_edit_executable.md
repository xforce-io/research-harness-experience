---
zone: active
tags: []
pin: false
score: 0.41395348837209306
dwell: 1
---
# Harness-R1: Learning to Edit Executable Runtime Harnesses from Agent Failure Trajectories

> Topic integration note derived from Library read artifact `.researcher-workspace/library/papers/paper_arxiv_2608_02276/reads/read_paper_arxiv_2608_02276.md`.

## Library read

---
kind: library-read-identity
authors: ["Shuai Shao","Kangning Zhang","Qingyao Li","Shijian Wang","Hao Wang","Wenxiang Jiao","Yuan Lu","Yi Guo","Weiwen Liu","Weinan Zhang"]
source_id: "arxiv:2608.02276"
source_url: "https://arxiv.org/abs/2608.02276"
pdf_url: "https://arxiv.org/pdf/2608.02276"
---
> 不改模型权重、也不靠大模型 prompt 猜补丁，而是训练一个 9B 专用编辑器，从失败轨迹生成可执行 runtime patch，再用重跑结果做 RL 奖励——让"改 harness"本身成为可学习的能力。

## Essence

**问题**：Agent 部署后不断产生交互轨迹，但行为通常固化。改进有两条路——更新模型权重，或优化 harness（上下文构造、工具中介、动作校验、恢复逻辑）。已有 harness 编辑方法要么用固定规则（Self-Refine 等不稳定甚至有害），要么让大模型 propose 补丁但不更新编辑器参数，无法从实际重跑效果中学习。

**做法**：Harness-R1 训练一个独立的 9B "harness engineer"。它读取一批目标 agent 的失败轨迹，生成一个可执行 patch（四个生命周期 hook：episode 初始化、决策前提示、动作前拦截、反馈后恢复）。patch 安装后冻结的目标 agent 重跑同一批任务，实际性能变化作为奖励，用 GRPO 更新 engineer 权重。冷启动用 GPT-5.5 生成的 SFT 数据初始化，再在线 RL 优化。

**证据**：在 WebShop / ALFWorld / DBBench 三个 benchmark 上，vanilla Qwen3.5-9B 成功率从 44.3% 提升到 53.6%（+9.3 pp），超过所有 frontier 模型 prompted editor（最强 GLM-5.2 仅 48.8%）。

**边界**：奖励信号绑定同批任务（transductive），无跨批次 patch 记忆或迭代细化；仅验证了 3 个 benchmark；patch 质量依赖 failure packet 的信息密度和 hook 接口的表达力。

## Claims

- 将 failure-conditioned、lifecycle-wide 的 harness 编辑形式化为一个在线 RL 问题，训练专用 engineer 而非更新目标 agent 权重，是可行且有效的范式 [§3.1, §5]。
- 基于结果的后训练（outcome-grounded post-training）比基于文本合理性的编辑更可靠：9B 的 outcome-trained engineer 超过 397B 及更大的 frontier 模型 prompted editor [§4.2, Table 1]。
- 固定的 harness 策略（Self-Refine、ReAct、Reflection）不 uniformly beneficial：Self-Refine 在三个 benchmark 上均降低奖励（平均 -2.5 pp）[§4.2, Table 1]。
- Cold-start SFT 初始化 + 在线 GRPO 的两阶段训练优于仅 SFT：outcome-trained engineer 比 supervised-only engineer 高 7.1 pp [§4.2]。
- Harness 编辑可与 target agent 改进 co-evolve：在 target SFT 后（59.2%），target-specific engineer 仍可提升 5.0 pp 至 64.2% [§4.2, Table 1]。
- 学习到的编辑策略可跨 20 个未见目标模型迁移，每个目标平均提升 7.06 pp，56/63 目标-benchmark 组合改善 [§4.3, Figure 3]。
- 从仅 10 条失败轨迹生成的 patch 可泛化到 1,270 条 held-out 任务，提升 8.9±1.5 pp，而 frontier 模型在同样设置下平均为负 [§4.4, Figure 4a]。
- 不同环境的 dominant intervention point 不同：WebShop 靠 pre-action mediation，ALFWorld 靠 post-feedback recovery [§4.5, Figure 4b]。

## Assumptions

- 目标 agent 的失败模式可以通过一组可执行 hook（四个生命周期位置）有效干预——即 harness 是 agent 能力的主要瓶颈而非模型能力本身。
- 同一批任务的 before/after reward 差异可归因于 patch 的效果，而非环境随机性或目标 agent 的采样方差。
- Failure packet（压缩的失败轨迹摘要）包含足够信息让 engineer 生成泛化性 patch，无需访问完整轨迹。
- Patch 的 hook 接口（`on_init` / `make_pre_hint` / `on_before_action` / `on_post_step`）覆盖了三个 benchmark 中所有有价值的干预维度。
- GRPO 的 group-relative 优势（K=8 候选）能为 patch 质量提供有效梯度信号，尽管 reward 是稀疏的 binary success。

## Method

**Before vs after 对比：**

| 维度 | 已有 harness 编辑 | Harness-R1 |
|------|------------------|------------|
| 编辑器 | 固定规则或 prompted frontier model | 独立 9B engineer，RL 后训练 |
| 反馈 | 文本合理性 / 候选筛选 | 冻结 target 重跑的实际 reward 变化 |
| 训练信号 | 无参数更新 | GRPO 优化 engineer 权重 |
| 覆盖面 | 单一 artifact（prompt/skill） | 四个生命周期 hook 的协调编辑 |

**输入**：冻结目标 agent A 在一批任务 B 上的失败轨迹，经确定性 extractor 压缩为 failure packet `s_B`（包含任务约束、动作-观测摘录、结果、环境状态）。

**核心计算**：
1. **Cold-start SFT**：GPT-5.5 从失败包生成候选编辑响应，经过验证（可执行、完整、非回归）后保留 ≤1 条/包，形成 ~877 条 SFT 数据，用 teacher-forced next-token prediction 初始化 9B engineer。
2. **Online GRPO**：对每个 failure packet，当前策略采样 K=8 个候选 patch；每个 valid patch 独立安装到冻结 target 并重跑全批任务；reward 为全批平均 reward 变化 `Δ_B(P) = (1/n)Σ(R_i^P - R_i^0)`，invalid/no-op 为 0；组内归一化为优势 `Â_k = (r_k - μ_B) / σ_B`；用 clipped surrogate 优化 token-averaged objective，仅更新 engineer 参数 θ。
3. **Patch 结构**：四个 hook 的可执行 Python 代码——`on_init`（设置初始上下文）、`make_pre_hint`（决策前注入提示）、`on_before_action`（动作前拦截/重写/veto）、`on_post_step`（反馈后恢复/调度下一动作）。

**输出**：训练后的 harness engineer，可对任意目标 agent 的失败轨迹生成可执行 patch。

## Eval

- **Benchmarks**：WebShop（500 tasks，web 导航购买）、ALFWorld（500 tasks，6 类家务，文本具身）、DBBench（300 tasks，SQL 数据库操作）。
- **Baselines**：无修改目标 agent；prompt-based 策略（ReAct / Self-Refine / Reflection）；6 个 frontier 模型 prompted as harness editor（Qwen3.5-397B / GLM-5.2 / Kimi-K2.6 / DeepSeek-V4-Pro / Gemini-3.5-Flash / GPT-5.5）；supervised-only engineer；outcome-trained Harness-R1。
- **Metrics**：Task success rate（三个 benchmark）+ WebShop shaped score；equal-weight 三 benchmark 平均。
- **Additional evals**：target agent SFT 后的 co-evolution（Table 1 最后两行）；21 个目标模型的 cross-model 迁移（Figure 3 / Table E.1）；10-failure held-out 泛化（Figure 4a / Table F.1）；lifecycle position ablation（Figure 4b / Table G.1）。

## Weaknesses

- **Same-batch reward 的过拟合风险**：reward 仅来自 patch 训练时所见的那批任务，没有 held-out reward 纳入训练信号。虽然 held-out 评测显示泛化，但训练目标本身不惩罚 unseen-task regression——论文未报告训练批次 vs held-out 的 reward gap 趋势 [§3.1, §5]。
- **Hook 接口的表达力边界未被探测**：四个 lifecycle hook 是预设的，论文未讨论是否存在无法通过这四个位置表达的常见失败模式（如跨 episode 的长期记忆策略、多 agent 协调），也未与更丰富的接口做对比 [§3.1, Appendix C]。
- **GRPO 的 reward 稀疏性未被分析**：三个 benchmark 中两个是 binary success，K=8 候选中可能多数为 0 reward（invalid/no-op/negative），group-relative 优势的方差和有效梯度信号质量未报告。论文未给出 valid patch 比例或 reward 分布的统计 [§3.2, Algorithm 1]。
- **SFT 数据的 teacher 依赖未被充分讨论**：877 条 SFT 数据由 GPT-5.5 生成并过滤，但未报告 teacher 的通过率、分布偏差，或不同 teacher 对 RL 阶段收敛的影响。Cold-start 质量是整个 pipeline 的前提，但敏感性分析缺失 [§3.2, Appendix B.1]。
- **Frontier 模型 baseline 的 prompt 可能未充分优化**：论文让 frontier 模型 "prompted as harness engineer"，但未公开这些 prompt 是否经过迭代优化。9B trained engineer vs 397B zero-shot prompted model 的比较可能不公平——如果 frontier 模型也获得 few-shot 或结构化 prompt，gap 可能缩小 [§4.1, §4.2]。
- **计算成本未报告**：每次 GRPO 迭代需要 K=8 个 patch 各重跑全批任务（包括已成功的任务），在线 RL 的总 rollout 次数和 GPU-hour 未披露，难以评估方法的实际部署门槛 [§3.2, Appendix B.2]。
- **Reflection 的 comparison 不公平但未充分标注**：Reflection 使用 two-episode success@2 协议（55.8%），与其他 single-episode 行不可直接比较，论文虽提及但仍放在同一 Table 1 中，读者容易误读 [§4.1, Table 1]。

## Relations

- **builds-on** GRPO / DeepSeekMath [high]：Harness-R1 的在线 RL 阶段直接使用 group-relative policy optimization（K=8 候选，组内归一化优势），是 GRPO 在 harness 编辑这一新问题上的应用 [§3.2, 引用 Shao et al., 2024]。
- **competes-with** Meta-Harness / AutoHarness / AHE [med]：这些系统也做 harness 优化（搜索/综合/迭代编辑），但 proposer 参数不更新——Harness-R1 的核心区分点是将学习目标从"产出 harness"移到"编辑策略本身"，用 RL 后训练 proposer [§2, Related Work]。
- **competes-with** HarnessX [med]：HarnessX 也使用 cross-harness GRPO，但训练的是 task model 而非编辑器，且保留 symbolic editing（AEGIS）。Harness-R1 将 RL 应用于编辑器本身，是正交但目标重叠的方向 [§2]。
- **extends** Self-Refine / ReAct / Reflection [high]：这些是固定 harness 策略，Harness-R1 明确以它们为 baseline 并展示其不稳定性（Self-Refine 在三个 benchmark 上均降分），将"固定策略"泛化为"学习到的、failure-conditioned 策略" [§1, §4.2]。
- **orthogonal** DSPy / MIPRO / OPRO / TextGrad [med]：这些方法优化 prompt 或 program 结构，但不 post-train proposer 权重，且通常隔离单一 artifact。Harness-R1 编辑的是多阶段 executable runtime 而非单个 prompt [§2]。

## Takeaway

- **方法论可借鉴**：将"编辑 agent harness"形式化为独立 RL 问题（编辑器可训练、target 冻结、reward 来自重跑）是一个清晰的范式分离，可推广到其他"系统级优化是否应学习而非搜索"的场景。
- **需 distrust 的点**：same-batch reward 作为唯一训练信号的泛化保证尚不充分——如果 failure packet 的信息密度不足或 hook 接口表达不够，patch 可能退化为此 benchmark-specific 的规则硬编码。跨 benchmark 的 patch 语义可迁移性未被直接测试（每个 benchmark 训练各自的 patch）。
- **成本盲区**：在线 RL 的 rollout 开销（每步 8 候选 × 全批重跑）可能远超 SFT，论文未报告总计算量，难以判断在小团队或单 benchmark 场景下的可行性。
- **如果只记一件事**：一个 9B 的 outcome-trained harness editor 超过 397B 的 prompted editor——因为"编辑是否有效"只能由重跑结果判断，不能由文本合理性判断。
