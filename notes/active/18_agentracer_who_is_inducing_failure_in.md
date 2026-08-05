---
zone: active
tags: []
pin: false
score: 0.9231334149326804
dwell: 1
---
# 论文阅读笔记：《AgenTracer: Who Is Inducing Failure in the LLM Agentic Systems?》

> **Created:** 2026-06-03
> **状态：** ✅ 已深读
> **arXiv:** [2509.03312](https://arxiv.org/abs/2509.03312)（v2, 2025-09-04, cs.CL）
> **作者:** Guibin Zhang, Junhao Wang, Junjie Chen, Wangchunshu Zhou, Kun Wang, Shuicheng Yan（NUS / CUHK / OPPO / NTU）
> **项目页:** https://bingreeky.github.io/atracer/（声明开放 TracerTraj 数据集与 AgenTracer-8B 模型；正文未给出 license/repo 直链，release 状态未在文本内确认）
> **分类轴：** layer = cross_evaluation 为主（multi-agent trace 的 failure attribution / root-cause 定位），并跨 L2（TracerTraj 数据构造管道——counterfactual replay + fault injection 产出 2,000+ 标注轨迹，属"training resources"轴）与 L3（把 attribution 转成 corrective feedback 驱动 MAS 自演化，+4.8∼14.2%）；signal_kind = cognitive（AgenTracer-8B 是一个**读 trace 做推理**的小 LLM，输出 `⟨think⟩…⟨/think⟩⟨answer⟩agentID|stepID⟨/answer⟩`，本质是 LLM-judge 的训练化小型化，而非非语义规则/内部信号）；cost_profile 双层——**推理期 small_surrogate**（Qwen3-8B 单 pass）但**数据构造期 llm_judge**（analyzer 与 perturbation operator 均为 DeepSeek-R1，且 counterfactual replay 要逐步重跑环境 Ω）；lifecycle = deployment_time（已失败 trace 上离线定位）+ training_time（GRPO 训练 tracer）+ online（feedback 注入下一轮 rollout）；deployability = method_only（方法细节、prompt、reward 公式完整，但数据/模型 release 未在文本内坐实，且数据构造依赖 oracle 与可重跑环境）。
> **角色定位：** 这是 thesis 在 **failure attribution** 任务上一个**结构暧昧**的样本——表面上它是 thesis "lightweight beats LLM-judge" 的强证据（一个 8B 训练小模型在 Who&When 上 agent-level 击败 Gemini-2.5-Pro/Claude-4-Sonnet 高达 18.18%），但它的"轻量"只在**推理端**成立：训练数据靠 DeepSeek-R1 analyzer + 可重跑环境的 counterfactual replay 在沙盒里造出来，方法是"用一个 trained LLM reasoner 取代一个 prompted LLM reasoner"，而非 thesis 真正偏好的 mechanistic detector。它与 MASPrism [17] 同任务、同基准（Who&When）正面对撞——AgenTracer 正是 MASPrism 点名批评的三条既有路线之一（training-based：合成失败日志训练一个 tracer）；与 AgentDebug [13] 同属"attribution → 自演化 feedback 闭环"，且**同样只报端到端增益、不审计 feedback 自身可靠性**——命中 thesis 反对的 auto-feedback 反模式。同时它在方法论上比 AgentDebug 更硬：counterfactual replay 是**真重跑环境**（oracle 修正 + 重新评估 Ω），不是 AgentDebug 那种 prompt 内"想象式 counterfactual"。

---

## Claims

1. 当前 SOTA 推理 LLM 在 agentic failure attribution 上"strikingly inadequate"，准确率普遍低于 10%；这是定义该任务价值的核心动机 [18: §1, Abstract]。
2. AgenTracer 是首个**全自动**标注失败多智能体轨迹的框架，通过 counterfactual replay 与 programmatic fault injection 在六个数据集上产出 2,000+ 条高保真失败轨迹（TracerTraj-2.5K）[18: §1 贡献❶, §4.1]。
3. AgenTracer-8B（Qwen3-8B + 多粒度 RL）在 Who&When 上 agent-level 超过 GPT-4.1 / Claude-4-Sonnet（handcrafted w/ G 各 +26.0% / +12.2%），超过 DeepSeek-R1 ∼12.21%、Gemini-2.5-Pro ∼18.18% [18: §1 贡献❸, §5.2 Obs❷, Table 1]。
4. AgenTracer-8B 给 off-the-shelf MAS（MaAS / OWL-Workforce / MetaGPT）提供 corrective feedback，跨 GAIA / HumanEval+ / MATH-500 带来 4.8∼14.2% 性能提升；其中 OWL（2025-06 GAIA 开源 SOTA）+4.8%，MaAS+MATH-500 +14.21% [18: §5.3 Obs❸, Fig.3]。
5. **decisive error** 被形式化为"最早的、其 oracle 修正足以把系统从失败扭转为成功的动作"：$C(\tau)=\{(i,t)\mid i=\mu(t)\wedge\Omega(\tau)=1\wedge\Omega(R(\tau,t,a'_t))=0\}$，取 $\arg\min_t$ 为根因 [18: §3, Eq.4-5]。
6. 经典 self-refinement（Self-Refine / CRITIC，即便由 GPT-4.1 驱动）对复杂 agentic 轨迹**无效甚至有害**：CRITIC+MaAS+GAIA 在 iteration-2/3 分别 −4.9% / −5.5%，而 AgenTracer 在所有设定单调改善 [18: §5.3 Obs❸, Fig.3]。
7. 提供 ground-truth G **并不一致地**改善 attribution——TracerTraj-math 上 Claude-4-Sonnet 46.03%(w/ G) vs 50.79%(w/o G)，Who&When-HC 上 Qwen3-Coder 51.72% vs 60.35%；论文称 G 有时会误导归因，与 MAST 先前发现一致 [18: §5.2 Obs❶]。
8. step-level attribution 普遍极难：即便最强基线，在 Who&When-handcrafted 上 step-level 也低于 20%（AgenTracer 20.68，Claude-4-Sonnet 17.24，DeepSeek-R1 13.29）[18: Table 1]。
9. fault injection 把 known-good 轨迹的某步用扰动算子 $\Pi$ 破坏，若诱发失败则该步**按构造即为 decisive error**，从而产出 positive-sample 集 $D^+$，与 counterfactual replay 的 $D^-$ 合并成 $D_{tracer}$ [18: §4.1, Eq.8-9, Algorithm 1]。
10. 多粒度 reward 是 format 门控下的 step/agent 加权组合：$R(\hat p_k)=I_{format}\cdot[\lambda r_{step}(\hat t_k)+(1-\lambda)r_{agent}(\hat i_k)]$，其中 $r_{step}$ 用 Gaussian kernel $\exp(-(\hat t_k-t^*)^2/2\sigma^2)$ 给"时间邻近"部分信用、$r_{agent}$ 为二元命中（$\lambda=0.5,\sigma=1$）[18: §4.2, Eq.11-12]。
11. w/o G（更现实、更难）设定下 AgenTracer-8B 仍稳健：TracerTraj-math 上 DeepSeek-R1 去掉 G 掉 9.21%，AgenTracer-8B 维持 57.63% [18: §5.2 Obs❷, Table 2]。
12. 案例研究：一条文档分析长链中，根因在 Step 2（Web Surfer 取了过期文件），只有在 Step 11 看到 `last edit: 2021-08-19` 才暴露；Qwen3-8B 误判 Step 6（Coder 表层报错）、Claude-4-Sonnet 判 Step 7（Manager），唯 AgenTracer-8B 回溯到 Step 2 [18: §5.4, Fig.4]。

## Assumptions

- **counterfactual replay 需要可重跑的环境与 ground-truth 解 G。** rectification operator $R(\tau,t,a'_t)$ 要"替换动作后重新模拟所有后续步并重新评估 $\Omega$"[18: §3, Eq.4]，且 analyzer 被显式喂入 G [18: §4.1, Eq.7]。这把数据构造绑定到**有 oracle 答案且环境可重放**的 benchmark 任务上——生产原始 trace 既无 G 也通常不可逐步重跑。
- **decisive error 是单步、唯一、最早可定位的。** Eq.5 取 $\arg\min_t$ 假设存在一个"修了它就能翻盘"的最早步；多根因/多 span 失败被坍缩为单点（与 MASPrism [17] 的 multi-symptom consensus、TRAIL 的多 error-span 标注假设相反）。
- **fault injection 造出的合成失败分布上等价于自然失败。** $D^+$ 由 DeepSeek-R1 对成功轨迹做"specific, implementable attack"产生（见 Appendix B attack prompt），假设这种程序化扰动诱发的失败与真实 MAS 失败同分布——论文未验证该分布匹配。
- **analyzer / perturbation operator（均 DeepSeek-R1）足够可靠以充当标注 oracle。** 整个 TracerTraj 的标签质量上界由 DeepSeek-R1 的诊断/攻击能力决定；论文未对自动标注做人工一致性校验。
- **trace 已被判定失败且已结构化为 turn-based 单活跃 agent 序列。** §3 形式化要求 $\mu(t)\in I$ 单 agent 每步、$\tau=(s_0,a_0,\dots,s_T)$；混合 success/fail 流的前置 triage（哪些 trace 该送进来）不在范围内。
- **agent/step 索引在标注与评测间是稳定锚点。** agent-level / step-level 双指标依赖 step 边界与 agentID 的一致切分。
- **feedback 注入下一轮 rollout 不泄漏 oracle。** §5.3 把 `⟨think⟩` 推理痕迹回灌给 MAS；analyzer 在数据构造期读过 G，但训练后的 tracer 在 w/o G 推理——论文未扫 feedback 文本是否仍泄漏任务级提示。

## Method

**问题框架。** multi-agent system $M=\langle I,S,A,\Psi,\mu\rangle$，turn-based 单活跃 agent，完整轨迹 $\tau=(s_0,a_0,\dots,s_T)$。给定已失败 $\tau$、环境反馈 $F$，输出失败责任 agent $i^*$ 与 decisive error step $t^*$ [18: §3]。

**Stage A — AgenTracer 自动标注（产 TracerTraj-2.5K）。**
- *Collection*：6 个 MAS（MetaGPT/AutoGen/Smolagents = 手工配置；AgentPrune = 部分自动；AFlow/OWL-Workforce = 全自动）× 6 benchmark（MBPP+/KodCode/Blackjack 编码、GAIA/HotpotQA 通用、MATH/GSM8K 数学）跑出原始轨迹，按 $\Omega$ 分成 $T_{succ}$/$T_{fail}$ [18: §4.1, §5.1]。
- *Locating Decisive Errors（失败轨迹 → $D^-$）*：analyzer agent $\pi_{analyzer}$（DeepSeek-R1）拿到 $(\tau,F,G)$，对每步提出"minimally invasive"修正动作 $a'_t\leftarrow\pi_{analyzer}(s_t,a_t,H_t,F,G)$（要求只给 guidance 不给完整解），重跑得 $R(\tau,t,a'_t)$，**最早**使 $\Omega=1$ 的 $t$ 即 $t^*$、$i^*=\mu(t^*)$（Algorithm 1 Lines 2-8，找到即 break）[18: §4.1, Eq.7]。
- *Utilizing Successful Trajectories（成功轨迹 → $D^+$）*：对 $\tau\in T_{succ}$ 随机采 $K$ 个注入点，perturbation operator $\Pi$（DeepSeek-R1，"intelligent attack expert" prompt，要求 SPECIFIC/IMPLEMENTABLE 代码级改动）生成 $\tilde a_t=\Pi(a_t)$，若 $\Omega(\tilde\tau)=0$ 则 $\langle\mu(t),t\rangle$ 按构造为 decisive error（Algorithm 1 Lines 9-17）[18: §4.1, Eq.8-9]。
- 合并 $D_{tracer}=D^-\cup D^+$ = TracerTraj-2.5K（2,000+ 对；Coding 1288 / Math 630 / Agentic 558，见 Appendix A Table 3）。

**Stage B — AgenTracer-8B 训练（GRPO）。** base = Qwen3-8B；对每条 $\tau$ 采 $G$ 个候选 $\langle\hat i_k,\hat t_k\rangle$，用多粒度 reward（Eq.11-12）算 advantage。相对标准 GRPO：**省去 KL 项**、引入**动态 clip** $B_s=\max(0.2B_0,B_0(1-s/S_{total}))$（早期鼓励探索、后期稳定利用）[18: §4.2, Eq.10]。
- *Format reward* $I_{format}$：严格二元，要求 `⟨think⟩…⟨/think⟩` + `⟨answer⟩agentID|stepID⟨/answer⟩` 可解析。
- *Agent reward* $r_{agent}=I(\hat i_k=i^*)$；*Step reward* $r_{step}$ = Gaussian kernel 给时间邻近部分信用（平滑 reward landscape 稳定训练）。
- 配置：batch 32，rollout 8，lr $1\times10^{-6}$，verl 平台，8×H100。

**Stage C — 推理与自演化。** 推理时 AgenTracer-8B 单 pass 输出 $(i^*,t^*)$ + `⟨think⟩` 解释。自演化用法：MAS 产出失败 $\tau$ → 把 $\tau$（w/o G）喂 AgenTracer-8B，取其 `⟨think⟩` 作 reflective feedback → 注入下一轮 problem solving，迭代 3 轮（对照 Self-Refine/CRITIC，均用 GPT-4.1 实例化）[18: §5.3]。

## Eval

- **基准。** Who&When [Zhang et al. 2025c]：handcrafted 子集源自 Magentic-One，automated 子集源自 AG2；均对 AgenTracer-8B 为 unseen。另从 TracerTraj-2.5K 按 9:1 留出 test，得 TracerTraj-{code/math/agentic}。指标：agent-level accuracy（是否命中 $i^*$）+ step-level accuracy（是否命中 $t^*$）；两设定 w/ G 与 w/o G；采用 MAST 的 "all-at-once"（整条 trace 单 pass 喂模型）[18: §5.1]。
- **基线。** 小模型 Qwen3-8B / Llama-3.2-3B；中模型 Qwen3-32B / Qwen3-Coder-480B；大模型 GPT-4.1 / DeepSeek-R1 / Gemini-2.5-Pro / Claude-4-Sonnet [18: §5.1]。
- **Who&When（Table 1，w/ G / w/o G）。** handcraft agent-level：AgenTracer **69.10/63.82**，次优 Claude/DeepSeek-R1 56.90；handcraft step-level：AgenTracer **20.68/20.68**，次优 Claude 17.24/18.97。automated agent-level：AgenTracer **69.62/63.73**；automated step-level：AgenTracer **42.86/37.30**，与 Claude 40.65/38.83 接近。
- **TracerTraj（Table 2，agent/step，w/ G / w/o G）。** Code：AgenTracer 72.95/72.21 · 18.85/18.85（agent 最高，step 最高）。Math：59.32/66.10 · **57.63/57.63**（step 远超 Claude 38.10/46.03）。Agentic：53.28/50.61 · 36.17/35.55（agent 略低于 Claude 55.20，step 最高）。
- **自演化（Fig.3）。** MaAS/OWL/MetaGPT × GAIA/HumanEval+/MATH-500 三轮迭代：AgenTracer 全设定单调上升（OWL+GAIA +4.8%、MaAS+MATH-500 +14.21%）；CRITIC 在 MaAS+GAIA 反降（−4.9% / −5.5%）。
- **观察。** Obs❶ 现有模型整体不胜任（step-level 多 <10–30%），且 w/ G 不一致改善（有时误导）；Obs❷ AgenTracer 全面超大模型且 w/o G 稳健；Obs❸ 训练化 tracer 的 feedback 优于通用 self-refine。
- **缺失的关键评测。** 无 self-evolution feedback 的**自身可靠性度量**（fix-prediction / regression-prediction accuracy）；无 TracerTraj 自动标注的人工一致性核验；无 reward 超参（λ, σ）消融；无确定性/方差报告；Dataset Details 正文引用为 "placed in ??"（broken cross-ref）。

## Weaknesses

1. **"轻量"只在推理端成立；数据构造重度依赖 oracle + frontier LLM + 可重跑环境，对生产 trace 不可迁移 [18: §3 Eq.4, §4.1 Eq.7]。** counterfactual replay 要逐步替换动作并**重新模拟整条后续轨迹**评估 $\Omega$，且 analyzer 被喂 ground-truth G。生产原始 trace 既无 G、环境通常也无法逐步重放——故 TracerTraj 本质是**沙盒内**用 DeepSeek-R1 造的训练集。thesis 关心的"lightweight signals 能否在真实非 τ-bench 生产分布上工作"在此被绕过：成本被搬到了离线数据工厂，inference 便宜但 supply chain 不便宜。
2. **它是 trained LLM-judge，不是 mechanistic detector——与 thesis "drive determinism deep" 的方向只是表面一致。** AgenTracer-8B 用 GRPO + rollout 采样、输出自由 `⟨think⟩` 推理再抽 `agentID|stepID`；论文**不报任何确定性/跨运行方差**。对照 MASPrism [17] 明确以"零解码、确定性聚合规则、重复运行排名不变"为卖点，AgenTracer 在 thesis "LLM variance 本身有害于生产判断" 的判据上是退一步而非进一步——它把 free-form judge 换成了"更小但仍随机"的 judge。
3. **自演化 feedback 闭环只报端到端增益、零自审计——命中 thesis 反模式 [18: §5.3, Fig.3]。** 4.8∼14.2% 无法区分"feedback 真的对"与"base model 在带噪 feedback 下自己恢复了"。对照 AHE [12: §4.4.2] 显式量化 fix≈5×random / regression≈2×random，AgenTracer 与 AgentDebug [13] 一样**两者都不报**——thesis Anti-patterns 已点名这一类。
4. **fault injection 的合成失败与自然失败分布匹配未验证 [18: §4.1, Appendix B]。** $D^+$ 由 DeepSeek-R1 "attack expert" 对成功轨迹做代码级扰动产生；tracer 训练于此可能 overfit 到"程序化注入痕迹"（如被替换的返回语句模式）而非真实多智能体涌现失败。论文未做 $D^-$-only vs $D^-\cup D^+$ 的迁移消融来隔离 $D^+$ 的贡献与偏置。
5. **step-level 绝对值仍低（Who&When-HC 20.68%）——根因步精确命中不到 1/4 [18: Table 1]。** "+18.18% over Gemini" 是在一个整体 <20% 的弱场上的相对提升；对生产 debugging，step-level <30% 意味着仍需大量人工巡检。agent-level 69% 与 step-level 21% 的巨大落差说明模型"知道谁错了但不知道哪步错"。
6. **决定性错误的单步/最早假设排除了多根因失败，恰是 long trace 的主要失败形态 [18: §3 Eq.5]。** 取 $\arg\min_t$ 把"早期错误被后续错误放大"坍缩为单点。MASPrism [17] 与 TRAIL 都显示真实长 trace 平均 5.68 错/trace、需 multi-symptom consensus；AgenTracer 的形式化与评测都假定单根因，对多 span 场景的适配性未测。
7. **"w/ G 不一致改善甚至有害"被报告却未诊断 [18: §5.2 Obs❶]。** 这是一个反直觉且重要的现象（oracle 监督让 attribution 变差），论文仅引 MAST 附议、未拆其机制（是 G 引入分心 context？还是模型不会用 G？）。对一个核心卖点是"w/o G 也稳健"的方法，这恰恰削弱了"G 本应是上界"的预期，需要解释而非一笔带过。
8. **自演化基线口径不对称 [18: §5.3]。** Self-Refine / CRITIC 用 GPT-4.1 实例化、AgenTracer-8B 是 8B 专训模型；"trained-for-attribution 的窄模型 vs 通用反思 prompt"的对比公平性存疑——更该比的是"同样用 AgenTracer 的 critical-step 定位但 feedback 由 GPT-4.1 写"以隔离"定位价值 vs 反思文本价值"。
9. **可复现性缺口：数据集统计正文引用为 "??"（broken ref）、reward 超参 λ=0.5/σ=1 固定无消融、自动标注无人工核验 [18: §4.2, §5.1, Appendix A]。** TracerTraj 标签质量完全由 DeepSeek-R1 analyzer 决定却无 inter-rater 或 spot-check；σ（Gaussian step reward 宽度）直接决定 step-level 的"部分信用"半径，却未扫敏感性。
10. **counterfactual replay 把"最早使 $\Omega=1$"当 decisive，但 analyzer 的修正动作本身可能引入信息 [18: §4.1 Eq.7]。** analyzer 拿着 G 提出 $a'_t$，"minimally invasive 且不泄漏完整解"是 prompt 约束、非可验证保证；若某步修正实际隐性灌入了 oracle 路径，则该步会被误标为 decisive——标签可能系统性偏向"analyzer 容易修的步"而非"真正的根因步"。
11. **跨 MAS 拓扑的泛化主要靠"覆盖六框架"断言，无按拓扑分层的 attribution 难度分析 [18: §4.1, §5.1]。** Debate-style、software-dev pipeline、fully-automated workflow 的 $H_t$ 结构差异极大（§3 自承 implementation-dependent），但评测未拆"在哪类拓扑上 attribution 更难/更易"，"覆盖全谱"不等于"在全谱上均匀有效"。

## Relations

- **competes-with `17_masprism_lightweight_failure_attribution_for_multi` [high]**：同任务（multi-agent trace 的 failure attribution）、同基准（Who&When，含 handcrafted/automated），且 **MASPrism 显式把 AgenTracer-8B 列为专用基线**（MASPrism Table 1：AG Top-1 AgenTracer 37.30 vs MASPrism 36.51；HC AgenTracer 20.68 vs MASPrism 27.59）。两者是 thesis "lightweight 路线" 内部的**两种相反实现**：AgenTracer = MASPrism 点名批评的 training-based 路线（合成失败日志 + RL 训练一个 tracer，需要 DeepSeek-R1 数据工厂与可重跑环境），MASPrism = 零训练、两次 prefill、确定性聚合。AgenTracer 在长 trace（HC）上更弱、在自演化闭环上更进一步；MASPrism 在确定性与零输出 token 上更彻底。放在一起是"该用 trained small judge 还是 frozen prefill signal 做 attribution"的直接对照。
- **competes-with `13_where_llm_agents_fail_and_how` [high]**：两篇都做 failure attribution + 把结果转成 corrective feedback 驱动 MAS 自我改进。差别有二：(a) **成本结构**——AgenTracer 训练一个 8B tracer（推理便宜），AgentDebug 用 GPT-4.1 全步全模块前线判官（推理昂贵且 All-Correct 在换小模型后掉到 2–6%）；(b) **counterfactual 的真伪**——AgenTracer 的 replay 是**真重跑环境**（oracle 修正 + 重新评估 $\Omega$，Eq.4），AgentDebug 的 "counterfactual" 经其 Algorithm 1 自证是 prompt 内"想象式"（note 13 Weakness #2）。**共同硬伤**：两者的自演化闭环都只报端到端增益、都不审计 feedback 的 fix/regression 预测准确率——同落 thesis auto-feedback 反模式（与 AHE [12] 形成对照）。
- **competes-with `07_agent_as_a_judge` [high]**：AgenTracer 的整个动机是"LLM-as-Judge（含 Gemini-2.5-Pro / Claude-4-Sonnet / DeepSeek-R1）在 attribution 上不胜任（<10%）"，并训练 8B 小模型去击败它们——这是 thesis "把 front-line LLM-judging 换掉" 在 attribution 任务上的又一实证（与 Trajectory Guard [9]、MASPrism [17] 同向）。**但需保留批判**：AgenTracer 的替代物仍是一个 LLM reasoner（cognitive signal、自由 `⟨think⟩` + 采样），只是更小且经 RL 专训；它证明的是"不需要 frontier judge"，而非 thesis 更强主张的"不需要 LLM judge / 应当机制化"。
- **orthogonal `02_agenther_hindsight_relabeling` [med]**：两篇都把**成功轨迹**当数据矿，但方向相反——AgentHER 把失败轨迹 hindsight-relabel 成偏好/SFT 数据（失败→数据），AgenTracer 把成功轨迹 programmatic fault injection 成带标失败（成功→失败）。AgenTracer 的 $D^+$ 构造（已知 decisive error 的合成失败）可视为一种"逆向 hindsight"：它制造而非回收失败。工程上可串联——AgenTracer 的 step×agent 根因标注能给 AgentHER 的 Stage-1 failure detector 提供更细粒度信号；但 thesis "successful-trajectory hindsight is a feature（挖隐性 friction）" 视角下，AgenTracer **没有**挖成功轨迹里的隐性 friction，它只是用成功轨迹当干净底片去注入显式 fault——是不同的 successful-trajectory 用法。
- **orthogonal `01_signals_trajectory_triage` [med]**：Signals 做 L1 triage（哪些 trace 值得复审），AgenTracer 做 within-trace attribution（已失败 trace 里哪步是根因）——同栈不同层，可流水线串联（Signals 筛高 informativeness 失败 → AgenTracer 摊薄定位成本到小子集）。两者都自称"lightweight"，但范式不同：Signals 是非语义规则/小代价信号，AgenTracer 是训练化 LLM reasoner。AgenTracer 默认输入端已判定失败、不做 triage（与 MASPrism 同样把"哪些 trace 该进来"推给上游）。
- **orthogonal `12_agentic_harness_engineering_observability_driven_automatic` [med]**：AHE 与 AgenTracer 都把"失败 → 自动解读 → 输出修复/反馈"做成闭环。**关键差别在自审计**：AHE [12: §4.4.2] 显式量化 evolve agent 的 fix≈5×random / regression≈2×random 作为 falsifiable 边界；AgenTracer 完全没做这种 self-audit，其 4.8∼14.2% 因此无法区分"feedback 对"与"base 自己恢复"。这条关系把 AgenTracer 钉在 thesis Anti-pattern "auto-fix loops that do not audit their own reliability" 上，并指向一个改进方向：自演化论文应采用 AHE 的 manifest/预测-准确率审计，而非只报端到端 delta。
- **orthogonal `03_tsr_trajectory_search_rollouts` [low]**：两者都含 training-time rollout 机制——TSR 选 rollout 进 RL 训练数据，AgenTracer 用 GRPO rollout 训练 tracer（其产物在 deployment 跑）。相关性低于上述 attribution 同行；列此以标记"训练时 rollout 选择 vs 部署时 attribution"的生命周期镜像关系。
