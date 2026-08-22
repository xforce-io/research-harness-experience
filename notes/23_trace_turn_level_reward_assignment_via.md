---
zone: active
tags: []
pin: false
score: 0.41200000000000003
dwell: 1
---
# TRACE: Turn-level Reward Assignment via Credit Estimation for Long-Horizon Agents

> Topic integration note derived from Library read artifact `.researcher-workspace/library/papers/paper_arxiv_2607_13988/reads/read_paper_arxiv_2607_13988.md`.

## Library read

---
title: "TRACE: Turn-level Reward Assignment via Credit Estimation for Long-Horizon Agents"
authors: ["Leitian Tao","Baolin Peng","Wenlin Yao","Tao Ge","Hao Cheng","Mike Hang Wang","Jianfeng Gao","Sharon Li"]
paper_id: "paper_arxiv_2607_13988"
source_kind: "arxiv"
source_id: "arxiv:2607.13988"
source_url: "https://arxiv.org/abs/2607.13988"
pdf_url: "https://arxiv.org/pdf/2607.13988"
read_id: "read_paper_arxiv_2607_13988"
kind: library-read
doc_type: "paper"
tags: []
---

# TRACE: Turn-level Reward Assignment via Credit Estimation for Long-Horizon Agents

> 长轨迹只靠终点对错给同一 advantage → 用冻结参考模型在 tool-call 边界上算 gold-answer log-prob 的 log-ratio 状态值，再取相邻 TD 差分当 turn reward，与 GRPO outcome 混训。

## Essence

**问题**：多轮工具 agent 的轨迹可达数十至上百次 tool call，outcome-only RL 把整条轨迹贴同一个 advantage：失败轨迹里有效的中间检索也会被罚，成功轨迹里的冗余动作也会被奖，信号稀疏且方差高。

**做法**：TRACE 不训 critic、不标 process label、不用强 judge。在 tool-call 边界把 rollout 切成 prefix 状态，用**冻结参考模型**（策略初始化拷贝）对 gold answer 的平均 log-prob 打分，再变成 log-ratio 状态值 \(V(S_k)=\log(d_0/d_k)\)；turn credit 取 \(V(S_{k+1})-V(S_k)\) 的 K-step TD 备份，并与 GRPO 组内 outcome advantage 线性混合更新策略。

**证据**：在闭网 BrowseComp-Plus 上，纯 RL（无 cold-start SFT / mid-training / live-web）把 Qwen3-4B 从 7.2 提到 35.6，显著高于同设定 outcome-only GRPO 的 30.0。

**边界**：训练时必须有 gold answer 与可验证终点；价值探针依赖参考模型 logprob，当前证据主要覆盖短答案、可核对的搜索任务，不是任意长输出 agent。

## Claims

- 将 rollout 视为 tool-call 边界上的状态转移，并用「冻结参考模型下 gold answer 是否更可预测」作为 prefix 进度代理，可在无 critic / process label / 强 judge 的条件下得到密集 turn-level credit [§3, §3.2]。
- log-ratio 状态值 \(V(S_k)=\log(d_0/d_k)\)（\(d_k=-\bar{\ell}_k+\epsilon\)）度量的是相对闭合初始答案似然差距的比例，而不是 raw log-prob 绝对增量；一阶 TD 信用 \(\delta_k=\log(d_k/d_{k+1})\) 在轨迹上可望远镜求和，冗余中间 turn 无法抬高该分量的总信用 [§3.2–3.3, Eq. 6–7]。
- K-step TD 备份把延迟证据（如 search 只返回链接、open 后才抬高答案似然）回传到更早的检索决策；turn credit 与 GRPO outcome advantage 以 \(\hat{A}=\alpha_{\mathrm{out}}A^{\mathrm{out}}+\alpha_{\mathrm{turn}}r^{\mathrm{turn}}\) 联合优化，outcome 仍是最终正确性锚点 [§3.3, Eq. 10–12]。
- 信用形式消融中，raw delta → remaining-gap → log-ratio 在 BrowseComp-Plus（Qwen3-4B）上依次提升：32.4 → 34.6 → 35.5（outcome-only GRPO 为 30.0）[§4.4, Table 2]。
- 关闭密集 TD（\(K=0\)）准确率约 30.0，接近 outcome-only；中等 look-ahead 最好（约 34.7–35.6），过大 \(K\) 降至 28.9，弱于 GRPO [§4.4, Fig. 5(b)]。
- turn 系数需适中：过小浪费 prefix 进度信号，过大让局部 readiness 压过最终正确性（系数扫描中最佳约 35.6，更大系数回落到 31.1）[§4.4, Fig. 5(a)]。
- 参考模型取初始化 checkpoint 已足够（35.6）；换 step-200 参考约 36.1，说明不依赖「更强教师」而是稳定的 value probe [§4.4, Fig. 5(c)]。
- 学习曲线上 TRACE 比 outcome-only 更早上升、斜率更陡，并稳定在更高平台；工具调用轮数也更早、更快拉长 [§4.3, Fig. 3–4]。
- 闭网 BrowseComp-Plus：Qwen3-4B 7.2→35.6，Qwen3-30B-A3B 8.4→42.6；四基准平均分别从 GRPO 的 29.5 / 32.5 提到 34.0 / 38.1 [§4.2, Table 1]。
- 仅在合成闭网语料上训练的行为可迁移到开网：30B-A3B 达到 BrowseComp 12.9、GAIA 52.0、xbench-DeepSearch 45.0 [§4.2, Table 1]。

## Assumptions

- 训练时每条轨迹都有已知 gold answer \(y^\star\)，且终点可用确定性 verifier（归一化 exact match + 轻量 format 分）打 outcome reward [§3.1, §4.1]。
- 冻结参考模型对 \(y^\star\) 的条件 log-prob 是「prefix 是否收集到相关证据」的充分进度代理——对短、可核对答案成立，对长/结构化/开放输出未验证 [§3.2, §6]。
- 合成多文档检索题（需链式检索 ≥2 份不可替代证据文档）足以激发长程 credit 问题；标准 multi-hop 太短，故不作主训练分布 [§4.1, §A.4]。
- 控制实验中浏览器接口（search/open/find）、rollout 协议、终端 reward、训练数据与评测协议固定，性能差主要归因于优化目标与 credit 信号 [§4.1]。
- 消融与主结果多为单次训练 run；小幅数值差应作方向性解读而非方差调整后的显著性结论 [§4.1]。
- 参考模型前向打分可批量完成且不参与优化；tool observation 对策略梯度 mask，仅 assistant token 更新 [§3.1–3.2]。

## Method

**与 outcome-only 的对照**：

| | outcome-only GRPO | TRACE |
|--|-------------------|--------|
| 监督粒度 | 整条轨迹一个 \(A^{\mathrm{out}}\) | 每 tool-turn 一个 TD credit + 轨迹级 \(A^{\mathrm{out}}\) |
| 进度估计 | 无 / 隐式 | 冻结 \(\pi_{\mathrm{ref}}\) 的 gold-answer log-prob |
| 额外模型 | 无 | 固定参考模型前向（非 critic 训练） |
| 标签 | 仅终点 verifier | 仅终点 verifier + gold answer 用于打分 |

**输入**：prompt \(x\)、gold \(y^\star\)、组内 \(G\) 条 rollout（ReAct：私有推理 + 一次 browser 动作或最终答案）。

**核心计算**（Algorithm 1）：
1. 终点 reward \(R_g\) → 组内标准化得 \(A^{\mathrm{out}}_g\)（组方差为 0 则全 0）。
2. 在每个 prefix \(S_k\) 上，用冻结 \(\pi_{\mathrm{ref}}\) 算平均 gold log-prob \(\bar{\ell}_k\)。
3. \(d_k=-\bar{\ell}_k+\epsilon\)，\(V(S_k)=\log(d_0/d_k)\)（故 \(V(S_0)=0\)）。
4. 一步 TD：\(\delta_k=V(S_{k+1})-V(S_k)=\log(d_k/d_{k+1})\)。
5. K-step 折扣平均得局部进度 \(c^{(K)}_{g,k}\)；窗口触达轨迹末端时再加 \(\lambda_{\mathrm{term}}\gamma^{T-k}A^{\mathrm{out}}\) 锚定。
6. 对 tool-interaction token：\(\hat{A}_{g,t}=\alpha_{\mathrm{out}}A^{\mathrm{out}}_g+\alpha_{\mathrm{turn}}r^{\mathrm{turn}}_{g,\mathrm{turn}(t)}\)；用 clipped GRPO 目标更新 \(\pi_\theta\)。

**关键设计选择**：log-ratio 而非 raw \(\Delta\bar{\ell}\)，因同等绝对增益在「已接近答案」时信息量更大，且一步分量望远镜；\(K>1\) 处理 search→open 的延迟证据；turn 值不做组内归一化，仅作轨迹内辅助信号。

**主设定超参（正文）**：\(\epsilon_{\mathrm{train}}=0.1\)，\(K=3\)，\(\gamma_{\mathrm{td}}=0.8\)，\(\lambda_{\mathrm{term}}=2.0\)，\(\alpha_{\mathrm{out}}=1.0\)，\(\alpha_{\mathrm{turn}}=0.2\)；Adam \(10^{-6}\)，batch 128，每 prompt 8 rollout，最多 60 tool turns。

## Eval

- **训练数据**：基于 OpenResearcher 离线语料的合成多文档搜索题；FAISS + Qwen3-Embedding-8B 闭网检索；非 live-web 训练 [§4.1, §A.4]。
- **Harness**：ReAct；工具 `browser.search` / `open` / `find`；答案须在 `<answer>` 中；outcome = 归一化 exact match + format 分量 [§4.1, §A.3]。
- **模型**：Qwen3-4B-Thinking-2507（主消融）、Qwen3-30B-A3B-Thinking-2507；均从 base search policy **直接纯 RL**，无 cold-start SFT / agentic mid-training [§4.1]。
- **控制基线**：Base、GRPO、GSPO、GiGRPO——同 backbone、接口、数据、终端 reward、评测协议 [§4.1, Table 1]。
- **外部参考（非控制）**：ASearcher-QwQ-32B、WebDancer-32B、CutBill-30B-A3B、TongyiDS-30B-A3B [Table 1]。
- **评测**：闭网 BrowseComp-Plus；开网 BrowseComp、GAIA、xbench-DeepSearch（Serper API）；Avg = 四基准未加权平均 [§4.1–4.2]。
- **主结果（Table 1 摘要）**：TRACE 在两尺度四基准上均高于同设定 GRPO/GSPO/GiGRPO；BrowseComp-Plus 上 4B 35.6 vs GRPO 30.0，30B-A3B 42.6 vs GRPO 36.4。
- **消融**：信用形式（Table 2）、\(\alpha_{\mathrm{turn}}\)、\(K\)、参考 checkpoint（Fig. 5）；离线 830 rollouts / 3742 turns 上 log-ratio 与终点 \(\bar{\ell}_T\)、正 outcome 的相关及 pairwise ranking 最优（Table 6）。

## Weaknesses

- **缺少「强密集监督」对照**：控制基线全是轨迹级 RL 变体（GRPO/GSPO/GiGRPO），未在同数据/算力下与 process reward model、LLM-as-judge step score 或 Monte Carlo process value 对比，故「无需 PRM/judge」是成本叙事，不是与最强密集信号的 head-to-head [§4.1, §5]。
- **单 run 消融放大边际结论**：论文明确控制消融多为单次训练，但正文仍以 35.6 vs 30.0、28.9（过大 \(K\)）等作为机制正确性证据，未报告多种子方差；数个百分点差距可能不稳定 [§4.1, §4.4]。
- **训练硬依赖 gold answer**：每个 prefix 都要在 \(\pi_{\mathrm{ref}}\) 下对 \(y^\star\) 求 log-prob；这是 train-time oracle 密集化，不能直接用于无标准答案的在线交互或偏好-only 设定——文中强调「无 process label」但淡化了 gold 依赖的强度 [§3.2]。
- **参考前向开销未量化**：每个 prefix 一次（可 batch）参考模型前向是相对 GRPO 的系统性额外成本；附录给了超参与 remote scoring 路径，但无 wall-clock / token / FLOPs 对比，难以判断「critic-free」是否真更便宜 [§3.2, §A.1]。
- **超参敏感被写成「调参故事」**：**\(K\) 过大准确率掉到 28.9，低于 outcome-only**；turn 系数过大掉到 31.1——说明密集信号可引入噪声，稳健默认区间窄，但摘要/结论几乎不强调失败模式 [§4.4, Fig. 5]。
- **外部 agent 数字参与叙事但不可比**：Table 1 上半是不同数据/管线/harness 的系统，正文仍写 4B「超过多个更大 deep-research agent」、30B「接近 TongyiDS」——容易被读成跨系统 SOTA 比较，尽管作者口头标了 non-controlled [§4.2]。
- **任务族窄于「tool-use」措辞**：训练与主增益集中在合成多文档 identification + 浏览器三工具；开网迁移仍是 QA/搜索。对软件工程、计算机使用、多文件补丁等，作者在 Limitations 承认 log-prob 代理可能失效，但主文「base-model tool-use」表述偏宽 [§4, §6]。

## Relations

- builds-on GRPO / DeepSeek-R1 式 outcome RLVR [high]：TRACE 的 outcome 分量与组相对 advantage、clipped 目标直接沿用 GRPO，仅在 token advantage 上叠加 turn credit [§2.1, §3.3, Guo et al., 2025]。
- builds-on temporal-difference credit / potential-based shaping [high]：核心是 \(\delta = V(s')-V(s)\) 的进度差分；相关工作明确对齐 Sutton TD 与 Ng et al. reward shaping 传统 [§2.2, §5]。
- extends Yuan et al. (2024) 类「无 process label 的隐式过程奖励 / 策略-参考 log-likelihood ratio」思路 [med]：从单轮推理的 log-ratio process reward，迁到 agent tool-boundary 的 prefix gold-answer readiness + TD 分解 [§5]。
- competes-with GSPO、GiGRPO [high]：同 backbone/数据/接口的控制比较中，三者改序列或组级估计但仍偏轨迹级；TRACE 主打轨迹内 turn credit，并在 Table 1 全面更高 [§4.1–4.2]。
- competes-with process supervision / PRM / LLM-as-judge 密集反馈路线 [med]：共享「轨迹以下信用」目标，但 TRACE 主张用冻结参考 log-prob 替代 step 标注、强 judge 与可漂移 PRM [§1, §5]。
- orthogonal-to SPA-RL / StepSearch 等「逐步进度归因 / step-wise PPO」agent RL 工作 [low]：同属多轮搜索 agent 的细粒度 credit 文献簇，机制（MC 进度、step PPO vs 冻结 ref log-ratio TD）不同，论文仅在 related work 并列而未直接实验对打 [§5]。

## Takeaway

- **值得借鉴**：把「gold 在冻结模型下是否更好预测」当作可 telescope 的状态势能，用 TD 差分做 turn reward，再**弱权重**挂到 outcome GRPO 上——在有 verifier + gold 的 agentic RL 里，这是一条不训 critic 的密集化路径。
- **需要复核**：主增益与消融多为单 run；相对 PRM/judge 的优势未直接比较；\(K\) 与 \(\alpha_{\mathrm{turn}}\) 敏感，过大 dense 信号会劣于 outcome-only。
- **硬前提**：训练必须能对 gold answer 做参考模型 logprob；当前强证据限于短答案、可自动核对的长程搜索。
- **一句话记忆**：不是再训一个过程奖励模型，而是用冻结参考模型在 tool 边界上读「离标准答案还有多远」，把相邻距离变化当作 turn credit。
