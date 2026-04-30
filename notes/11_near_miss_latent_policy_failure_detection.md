# 论文阅读笔记：《Near-Miss: Latent Policy Failure Detection in Agentic Workflows》

> **Created:** 2026-04-30
> **状态：** ✅ 已深读
> **arXiv:** [2603.29665](https://arxiv.org/abs/2603.29665)（v1, 2026-03-31）
> **作者:** Ella Rabinovich, David Boaz, Naama Zwerdling, Ateret Anaby-Tavor（IBM Research）
> **分类轴：** layer = cross_evaluation（更确切：post-hoc trajectory-level eval，可作为 L1 triage 信号）；signal_kind = execution + environment（工具调用序列 + guard 引用的世界态读路径）；cost_profile = small_surrogate（在线只跑生成出的 Python 搜索代码，offline 用 LLM 写 guard / 写 search code）；lifecycle = deployment_time（论文定位为 offline evaluation，§6 自承也可在线但 latency 大）；deployability = method_only（code "will be made available"，τ²-verified 基准已开源，但 ToolGuard guard 库私有依赖）。
> **角色定位：** 把"成功轨迹里隐藏的策略失败"做成可计算指标——与 thesis "successful trajectory hindsight is a feature, not a curiosity" 直接呼应。Near-miss 是"任务完成但 agent 漏检前置条件"的精确化定义；其检测算法是**用 ToolGuard 的 guard code 当 oracle、然后在 trajectory 里反向找 read-only 读路径**——这是 thesis 中"成功轨迹的 2/3 仍含可学习 friction"的一个**具体可实现实例**。对 KWeaver L1 triage 的价值：在 outcome=success 的子集上提供一个非 LLM-judge 的 friction 信号；对 L2 relabeling 的价值：被检出的 latent failure 即天然的"应当读但未读"对偶演示，可直接形成 SFT 对（"先调 RO 再调 MTC" vs "直接 MTC"）。

---

## Claims

1. 当代基于 LLM 的 agentic 工作流在策略合规评估上存在系统性盲点：仅以最终系统状态对比 ground truth 的 reference-based 评估会漏掉一类失败——agent 跳过了策略要求的前置检查、却因为客户提供的事实恰好正确而**侥幸**到达正确终态 [11: §1, §3.2]。论文将其定义为 *near-miss* / *latent failure*。
2. 在 τ²-verified Airlines 上，将这一指标用于 6 个当代 LLM agent，**含 mutating tool call 的轨迹中** latent failure 率为 8.6%–17.3%（GPT-oss-120b 最低 8.6%，GPT5-chat 最高 17.3%），即使这些轨迹的最终数据库态都与 gold state 一致 [11: §4.3 Table 2]。
3. 在 τ²-verified Airlines 全部失败案例中，约 25% 的失败属于显式策略违规（由 ToolGuard 的 guard code 直接捕获），其余为其他类型失败——这是论文给出的"显式违规 vs 终态不达标"的相对量级 [11: §1 footnote]。
4. Latent failure 的分布在 mutating 工具间**严重不均匀**：`update_reservation_flights()` 是最易被无前置检查直接调用的工具，其次为 `cancel_reservation()` 与 `book_reservation()`；`update_reservation_passengers()` 上未观测到 near-miss [11: §4.4 Figure 4 left]。
5. 被遗漏的 read-only 工具同样集中：`get_flight_status()` 是最常被绕过的（agent 经常在不验证 "available" 状态的情况下直接更新预订航班）；其次为 `get_user_details()`（与"更新航班需要重新验证支付方式"这条策略关联）[11: §4.4 Figure 4 right]。
6. 用同一 SOTA LLM（Claude-Sonnet4）同时承担 (a) 生成 history-search code 和 (b) 直接 LLM 搜索 history 这两种实现策略，结果是**生成代码的方法在两个 agent 上都达到 P=1.00 / R=1.00**，而直接 LLM 搜索 P 也是 1.00 但 R 仅 0.71–1.00；GPT5.1-Codex 生成的代码在两个 agent 上 P 仅 0.41 / 0.35（大量 false positive，无法定位实际存在的 RO 调用）[11: §4.2 Table 1]。
7. 论文论证"评估应当评判决策过程而不仅是最终结果"，并提出 latent failure rate 作为一个可与传统 reference-based 指标并列的轨迹级指标——其前提是"non-adversarial users provide accurate details"（敌对设定下 latent failure 会变成实际损害）[11: §1, §5]。
8. 显式策略违规率（policy violation ratio，由 ToolGuard 直接捕获）与总失败率呈正相关，但与 latent failure 率**没有清晰关系**——即在终态准确度相近的两个模型间，latent failure 率可以差近 2× [11: §4.3 Table 2]。
9. Qwen2.5-72b 在 200 个 simulation 中产生了异常高的 144 个含 MTC 的轨迹（其他模型为 85–116），人工检查发现这是 agent "过度执行"——在用户请求并不严格要求 mutating 动作时仍然触发；该模型同时具有最高总失败率 0.611，说明"积极执行"与"高失败率"在此场景下捆绑出现 [11: §4.3]。
10. ToolGuard 的"对 mutating call 自动生成 Python guard 函数"管线在原论文（Zwerdling et al. 2025）中的设计目标是 runtime enforcement；本文将其**重新用作 offline evaluation oracle**——guard code 在执行时遇到的任何 read-only 调用都被解读为"该次 MTC 的策略前提"，这构成了 near-miss 检测算法的核心 [11: §3.1, §3.3]。

## Assumptions

- **客户提供的事实是真实且准确的（non-adversarial threat model）。** 论文反复强调，在 non-adversarial 设定下 latent failure 不引发实际后果；这个假设把 near-miss 的"危害论"从安全风险降为"评估盲点 + 潜在攻击面" [11: §1, §5]。
- **必要的数据访问工具存在且对 agent 可用。** 论文为此还**给 τ²-verified Airlines 补了两个原本缺失的 read-only 工具**（`get_flight_status()` 与 `get_flight_instance()`），承认在原始基准上某些策略前置事实根本无法访问 [11: §4 footnote 8]。
- **ToolGuard 自动生成的 guard code 正确地编码了策略。** 任何 guard 漏写或写错都会直接传播为 near-miss 的 false negative / false positive；论文 §6 在 Limitations 里承认这是"质量上界" [11: §6]。
- **同一信息可由多个 read-only 工具等价获得**——例如 "within-24-hours" 既可由 `get_reservation_details()` 也可由（论文虚构的）`get_reservation_timestamp()` 验证；判定的核心是"信息需求是否被满足"，不是"具体 tool name 是否匹配"。这要求 search 阶段能识别**等价 RO 集合** RO=(ro₁, …, roₖ) [11: §3.2]。
- **轨迹的 tool-call 历史是结构化、可访问、且包含完整的 args 与 return value 的——** Listing in §3.3 的搜索逻辑直接依赖 `self._messages` 中每次工具调用的完整记录及其返回。
- **GPT4.1 作为 user simulator 的行为代表真实用户分布。** 论文沿用 τ²-verified 默认设置，未对 simulator 选择做敏感性分析——但 latent failure 的判定只依赖 agent 工具调用序列，不直接依赖 user simulator 文本，所以这项依赖较弱。
- **作者其中一人完成所有 ~400 simulation 的 near-miss 标注**（§7 Ethical Considerations）；不存在第二人交叉标注、未报告 inter-annotator agreement。

## Method

**问题框架。** 给定一条已完成且**最终态符合 ground truth** 的对话轨迹，对其中每个 mutating tool call MTC(args) 执行下列流程，判断该次 MTC 是否构成 latent failure：

1. **加载 guard code。** 调用 ToolGuard offline 阶段为该工具生成的 `guard_<toolname>(args)` 函数（论文给出的样例为 `guard_cancel_reservation()` 包含 `within_24_hours(res_id)` 子谓词）。
2. **执行 guard。** 在执行轨道上遇到一次 read-only 工具调用 `ro` 时（例如 `get_reservation_details(res_id)`，用于满足 within-24-hours 的事实需求），暂停并提取该 RO 的"信息需求描述"。
3. **在 trajectory 历史里反向搜索等价 RO。** 给定信息需求，在 MTC 之前的所有历史 tool call 中查找：(a) 完全相同的 ro(args) 调用，或 (b) 任意一个**等价**的、能满足同一信息需求的 RO 工具调用——例如要验证航班 "available"，可以是 `get_flight_status()`，也可以是 `search_direct_flights()`（其返回包含状态字段）。
4. **判定。** 若历史中至少存在一个等价 RO，则 NM(MTC(args)) = FALSE（agent "adequately informed"）；否则 NM = TRUE，记一次 latent failure。

**两种 history-search 实现策略（§3.3）：**

- **(2.1) LLM-based search**：直接把"信息需求 + tool 列表 + 历史"塞给一个 LLM，问"该需求是否已被历史中的 RO 满足"。Prompt 全文见 Appendix A.1，要求模型**仅基于历史 tool call 结果**回答（用户消息明确视为不可信）、可跨 schema 提取部分字段。
- **(2.2) Code-generated search**：LLM 一次性为每个 MTC 生成一段 Python 搜索代码（基于 `search_tool_calls(messages, tool_name, partial_args, return_type)` 的工具函数），离线生成、可审；运行时**只跑代码、不再调 LLM**。Prompt 全文见 Appendix A.2，约束模型只用一个 RO 调用（不组合多个）满足信息需求。

**实现选择。** 论文比较两种策略 × 两个底层模型（GPT5.1-Codex / Claude-Sonnet4），并以人工标签为 ground truth；选定 **code-generated search + Claude-Sonnet4** 作为后续主实验设置（§4.2 Table 1，gray row P=1.00 R=1.00）。

**评估协议（§4.1）。**
- 数据集：τ²-verified Airlines（50 tasks × 4 trials = 200 simulations per agent）。
- 6 个 agent 模型：闭源 GPT5-chat、Claude-Sonnet4、Gemini-3-pro；开源 GPT-oss-120b、Kimi-K2.5、Qwen2.5-72b。
- User simulator：GPT4.1（τ²-verified 默认）。
- 评估指标三档：(a) total failure ratio（终态不匹配 gold）；(b) policy violation ratio（ToolGuard 直接捕获显式违规）；(c) **near-miss rate**（终态匹配 gold 但绕过策略检查；分母可选 200 全集或仅含 MTC 的子集）。
- 论文重点报告"分母 = 含 MTC 的子集"的 NMR，因 latent failure 在概念上仅对 mutating 动作有意义。

**总结。** 这篇方法是**用一个外部 oracle（ToolGuard guard code）的执行轨迹来反推"agent 应当读哪些 RO"**，再用一段 LLM-生成的搜索代码在 agent 轨迹里**反查**这些 RO 是否真的被读了——若没读、且任务居然成功了，就是一次 near-miss。整套设计与 LLM-as-Judge 范式正交：判定结果是确定性的、可审的，且只依赖工具调用历史的结构，不依赖语义理解。

## Eval

- **测试床。** τ²-verified Airlines 50 tasks × 4 trials × 6 agents = **1,200 simulation 总量**；其中 NMR 分母为"含至少一次 MTC 的轨迹"，6 个 agent 分别为 98 / 116 / 85 / 93 / 144 / 97 条 [11: §4.3 Table 2]。
- **History-search 方法验证（§4.2 Table 1）。** ~400 simulations（Claude-Sonnet4 与 Kimi-K2.5 各 200）由作者人工标注 NMR ground truth，两 agent 真实 NMR 均为 7%。结果：
  - Code-gen + Claude-Sonnet4：两 agent 上 NMR 0.07 / 0.07，**P=1.00, R=1.00**（论文据此选定为主实验配置）。
  - Code-gen + GPT5.1-Codex：NMR 0.17 / 0.20，P=0.41 / 0.35，R=1.00（高 recall 但代码经常找不到实际存在的 RO，导致大量 FP）。
  - LLM-search + Claude-Sonnet4：NMR 0.07 / 0.10，P=0.78 / 0.74，R=0.78 / 1.00。
  - LLM-search + GPT5.1-Codex：NMR 0.05 / 0.07，P=1.00 / 1.00，R=0.71 / 1.00。
- **主结果（§4.3 Table 2，括号中为含-MTC 子集的 NMR）：**
  - GPT5-chat：total fail 0.480、policy 违规 0.120、NMR 0.085（**0.173**，最高）。
  - Claude-Sonnet4：0.405 / 0.220 / 0.070（**0.121**）。
  - Gemini-3-pro：0.229 / 0.205 / 0.060（**0.140**）。
  - GPT-oss-120b：0.399 / 0.182 / 0.040（**0.086**，最低）。
  - Qwen2.5-72b：0.611 / 0.207 / 0.061（0.084*，含 144 个 MTC 异常高）。
  - Kimi-K2.5：0.270 / 0.195 / 0.070（**0.144**）。
- **比较基线。** 论文未与 LLM-as-Judge / Agent-as-a-Judge 直接对比；唯一基线是"不做 latent failure 检测"——即标准 reference-based 评估，其本质上将所有 NMR 案例错判为成功。隐含的 baseline = 0% recall on latent failures。
- **跨模型一致性观察。** total failure ratio 与 policy violation ratio 正相关（policy 违规是 total 失败的子集）；total failure ratio 与 NMR **无清晰关系**——失败率最高的 Qwen 反而 NMR 不突出，NMR 最高的 GPT5-chat 反而 total fail 中等。
- **工具级分布（§4.4 Figure 4）。** 易触发 latent failure 的 mutating 工具：`update_reservation_flights()`（绝对最多）> `cancel_reservation()` > `book_reservation()`；`update_reservation_passengers()` 零案例。被绕过的 read-only 工具：`get_flight_status()`（绝对最多）> `get_user_details()` > 其他。闭源模型整体 latent failure 数量略高于开源模型。
- **Limitations 自承（§6）。** 整套依赖 ToolGuard 的 guard 质量；LLM-search 变体每条轨迹/每次工具调用都增加 LLM 调用成本，code-gen 变体减少 runtime 但需要 offline 生成 + 校核；只在单一 benchmark（τ² Airlines）上评估，工具/策略集合有限；未在其他领域验证。

## Weaknesses

1. **"Latent failure"的危害叙事建立在敌对假设上，但全部实验都在非敌对设定。** 论文的论证逻辑是"non-adversarial 下无害但可能在 adversarial 下危险"，却**完全没做对抗实验**——例如让 user simulator 谎报 Gold 会员、或谎报 reservation 时间。这是该论点最该用实证支撑的位置；当前文本只能算 motivation，不算证据。
2. **NMR 的统计基础脆弱。** 含 MTC 的子集仅 85–144 条；GPT-oss-120b 的"最佳 8.6%"实际是 8/93，与 GPT5-chat 的 17/98 之间的差距落在二项置信区间相互重叠的范围内，论文未给出 confidence interval、未做 bootstrapping、未跨 random seed 重复。"GPT-oss-120b is best"这种结论排序在如此样本量下不可靠。
3. **完全依赖 ToolGuard 的 guard code 但缺乏 guard 质量审计。** 论文用 ToolGuard 作为 oracle，但 §6 只一句话承认"bounded by the quality of generated code"——没报告 guard 是否经人工审核、guard 有多少 bug 已知未修、guard 在多大比例的 MTC 上完整覆盖了策略文档。读者无法区分 "agent 漏读 RO" 和 "guard 漏写 RO"。
4. **"等价 RO"判定的语义边界完全交给 LLM 决定，未做外部校验。** Search 时是否承认 `search_direct_flights()` 满足 `get_flight_status()` 的需求，是 Claude-Sonnet4 自己判断的；论文给出几个例子作为 motivation，但没有公开等价 RO 集合的 ground truth、没有报告这一步上 LLM 的标注与人类的一致率。所有"P=1.00 R=1.00"都暗含着"标注者与 search-LLM 共享同一套等价语义直觉"，存在 oracle leakage 风险。
5. **单作者标注的 ~400 simulations 即是 search 方法 P/R 评估的全部 ground truth。** §7 明确说不外包标注，由"作者之一"完成；不存在第二人复核或 IAA 报告。当 NMR 真实值在 7% 这种小比例上时，单标注者的偏置足以解释"完美 P=1.00 R=1.00"——一个边界案例的判断翻转就可能从 P=1.00 落到 P=0.85。
6. **跨模型 NMR 排序受 MTC 触发率污染。** Qwen 的 NMR 表面"正常"是因为分母（144）异常大；如果模型本身被 prompt 鼓励避免 MTC（保守策略），分子分母都会下降——NMR 不能解耦"agent 是否积极行动"与"agent 是否谨慎读 RO"。论文意识到这一点（§4.3 自承"interpret with caution"），但仍把 NMR 作为模型横评指标列出。
7. **Code-gen 路径的成本叙事被低估。** 论文把 code-gen 描绘为"减少 runtime 成本"，但需要为**每个 MTC 类型**离线生成搜索代码（不是一次性写 6 个 search function）；如果 mutating 工具集合扩张或参数模式变化，需要重新生成并审核。论文未量化"生成代码的离线总成本 + 人审核成本"，也未报告生成代码的 bug 率（GPT5.1-Codex 的 P=0.41 暗示 LLM 写出的搜索代码经常错，但只对一个模型抽查了一次）。
8. **NMR = 7% 的人工 ground truth 与 NMR 8–17% 的主实验之间存在隐藏不一致。** §4.2 报告 Claude-Sonnet4 和 Kimi-K2.5 在 ~200 sim 上人工标注 NMR=7%；§4.3 主表里 Claude-Sonnet4 的 NMR(整体)=0.070、NMR(MTC子集)=0.121、Kimi-K2.5 NMR(整体)=0.070、NMR(MTC子集)=0.144。两套数据未明确说明是否同一批 simulation；若是同一批，0.121/0.144 对应的"含 MTC 子集"分母与"全 200"分母比应即 0.07/0.121=58%、0.07/0.144=49%，与 Table 2 中含 MTC 比例 116/200=58%、97/200=48.5% 高度吻合，但论文没把这条一致性核对说清楚。这种含糊使复读者难以判断 §4.2 的 P/R 是否能够外推到主表的其他 4 个未做人工标签的 agent。
9. **方法只能在"finished + outcome=success"的子集上跑——这正好是 thesis 关心的"成功轨迹隐藏 friction"——但论文从未把这一点框定为 deployment-time triage 的卖点。** 它把 latent failure 描述为 evaluation gap，错过了"latent failure 即 successful trajectory 中可学的高价值样本"这条 KWeaver 直接关心的用法（这是解读上的损失，不是方法上的缺陷，但值得在 relations 里补足）。
10. **测试床 τ²-verified Airlines 的策略集合是单域、规则相对简单（24小时窗口、舱位等级、Gold 会员等），且作者还**主动给该域补了两个 RO 工具**才让方法可跑。在更复杂的策略组合（嵌套条件、跨工具事务、随时间变化的策略版本）下，"等价 RO 集合"会爆炸，code-gen 搜索和 LLM-search 的可靠性都未被检验。
11. **Search 阶段对"用户消息中的事实"显式标记为不可信（Appendix A.1）。** 这与 latent failure 的定义其实存在张力：在 Figure 1 例子中，agent 之所以跳过 `get_reservation_details()` 正是因为用户**已经口头声明**了 res_id 与时间。如果检测器假设"用户消息不能作为信息源"，那么任何"agent 信任了用户"的轨迹都会被判 latent failure——但论文又说明非敌对用户的事实通常正确。这两个立场（评估时不信用户 / 危害论里相信用户）共存的边界缺乏讨论。
12. **没有量化 latent failure 与下游可学性的关系。** 论文止步于"这是一个评估盲点"，没尝试把 latent failure 轨迹作为 hindsight relabeling 的来源、生成偏好对、训练或微调一个 agent，验证"修补 latent failure"是否带来策略遵从度提升。这正是 thesis (b) 条可证伪命题（hindsight relabel 后下游 win rate）所期待的实验，但论文不在其作者议程上。

## Relations

- **builds-on `10_policy_invisible_violations_in_llm_based` [high]**：两篇论文同处一条评估光谱：Sentinel 在动作时通过反事实图模拟拦截**显式**违规，Near-Miss 在动作后通过 guard-code 重放检测**侥幸**绕过——后者补上了前者明确不覆盖的子集（"agent 没读 RO 但终态恰好正确"）。Sentinel 论文 §6.2 自承 "Block-only enforcement 不会捕获 outcome=correct 但 process=non-compliant 的案例"；Near-Miss 几乎是为这个空缺量身定做的指标。两者共享"用可执行规则取代 LLM-as-Judge"的设计哲学，且都把工具调用历史作为唯一证据源。在 KWeaver L1 设计上可联合部署：Sentinel 在线拦截、Near-Miss 离线评估剩余成功流。
- **competes-with `07_agent_as_a_judge` [high]**：论文虽未指名，但 §1 与 §5 反复论证"reference-based + 可执行规则比单纯 LLM-judge 更精确地捕获策略合规"——这是对 LLM-as-Judge / Agent-as-a-Judge 范式的**实质性挑战**。AgentAsJudge 把判断委托给 LLM 推理；Near-Miss 把判断委托给一段确定性 Python 代码 + 一次结构化历史搜索。在 thesis "sampling informativeness ≠ judgment accuracy" 的延长线上，本篇的论点是"判断准确性也可以从语义模型转移到结构化代码"。
- **builds-on `01_signals_trajectory_triage` [med]**：两者都属"非语义、规则化、轨迹级"的判定家族。Signals 在 trajectory 内挖通用采样信息量（Interaction / Execution / Environment 三类信号）；Near-Miss 是 Signals "Execution 类信号"的一个**领域特化版本**——具体到"MTC 是否被 RO 前置"。可视为 Signals §2.1 中提到的"tool-call sequence patterns"的一类具体落地实现：把通用 phrase pattern 替换为"guard-code 引出的必读 RO 集合"。
- **orthogonal `02_agenther_hindsight_relabeling` [high]**：Near-Miss 检测的是"成功但漏检"轨迹——这正是 AgentHER 期待的 hindsight relabeling 的**最佳原料**。一个 latent failure 自带显式可纠正模式（"原轨迹"vs"先调 RO 再调 MTC 的纠正轨迹"），可直接生成 DPO 对：负样本 = 原 trajectory（绕过策略），正样本 = 注入必读 RO 后再做 MTC。论文未提此用法，但其方法输出物（latent failure 标注 + 漏读 RO 列表）即为 L2 可消费的结构化 relabel 提示。这是该论文对 KWeaver 主线最直接的贡献。
- **builds-on `04_agenttrace_structured_logging` [high]**：Near-Miss 的 search 阶段强依赖于 trace schema 中**完整保留每次 tool call 的 name + args + return value**——这正是 AgentTrace 的 Operational surface。如果日志只存了 tool name 而省略 args 或 return value，等价 RO 判定就无法跨 schema 匹配字段。论文 Appendix A.1 prompt "search prior tool call results... extract matching fields like origin, destination, flight_number"明确依赖这种结构化记录。在 thesis "L0 schema 是 silent gating constraint"的论证里，Near-Miss 提供了又一个具体例证：schema 缺失会让该方法直接失效。
- **competes-with `08_tide_trace_diagnostics` [med]**：TIDE 用语义化诊断框架在事后分析失败原因；Near-Miss 用纯结构化方法在事后判断是否漏检。两者都属于 deployment-time post-hoc 评估，但走完全相反的路线（语义 vs 结构）。在 thesis "可解释机制 > 端到端指标"的判断方向上，Near-Miss 的方法更靠近作者偏好的一端——它的 NMR 数字背后挂着一段确定性 Python，TIDE 的诊断结论背后挂着 LLM 推理链。
- **contradicts `09_trajectory_guard_a_lightweight_sequence_aware` [med]**：TrajectoryGuard 用 Siamese RNN 黑箱判 anomaly；Near-Miss 隐含立场是"判定应当能给出'缺少哪个 RO'的可解释解释、而不是 anomaly score"。两条路线在 F1 数字上落点接近（TrajectoryGuard ~0.92，Near-Miss code-gen 路径 P=R=1.00 但 ground truth 单标注者），但范式相反——前者把世界知识压进权重，后者要求世界知识住在 guard code 与历史结构里。在 KWeaver 的 L1 设计选择上，两者代表"learned small surrogate vs declarative oracle"的明确分叉。
- **orthogonal `03_tsr_trajectory_search_rollouts` [low]**：TSR 是训练时 rollout 选择，Near-Miss 是部署时事后评估，生命周期不同。但 Near-Miss 检出的 latent failure 子集恰好是 TSR-style "high-information rollout" 在部署侧的镜像——它们的训练侧对偶可以联起来：训练时用 TSR 选 rollout，部署时用 Near-Miss 过滤"看似成功实则漏检"的成功轨迹反哺训练数据池。
- **orthogonal `06_agentseer_agentic_vulnerabilities` [low]**：AgentSeer 在轨迹结构图（action-component graph）上找异常；Near-Miss 在工具调用序列上找"应有但缺席"的 RO。两者关心的图视角不同（轨迹结构图 vs 工具调用历史的隐式 dataflow），但共同点是"通过结构而非语义判定 agent 行为问题"——都是 thesis 偏好的 mechanistic 路线。在 KWeaver 中可叠层：AgentSeer 抓行为异常，Near-Miss 抓策略漏检。
- **orthogonal `05_breaking_observability_tax` [low]**：Near-Miss 的 cost 主要在 LLM 调用（LLM-search 路径）或 offline code generation（code-gen 路径），与 observability tax 论文关心的"日志体积 / 采样率"正交；但其 history search 步骤要求保留**完整 trace**，与 sentinel sampling "只保留 1% 详细日志"的策略潜在冲突——这是 KWeaver 在把 Near-Miss 装进 L0 + L1 流水线时需要解决的工程权衡。
