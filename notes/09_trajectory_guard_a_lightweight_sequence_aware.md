# 论文阅读笔记：《Trajectory Guard — A Lightweight, Sequence-Aware Model for Real-Time Anomaly Detection in Agentic AI》

> **Created:** 2026-04-26
> **状态：** ✅ 已深读
> **arXiv:** [2601.00516](https://arxiv.org/abs/2601.00516)（v1, 2026-01-02）
> **作者:** Laksh Advani（Independent Researcher, Seattle）
> **分类轴：** layer = L1_signal_triage；signal_kind = execution；cost_profile = small_surrogate；lifecycle = deployment_time / online；deployability = method_only（无 repo 链接）
> **角色定位：** L1 分诊层中"学习型小代理模型"路线的代表，与 Signals [1] 的"可解释规则"路线形成方法论分叉。

---

## Claims

1. 在 agent 轨迹异常检测任务上，对预训练 sentence embedding 做 mean-pool 后再套用通用无监督方法（VAE、Isolation Forest、One-Class SVM）的 F1 ≤ 0.69，且大量 anomaly 步会被池化稀释 [9: §Methodology Hyp 1, Table 3]。
2. 仅对 (task, trajectory) 做对比学习微调（all-MiniLM-L6-v2 + MultipleNegativesRankingLoss）能把 F1 提升到 ≈0.82，但在不同轨迹格式间存在 negative transfer，且忽略序列结构 [9: §Methodology Hyp 2]。
3. Trajectory Guard（Siamese Recurrent Autoencoder + 混合损失）在 Galileo/AgentAlign 平衡测试集上达到 F1 0.88–0.94（加权平均 0.92），加权 recall 0.91 [9: Table 3]。
4. 在外部真实失败基准上：RAS-Eval（3,802 条 anomaly）recall 0.86；Who&When（184 条多智能体失败日志）recall 0.92——超过 Phi-3-mini（0.76 / 0.88），接近或匹敌重量级 LLM Judge [9: Table 3]。
5. 单样本推理延迟 32.48 ms（T4 GPU）/ 161.16 ms（CPU），比最快 LLM Judge（Deepseek 557 ms）快 17×、比 Phi-3-mini（A100, 892 ms）快 27× [9: Table 4]。
6. 消融显示：仅对比损失 F1=0.82，仅重建损失 F1=0.75，混合损失 F1=0.92——双目标显著优于任一单目标 [9: Table 5]。
7. 模型在长轨迹上性能退化：2–5 步 F1=0.96 vs 11+ 步 F1=0.87，作者归因于 128 维 GRU thought vector 的信息瓶颈 [9: §Limitations, §Error Analysis]。
8. 论文将 anomaly 划分为两类——contextual（"wrong plan for this task"）与 structural（"malformed plan structure"）——并主张混合损失能在单一模型中统一检测两者 [9: §Final Approach]。

## Assumptions

- 部署期 agent 异常的本质可被还原为"上下文失配 + 结构不连贯"两类的并集；超出此二分的失败模式（例如：环境侧 rate limit、外部服务退化）未被建模。
- GPT-5 perturbation 合成出的 anomaly 与真实部署中的失败分布充分接近（论文用 RAS-Eval / Who&When 部分缓解，但未量化分布差异）[9: §Limitations]。
- 128 维 GRU thought vector 对生产轨迹长度（论文未给出长度直方图）足够；作者已承认 11+ 步退化但未改架构。
- 任务描述（task）在生产中可用且语义清晰——Siamese 任务塔以 task 为锚点，task 缺失/含糊时整个对比损失失效。
- 阈值在验证集上选 F1 最优后即冻结，迁移到极不平衡的外部 benchmark 时仍是合理操作点（论文未在外部集重新选阈值）。
- 在外部 benchmark 上仅报告 recall 是合法的：作者论点是"漏报代价高于误报"，但生产分布是"99%+ 正常"时该论点站不住——见 Weaknesses §2。

## Method

**输入定义。** 轨迹 τ = {s_1, ..., s_n} 是有序步骤序列；任务 T 是自然语言描述。Galileo 的步骤是 NL 指令字符串，AgentAlign 的步骤是结构化 JSON tool call。

**架构（双塔 Siamese）。**
- **Task Tower**：以 all-MiniLM-L6-v2 嵌入 task（384 维），经 MLP 投影到 128 维 latent vector v_t。
- **Trajectory Tower**：对每个 s_i 取 sentence embedding，喂入 GRU encoder，产出"thought vector" v_s（128 维）；GRU decoder 重建原始步骤序列。

**混合损失。**
$$L = L_{contrastive} + \alpha \cdot L_{reconstruction}, \quad \alpha = 0.5$$

- L_contrastive 用 Triplet Margin Loss（margin=1.0）+ in-batch negative sampling：task 为 anchor、配对 trajectory 为 positive、batch 内其他 trajectory 为 negatives。意图："这个 plan 是否对得上这个任务"。
- L_reconstruction 用 MSE 学步骤序列的"语法"。意图："这个 plan 自身是否连贯"。

**训练数据与划分。**
- 初始 good 池：6,767 条（Galileo 自适应工具用 + AgentAlign benign 类）；85/15 切分得 5,752 训练 + 1,015 验证。
- 测试集：1,000 good（250 Galileo + 750 AgentAlign）+ 4,822 anomaly = 836 合成（GPT-5 在 holdout good 上注入 1–3 步坏步骤）+ 3,802 RAS-Eval + 184 Who&When。
- 测试集在训练前完全隔离，验证集仅用于阈值选择和早停。

**合成扰动。** 由 openai/gpt-5 按论文 Listing 1 提示词为每条 good 注入 1/2/3 步"逻辑不一致、in-domain、句法看似合理但语义错"的步骤，覆盖两类 anomaly：
- Contextual：在电信任务中插入"搜应用下载"，或在听音乐流程中插 CloseMusicApp。
- Structural：危险参数的 delete_file，或简单查询配上不合理 reasoning trace。

**实现细节。** Adam, lr=2e-5, batch=16, 20 epochs；阈值在验证集上选最大化 F1 的点；推理期 forward 一次得 (v_t, v_s, 重建序列)，组合 contrastive 距离 + 重建误差为 anomaly score 与阈值比较（论文未明确公布 score 聚合的精确公式，仅说混合损失在训练时联合优化）。

## Eval

- **测试床构成：**
  - 合成内场（平衡）：Galileo 子集（250 good vs 248 anomaly），AgentAlign 子集（750 vs 588），加权混合（Mixed Synth）。
  - 外部 hold-out（仅 anomaly）：RAS-Eval 3,802 条安全审计失败、Who&When 184 条多智能体失败日志。
- **基线（轻量）：** VAE (bge-large)、One-Class SVM、Isolation Forest——均跑在 mean-pooled 预训练嵌入上。
- **基线（重量）：** Gemini Flash 1.5、GPT-4o Mini、Deepseek v3.2-exp、Phi-3-mini，作为 LLM Judge（提示词见 Listing 2，零样本，单次判定 good/anomaly）。
- **指标：** 平衡集报 P/R/F1（anomaly 类）；不平衡外部集仅报 R（无 normal 样本，P/F1 不可算——这是论文自证的安全度量选择）。
- **硬件：** Trajectory Guard 在 NVIDIA T4 GPU；Phi-3-mini 在更高端 A100；其他 LLM Judge 走 API。
- **关键结果（Table 3）：**
  - Trajectory Guard：Mixed Synth (0.94/0.91/0.92), RAS-Eval R=0.86, Who/When R=0.92。
  - Gemini Flash 1.5：Mixed (0.93/0.98/0.95), RAS-Eval R=0.91, Who/When R=1.00（最强但最慢）。
  - GPT-4o Mini：Mixed (0.89/0.97/0.92), RAS-Eval R=0.96, Who/When R=0.79。
  - Deepseek v3.2-exp：Mixed (0.77/0.94/0.85), RAS-Eval R=0.90, Who/When R=1.00。
  - Phi-3-mini：Mixed (0.80/0.82/0.81), RAS-Eval R=0.76, Who/When R=0.88。
  - VAE/OC-SVM/Iso Forest：F1 = 0.69 / 0.55 / 0.53。
- **延迟（Table 4）：** Iso Forest <1.0 ms（CPU），Trajectory Guard 32.48 ms（GPU）/ 161.16 ms（CPU），Phi-3-mini 892 ms（A100），Deepseek 557 ms，Gemini 603 ms，GPT-4o Mini 735 ms。
- **消融（Table 5）：** contrastive-only F1=0.82，reconstruction-only F1=0.75，hybrid F1=0.92。
- **错误分析：** 长度敏感——2–5 步 F1=0.96，11+ 步降至 0.87。

## Weaknesses

1. **外部基准不报 false-positive，安全论证存在结构性漏洞 [9: Table 3]**。RAS-Eval / Who&When 仅含 anomaly，论文据此只报 recall。但生产分布是"绝大多数轨迹正常"，一个把所有轨迹都判为 anomaly 的退化分类器同样能在 RAS-Eval 上拿到 1.00 recall。论文的"安全场景下 recall 优先"论证回避了"误报淹没分诊队列"这个 L1 分诊层最关键的代价问题——而这恰是 Signals [1] 着重处理的。
2. **GPT-5 合成 anomaly 与训练–评估之间的循环风险 [9: §Limitations]**。作者自承使用 openai/gpt-5 同时作为合成器与（隐含的）分布定义者。外部基准能部分缓解但 RAS-Eval recall 已从合成集的 0.91+ 跌到 0.86，提示分布迁移损耗已经发生且未量化。
3. **二元黑箱输出，无法对接 L2 重打标 [9: §Conclusions]**。模型只输出 anomaly/good 单一标签，未暴露"contextual 还是 structural"细分——尽管混合损失声称同时学两者，论文没有按异常类型分别报告 recall（最该做的消融），无法回答"哪种异常被哪个 head 主导抓到"。AgentHER [2] 类管道需要"为什么失败"才能改写轨迹，Trajectory Guard 的输出不足以驱动该管道。
4. **128 维瓶颈是已知病症却未修 [9: §Limitations]**。F1 在 11+ 步上从 0.96 跌到 0.87；作者只在 Future Work 提"也许加 attention"。生产 agent 轨迹长度中位数往往就在 10+，论文宣称"production deployment"却恰好在生产分布的主区间退化最严重。
5. **LLM Judge 基线被弱化呈现 [9: §Latency, Listing 2]**。LLM Judge 用零样本单 prompt，比较的是"未优化的 LLM Judge vs 精调的小模型"。一个加 few-shot / CoT / 多次投票的 Judge 实际差距会缩小，"17–27× 快"的相对优势叙事建立在这个基线选择上。论文未对 Judge 做 prompt-engineering 上界。
6. **混合损失权重 α=0.5 静态、无敏感性分析 [9: §Implementation]**。论文称双目标"协同"，但未扫描 α；从消融数字（contrastive 0.82, reconstruction 0.75）看，contrastive 项是主要贡献，重建是辅助——而作者把它们呈现为对等的两条腿。
7. **任务描述鲁棒性未测**。Task Tower 以 task embedding 为对比锚点；生产中 task 常含糊、缺失或被 system prompt 隐藏。task 退化时 contrastive 项会失稳，但论文未在 task-noised 条件下测试。
8. **Who&When 样本太少（184）**，置信区间宽到不能支撑"匹敌重量级 baseline"的论断；论文却把 0.92 recall 作为核心宣传数字之一。
9. **环境侧失败被合并入"anomaly"，无法区分 agent vs environment**——这是 Signals [1] 明确隔离的一类信号（Environment.Exhaustion 不进训练管道）。Trajectory Guard 的二元标签把 rate limit、context overflow 等系统侧失败与 agent 失误混为一谈，下游若用其作为 RLHF/DPO 数据源会引入虚假相关性。
10. **检测器漂移、版本管理、多语言鲁棒性零讨论**——与论文"real-time deployment""production"叙事直接矛盾，是生产盲区，且无 repo / 阈值 / 模型权重发布，"deployable tool"宣称不可验证。

## Relations

- **competes-with `01_signals_trajectory_triage` [high]**：两篇都在 L1 分诊层主张"非 LLM-Judge 的廉价前置过滤"，但方法论尖锐对立——Signals 坚持"信号不是质量分"、用可解释短语 + 序列规则；Trajectory Guard 用监督学习的 Siamese RNN 输出黑箱二元 anomaly score，正是 Signals 反对的设计。两者在同一职责位上提供互斥范式：可解释规则 vs 学习型小代理。Trajectory Guard 的 F1 0.92 与 Signals 的 82% informativeness 不在同一指标上（前者是分类正确性，后者是采样信息量）——直接数字对比是误导，但作为方法论分叉是 KWeaver 必须做的二选一。
- **competes-with `07_agent_as_a_judge` [high]**：论文显式把 GPT-4o Mini / Gemini Flash 1.5 / Deepseek / Phi-3-mini 作为 LLM Judge 基线对比，并以 17–27× 速度优势作为核心卖点。这是 thesis 中"评估范式 vs 采样范式"对峙在新一轮论文中的延续——只是 Trajectory Guard 用一个有监督学到的"小判官"替代了规则系统。
- **orthogonal `06_agentseer_agentic_vulnerabilities` [med]**：两篇都拒绝纯语义检测，但用的是同一轨迹的两种结构视图——AgentSeer 看图（action-component graph 拓扑），Trajectory Guard 看序列（GRU 时序）。两者天然可叠加：图侧抓"路径异常"，序列侧抓"时序异常"。
- **builds-on `04_agenttrace_structured_logging` [low]**：Trajectory Guard 接受结构化工具调用序列作为输入（AgentAlign 即 JSON tool call 序列），隐式假设 AgentTrace 类的 Operational + Contextual 日志已就位。论文未引用 AgentTrace；该关联是按数据格式兼容性反推的工程依赖。
- **orthogonal `02_agenther_hindsight_relabeling` [med]**：管道上 Trajectory Guard 可作 AgentHER 的上游过滤器，但其二元输出不足以驱动 AgentHER 需要的失败原因分类（Constraint_Violation / Tool_Error / Incomplete 等）。要对接需在 Trajectory Guard 输出端额外增加"contextual vs structural"二分类 head——论文声称模型内部已学但未独立暴露。
- **orthogonal `03_tsr_trajectory_search_rollouts` [low]**：TSR 是训练期 rollout 选择，Trajectory Guard 是部署期 anomaly 检测，生命周期不同。但二者共享"用学习型代理模型替代 LLM 评估"的思路，可作为同一方法论家族在两个生命周期端的对应物。
