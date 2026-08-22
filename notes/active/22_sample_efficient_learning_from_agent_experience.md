---
zone: active
tags: [update_weights]
pin: false
score: 0.5413333333333333
dwell: 3
---
# Sample-Efficient Learning from Agent Experience

> Topic integration note derived from Library read artifact `.researcher-workspace/library/papers/paper_arxiv_2607_21051/reads/read_paper_arxiv_2607_21051.md`.

## Library read

---
title: "Sample-Efficient Learning from Agent Experience"
authors: ["Chenhui Gou","Haoqin Tu","Yunhao Fang","Jianfei Cai","Hamid Rezatofighi"]
paper_id: "paper_arxiv_2607_21051"
source_kind: "arxiv"
source_id: "arxiv:2607.21051"
source_url: "https://arxiv.org/abs/2607.21051"
pdf_url: "https://arxiv.org/pdf/2607.21051"
read_id: "read_paper_arxiv_2607_21051"
kind: library-read
doc_type: "paper"
tags: []
---

# Sample-Efficient Learning from Agent Experience

> ICL 让 agent 从经验中快速学习但移除上下文后增益消失——本文通过 one-step branch rollout 将经验蒸馏进权重，无需额外环境交互即保留 ≥64.8% 的 ICL 增益。

## Essence

**问题**：In-context learning（ICL）能让 agent 从自身交互历史中快速学习，但一旦从上下文中移除经验，增益即刻消失；直接用收集的经验做 SFT 几乎无法恢复任何增益（GICL 仅 3.8%）。Context distillation 虽可将上下文信息内化进权重，但传统做法需要对教师模型在环境中重新做 rollout，牺牲了环境样本效率。

**做法**：不重新在环境中跑教师 rollout，也不学习世界模型。在已收集轨迹的每个历史截断点 $h_t^{\text{exp}}$ 上，仅让经验条件教师生成**一个**下一步决策 $a'_t$（one-step branch），用该决策作为监督信号训练学生——学生不接收经验上下文。Branch packing 将同一条轨迹上所有 branch point 的教师决策打包进单条训练序列，将训练步数从 768 降至 64。

**证据**：在 749 个 SWE 任务上，Experience Distillation 保留 ICL 增益的 64.8%（pass@1 从 5.3% 升至 51.4%），而直接 SFT 仅保留 3.8%。

**边界**：方法依赖教师模型能从经验上下文中产生比零样本更好的决策；教师生成成本随经验规模线性增长，论文自身承认这是当前限制。

## Claims

1. 直接 SFT 在收集的 agent 经验上几乎无法恢复 ICL 增益：curated SWE 上 GICL = 3.8%，TaleSuite 上 GICL = −2.6% [§4.2, Table 1]。
2. Experience Distillation 通过 one-step branch rollout 将 ICL 增益蒸馏进权重，在 curated SWE 上保留 64.8%、在 TaleSuite 上保留 93.4% [§4.2, Table 1]。
3. Model-free one-step 蒸馏优于 model-based rollout 变体：平均 task-level GICL 83.8% vs. model-based branch rollout 57.3%，因为世界模型误差在长 rollout 中累积 [§4.4, Table 2]。
4. Branch packing 将 4096 个独立 branch 示例压缩为 128 条打包序列，训练步数从 768 降至 64，总时间降低一个数量级以上，且 GICL 从 84.2% 提升至 90.1% [§4.5, Table 3]。
5. Teacher-sampled forward KL 近乎匹配 ICL（Balances 96.7%、Detective 98.4%），而 student-sampled reverse KL（on-policy distillation）几乎无增益（9.1%、0.4%），因为学生在无经验上下文时很少采样到高质量决策 [§4.8, Table 5]。
6. ICL + Experience Distillation 在 curated SWE 上以 9.6× 更少环境样本匹配 PPO 性能（51.4% vs. 17.7%），在 TaleSuite 上以 57.2× 更少样本匹配 GRPO（43.8 vs. 29.9）[§4.3, Figure 1]。
7. 多任务蒸馏能力可迁移至 OOD 任务：494 个 OOD SWE 任务上 pass@1 从 4.62% 升至 8.84% [§4.6, Table 4]。
8. 经验增益可跨 collect-and-distill 周期累积：5 个周期后 TaleSuite 平均归一化分数从 7.1 升至 47.0 [§4.7, Figure 2]。
9. Sampled-token NTP 与 full-distribution KL 性能相当（GICL 84.2% vs. 82.0%），但前者无需存储教师 logits [§4.9, Table 6]。
10. Enhanced Teacher Reasoning（让教师生成更长推理）将 Detective 上 GICL 从 34.9% 提升至 72.5% [§A.1, Table 7]。

## Assumptions

- 教师模型在给定经验上下文时能产生比零样本显著更好的下一步决策——这是整个蒸馏信号质量的源头，论文未在无经验条件下验证教师本身的决策质量上限。
- 任务可被建模为 POMDP 且交互历史可被序列化为 token 序列，适用于文本交互任务（SWE、text-adventure），但对非文本动作空间（如连续控制）的适用性未讨论。
- 经验预处理函数 $g(\tau^{\text{exp}})$ 能有效浓缩长交互历史中的任务相关信息——论文使用了摘要/重写但未给出 $g$ 的具体实现细节或消融。
- Branch packing 的近似（教师决策条件化于先前教师决策而非真实记录）不会显著偏离原始目标——论文承认这是近似，经验上 Table 3 支持但无理论保证。
- 评估时 ICL 参考值（经验留在上下文中）是合理的性能上界——但 ICL 本身受上下文窗口限制，对超长经验可能不是最优。

## Method

**Before vs. After 对比**：

| 方面 | 传统 Context Distillation | Experience Distillation |
|------|--------------------------|------------------------|
| 教师目标轨迹来源 | 在真实环境中重新 rollout | 不需要额外环境交互 |
| 世界模型 | 或用 learned world model 模拟 | 不使用世界模型 |
| 监督粒度 | 完整轨迹级 KL | 每个历史点的单步决策 |
| 环境样本效率 | 需额外环境样本 | 保持原始经验的环境样本量 |

**形式化**：给定任务 $M$、基础模型 $\theta_0$、收集的经验 $\tau^{\text{exp}} \sim p_{\theta_0}^M(\cdot)$，理想目标是 trajectory-level KL：

$$L_{\text{CD}}(\theta; M, \tau^{\text{exp}}) = D_{\text{KL}}\big(p_{\theta_0}^M(\tau' | \tau^{\text{exp}}) \;\|\; p_\theta^M(\tau')\big)$$

该目标可分解为各历史点的 local policy matching，但估计仍需教师在 $M$ 中 rollout。

**One-step branch rollout**：将 branched rollout 的步数 $k$ 取极限 $k=1$，分支仅含教师的一个决策 $a'_t \sim \pi_{\theta_0}(\cdot | h_t^{\text{exp}}, \tau^{\text{exp}})$，完全消除世界模型。目标简化为：

$$L_{\text{EPD}}(\theta; \tau^{\text{exp}}) = -\sum_{t=0}^{T-1} \mathbb{E}_{a'_t \sim \pi_{\theta_0}(\cdot | h_t^{\text{exp}}, \tau^{\text{exp}})} \big[\log \pi_\theta(a'_t | h_t)\big]$$

**Branch packing**：将同一轨迹上连续 branch point 的教师决策自回归打包进单条序列 $c_T = (o_0, a'_0, a_0, o_1, \ldots, a'_{T-1}, a_{T-1}, o_T)$。仅 $a'_t$（教师生成决策）参与 loss，记录的 $(a_t, o_{t+1})$ 作为上下文。这使得 $c_t$ 包含先前教师决策，是原始目标的近似。

**Enhanced Teacher Reasoning**：在教师生成时加入固定推理 prompt $I$，引导教师对预处理经验 $g(\tau^{\text{exp}})$ 做更充分的推理后再产出决策。

**Experience Preprocessing**：对原始经验做重写/摘要/压缩（$g(\tau^{\text{exp}})$），去除冗余探索和无关信息，适配教师上下文窗口。

## Eval

**任务**：749 curated SWE 任务（agent 接收 issue + repo，可检查代码、编辑文件、运行测试、观察 commit 反馈）；6 个 TaleSuite text-adventure 任务（自然语言命令交互，游戏分数为指标）。

**经验收集**：同一任务多次尝试，后续 trial 条件化于先前 trial 的记录。SWE 每 task 8–12 个独立 rollout 进程、最多 10 次 trial，仅保留多次 trial 后成功的轨迹；TaleSuite 每 task 6–12 次 trial，trial 间重置游戏状态。经验平均含 60.5（SWE）/ 612（TaleSuite）交互 turn，总计 61.7M（SWE）/ 502k（TaleSuite）token。

**指标**：domain-specific 性能（SWE: pass@1；TaleSuite: 归一化分数）+ ICL 归一化增益 $G_{\text{ICL}} = \frac{S_m - S_{\text{ZS}}}{S_{\text{ICL}} - S_{\text{ZS}}} \times 100\%$。

**基线**：ICL（经验留在上下文，上界参考）、Zeroshot（无经验）、SFT（直接在收集轨迹上训练）、PPO（SWE）、GRPO（TaleSuite）、on-policy distillation（reverse KL）、model-based rollout 变体。

**运行次数**：SWE 结果为 10 次平均；TaleSuite 为 ≥16 次平均。

## Weaknesses

- **教师生成成本未充分约束**：Figure 3 显示达到 64.8% GICL 需要每经验 trial 生成 16 条 branch-packed 序列（2.79× 经验 token 量的教师生成数据），但论文未报告教师推理的绝对 token 成本或 wall-clock 时间，无法评估实际部署可行性。
- **经验预处理 $g$ 的实现未公开**：$g(\tau^{\text{exp}})$ 是方法链路中的关键环节（原始经验 80k+ token 需压缩），但论文仅说"may rewrite, abstract, or summarize"，未给出具体方法或消融，影响可复现性。
- **base model 未公开且不可比较**：curated SWE 和 TaleSuite 使用不同的 in-house base model，论文未披露模型规模、架构或预训练细节，无法判断结果对模型规模的依赖性。Table 8 的前沿模型对比仅是 zeroshot 参考而非同模型对比。
- **SFT 基线可能未充分调优**：SFT 的 GICL 为 3.8%/−2.6% 极低，论文未报告 SFT 的学习率搜索、数据格式或训练 epoch 等细节，读者无法排除 SFT 实现不当的可能性。
- **OOD 泛化的绝对增益很小**：pass@1 从 4.62% 升至 8.84% 虽然相对提升 91%，但绝对水平仍然很低，论文将此作为"generalization"证据但未讨论实际可用性。
- **Continual setting 仅在 TaleSuite（6 task）上测试**：5 个周期后 47.0% 的结果基于极小任务集，且每个周期需重新收集 4 trial/task 的经验，论文未在 SWE 上验证 continual 设置。
- **Forward KL vs. Reverse KL 的对比仅基于 2 个任务**：Table 5 的核心结论（forward KL 远优于 reverse KL）仅在 Balances 和 Detective 上得出，且使用 single-task 训练，泛化性存疑。

## Relations

- **builds-on** Snell et al. 2022 (Learning by Distilling Context) [high]：Experience Distillation 直接继承 context distillation 框架（教师条件化于上下文、学生不接收上下文），论文 §2 和 §5.1 明确引用并形式化引用其 KL 目标。
- **extends** Janner et al. 2019 (MBPO, branched rollouts) [high]：论文 §3.2 明确将 one-step branch rollout 作为 branched rollout 的极限情形（$k=1$），并引用该工作作为 branched rollout 方法的来源。
- **competes-with** Ye et al. 2026b (On-Policy Context Distillation) [high]：论文 §4.8 直接对比 teacher-sampled forward KL（本文默认）与 student-sampled reverse KL（OPD），结果显示 forward KL 近乎匹配 ICL 而 reverse KL 几乎无增益——两者解决相似问题但蒸馏方向相反，论文 §5.1 明确讨论了区别。
- **competes-with** PPO / GRPO 基线 [high]：论文 §4.3 直接在环境样本效率上对比，ICL + EPD 以 9.6×–57.2× 更少样本匹配 RL 性能——这是论文的核心卖点之一。
- **orthogonal** Penaloza et al. 2026 (Privileged Information Distillation) [med]：论文 §5.1 引用并区分——Penaloza 研究多轮 agent 环境中的 privileged information 蒸馏，但 Experience Distillation 的上下文是真实 agent-environment 轨迹且目标是改进未来轨迹分布，问题设定不同。
- **builds-on** Laskin et al. 2023 (Algorithm Distillation) [med]：ICL 从交互历史中学习的思路与 Algorithm Distillation 的跨 episode 历史建模相关，论文 §5.2 引用但 Experience Distillation 关注将 ICL 增益固化到权重而非在上下文中保持。

## Takeaway

- **方法论上值得借鉴**：one-step branch rollout 的核心洞察——"在已收集历史点仅重采样教师下一步决策，不做任何未来 rollout"——是一种将 context distillation 从需要额外环境交互的约束中解放出来的简洁设计，可直接复用于任何有交互历史的 agent 学习场景。
- **需要警惕的点**：结果严重依赖教师模型从经验上下文中产生高质量决策的能力，而论文使用的 base model 未公开；forward KL vs. reverse KL 的对比结论仅在 2 个任务上成立，不宜过早推广。
- **一个可迁移的工程技巧**：branch packing 将多个 branch point 打包进单条序列、仅对教师生成 token 计算 loss，将训练效率提升一个数量级——这个技巧不限于 agent 蒸馏，适用于任何在长序列上做稀疏监督的场景。
- **如果只记一件事**：直接 SFT 在 agent 经验上几乎无效（3.8%），而在同一经验上用经验条件教师做 one-step 决策重采样再蒸馏可保留 64.8% 的 ICL 增益——监督信号的来源（教师带经验的决策 vs. 原始轨迹）比数据量更重要。
