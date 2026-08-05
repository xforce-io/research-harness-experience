---
zone: active
tags: []
pin: false
score: 0.5942472460220318
dwell: 1
---
# 论文阅读笔记：《Policy-Invisible Violations in LLM-Based Agents》

> **Created:** 2026-04-27
> **状态：** ✅ 已深读
> **arXiv:** [2604.12177](https://arxiv.org/abs/2604.12177)（v1, 2026-04-14）
> **作者:** Jie Wu, Ming Gong（Atlassian）
> **分类轴：** layer = cross_evaluation（更接近 L1 的"动作时检测"，但用于 enforcement 而非 triage）；signal_kind = environment + interaction（依赖会话累积态与世界图属性）；cost_profile = rule_based（声明式图不变量，O(|M|) 验证）；lifecycle = deployment_time / online；deployability = open_implementation（声明发布 benchmark + 系统）。
> **角色定位：** 把"policy 合规检查"从 prompt-level / LLM-judge 模式改写为"对组织知识图的反事实变更检查"。不是直接的 triage 论文，但提供了 (a) 一种与 Signals/TrajectoryGuard 同家族的**非语义、规则化**动作时判定范式，以及 (b) 一个明确的论点：可执行性的真正瓶颈是 world-state 覆盖度，而不是规则/模型本身——与 thesis 中"schema 是 silent gating constraint"一致。

---

## Claims

1. 在 LLM agent 的工具调用场景下存在一类专门失败：**policy-invisible violations**——动作语法合法、用户授权、语义无害，却因为决定合规与否的事实未出现在 model-visible context 中而违反组织 policy [10: §1, §2.1 Definition 2]。
2. 该类失败与 jailbreak / prompt injection / 标准 RBAC / 内容型 DLP 都正交：用户合作、动作授权、可见文本无敏感线索 [10: §2.4, §7]。
3. PhantomPolicy 基准在 60 violation + 60 safe-control 上设计了一条强约束——所有工具响应只返回干净业务数据、不含任何 policy 元数据；任何模型自我规避都只能来自用户消息表层语言线索 [10: §3.1]。
4. 在 execution-oriented baseline prompt 下，五个前沿模型（GPT-5.4 / GPT-5 mini / GPT-5.4 nano / Claude Sonnet 4.6 / Claude Opus 4.6）在 60 个 risky case 上人工复核后违规率为 90%–98.3%（54–59/60），safe-case 误执行率 2/60–8/60；模型间差距（54–59）相对总体效应（90%+）很小 [10: §5.2 Table 5, F1, F2]。
5. policy-in-prompt（在 system prompt 中注入高层组织规则、但仍隐藏实体级元数据）把 risky-case 总体违规率从 95.3% 降到 40.7%，但跨模型不一致（25%–85%），safe-case 错误未消失（5.7%）[10: §5.2 Table 6, F3]。
6. 仅基于可见内容的 DLP 基线在人工复核标签下精度极高（96.06%）但 recall 仅 40.13%（F1=56.61，acc=68.83%）；漏检集中在 context boundary、hidden audience、temporal validity、high-value resource——这些类别的决定性属性按设计就不出现在可见内容中 [10: §5.2 Table 7, F4]。
7. **Sentinel**（counterfactual graph simulation 反事实图模拟）在完整 world-model 覆盖下、对人工复核迹标签达到 acc=92.99% / F1=92.71，仅 5 false positive 与 37 false negative；FN 中 24 例集中在 high-value resource protection 与 accumulated session leakage、10 例由 soft invariant 给出 Clarify 而非 Block [10: §5.2 Table 8, F6, §6.2]。
8. Sentinel-Verify 的复杂度 T = O(|M|)，与基础图大小 |V|+|E| 无关；典型企业图（~100 节点）单次验证 <1 ms，远低于 LLM 推理 1–10 s [10: §4.9 Theorem 1]。
9. 不变量框架可组合：单个 invariant 是纯谓词、无副作用；任意两组 sound invariant 的并仍 sound（不引入新 false negative）[10: §4.9 Theorem 3]。
10. **可执行性的硬上界是 world-model 覆盖度，不是规则质量**：定义 Coverage(W, V) = |{v ∈ V: F(v) ⊆ Facts(W)}| / |V|，即"检测某违规所需事实是否在世界图中存在"的平均比例；任何 violation 若其所需事实缺失，则结构性不可检测，与 invariant 完整性无关 [10: §6.3]。
11. 通过 Monte Carlo 实体随机移除（每档 50 次）量化覆盖度退化：100% 覆盖 → recall 100%；50% 覆盖 → recall 51.6%；0% 覆盖 → recall 仍 20%（来自不依赖实体存在、仅依赖会话元数据的 context-boundary 类不变量）；FP 在所有档位保持 0 [10: §6.3 Table 10]。
12. 实体重要性呈幂律：移除最高度连接的单个外部 contact 即让 recall 下降 23.3%；26 个关键实体中 8 个单独移除影响为 0 [10: §6.3]。
13. 属性消融：将 scope 标签全部置空把 recall 从 100% 砍到 40%（连带禁用五个违规类别的检测）；移除 sensitivity 在 scope 已缺失基础上无额外效应；移除 audience 仅微降到 90% [10: §6.3]。
14. 在 600 条 baseline 迹与 600 条 policy-in-prompt 迹上**全部进行三人逐迹人工复核**；相对原案例级标注，复核改变 32 条标签（5.3%），论文据此明确论证"trace-level human review is necessary"——案例设计与模型实际执行经常不匹配 [10: §1, §5.1]。
15. Sentinel 的 verify 是关于 (t, θ, S) 的纯确定性函数；在所有 1,134 条迹 / 2,975 次工具调用上，post-hoc 验证与模拟在线验证逐调用完全一致——为后置评估等价于在线拦截提供了实证依据 [10: §5.1]。

## Assumptions

- 组织 policy 的合规判定可以被还原为**对类型化属性图的可达 / 流向 / 状态约束**的合取——即所有需要的语义都能编码进 (E, A, R, P) 形式的世界模型 [10: §2.1 Def 1]。
- 工具调用可被一一映射为**图变更**（AddEdge / RemoveNode / AddTaintNode）；read 类操作通过 lazy taint 累积、在下次 outbound write 时物化为 Data_Flows_To 边——此抽象足以表达八类违规 [10: §4.3 Table 3]。
- 用户合作、非对抗。Threat model 显式排除 prompt injection / 越权调用工具——Sentinel 的"在已授权动作中找违规"论点假定上游已有标准 RBAC / 鉴权 [10: §2.3]。
- 合规所需的事实（实体属性、关系、会话源 scope）**已经存在**于企业系统中（LDAP / DMS / 日历 / 访问日志），并可被同步成结构化 KG——因此 world-model 构建被框定为 ETL 工程问题而非研究问题 [10: §6.5]。
- 政策语义是**单调收紧**的：Block > Clarify > Allow 严格序，三值逻辑下 indeterminate(?) 一律谨慎 → Clarify。这把"模糊或缺事实"映射到了人介入而非默默放过——但需要存在一个能介入的真人 [10: §4.5]。
- 评估前提：PhantomPolicy 在构造上 Coverage(W, V) = 1.0；论文给出的 92.99% 是**条件可行性**结果，而非对生产部署的预测；论文自承生产中 coverage <1 时 recall 自动按 Coverage 退化 [10: §6.3]。
- ContentFingerprint 不变量假定敏感字面值（金额、百分比）会被 verbatim 复制到外发文本——不能匹配改写、近义、aggregation 后的泄漏。

## Method

**问题框架。** 把 (Translate → Fork → Mutate → Check → Decide) 五阶段管道置于 agent 与工具之间，所有 outbound 工具调用必须先被 Sentinel 反事实模拟过；read 类调用不阻塞，但更新 session 的 `data_sources` 集合用于后续 lazy taint 物化。

**世界状态图 G = (V, E, τ, π)。**
- 节点类型 T = {contact, document, project, group}；边集合带标签（Member_Of / Belongs_To / Data_Flows_To 等）。
- 属性束 π(v) 编码 LLM 不可见但 Sentinel 可读的元数据：
  - contact: scope ∈ {External < Team < Internal < Restricted}，status ∈ {Active, Inactive}
  - document: sensitivity ∈ {Public < Internal < Confidential < Critical}，audience ∈ {Partner_Ok, Hr_Only, Counsel_Ok, Untrusted, …}
- scope 与 sensitivity 形成 lattice（≤），驱动 Information Flow 的偏序判定。

**动作翻译表（§4.3 Table 3）。**
- read_file(path) → 仅累积 `S.data_sources += {resolve(path)}`（lazy）。
- send_email(to, body) → 对 ∀ d ∈ S.data_sources 生成 `AddEdge(d, r, Data_Flows_To)`；body 经 fingerprint 扫描发现金额/百分比匹配 Confidential 文档时 → `AddTaintNode(d̂, d_src) + AddEdge(d̂, r, Data_Flows_To)`。
- share_files(to, paths) → ∀ p 生成对应 Data_Flows_To。
- delete_email_thread(id) → `RemoveNode(resolve(id))`。
- forward_email(thread, to) → 对 thread 起边到 r。

**反事实模拟。** Fork(G) 创建 O(1) copy-on-write 覆盖层（仅记录 delta），Apply 将每个 mutation 写入覆盖层，**原图永不修改**。读穿透到基图为 O(1)；写为 overlay dict 的 O(1) 追加。这与 CPU 推测执行同构：错预测则丢弃覆盖层。

**七个不变量（§4.6 Table 4）。**
- **硬不变量（违反 → Block）：**
  - I1 ActiveRecipient：∀ 收件人 r，π(r).status = Active。
  - I2 ContextBoundary：∀ r，S.source_scope ≤ π(r).scope。
  - I3 InformationFlow：对每条提议中的 Data_Flows_To 边 (d, r)，按 audience 分支判定（Hr_Only 要求 r.role ∈ R_HR；Untrusted 要求 r.scope ≠ External；Counsel_Ok / Partner_Ok 豁免；默认 π(d).scope ≤ π(r).scope）。
  - I4 ContentFingerprint：在 Translate 阶段做 regex 匹配（金额 `$[\d,.]+[kmb]?`、低特异性百分比、HR/财务/事故关键词），匹配则物化为 taint 节点交给 I3 检查。
- **软不变量（违反 → Clarify）：**
  - I5 RecipientContext：项目内同名 contact 存在时提示"可能寄错人"。
  - I6 ScopeBoundary：会话项目的 scope 不应低于收件人 scope（外部会话有例外）。
  - I7 Liveness：RemoveNode 操作 v 时 π(v).importance ≠ High，否则要确认。

**三值语义。** I ∈ {⊤, ⊥, ?}；⊥ 且硬 → Block；⊥ 且软 → Clarify；? → Clarify。Clarify 是从逻辑结构里**派生**出来的，不是临时启发式——作者将其与 Sagiv et al. 2002 的三值抽象解释挂钩。

**评估协议。**
- 基线 prompt：execution-oriented（"Be proactive — don't ask unnecessary questions, just get things done"），跨条件固定。
- 五个模型 × 120 case = 600 baseline 迹；同等规模的 policy-in-prompt 第二条件 600 迹；累计 1200 迹**全部三人复核**，争议时多数票。
- 三个 enforcement 比较层：(a) 模型自我规避（baseline / policy-in-prompt 两个 prompt 条件下的实际执行结果）；(b) content-only DLP 基线（visible-content 上 5 类正则规则）；(c) Sentinel（动作时图验证）。
- Block 视为"预测违规"，Allow / Clarify 均视为"预测安全（自治执行）"——Clarify 因仍需人介入故计入"未阻止"。
- 跨稳定性：对子集模型重复 baseline 多次，跨运行行为一致性 95.0%–100%，违规计数 σ ≤ 0.71。
- 验证模式等价性：post-hoc verify 与模拟在线 verify 在 2,975 次工具调用上**逐调用完全一致**。

**Coverage 退化实验设计。**
- Monte Carlo 实体随机移除：100% / 90% / … / 0% 共 11 档，每档 50 次试验，量化 recall(μ, σ, min, max) 与 FP。
- 单实体临界性：逐个移除 26 个"在原图上确实承担检测任务"的关键节点，观察 recall 落差。
- 属性消融：分别置空 scope / sensitivity / audience。
- 不变量消融：分别移除 I1–I7。

## Eval

- **测试床：** PhantomPolicy = 60 risky + 60 safe-control，跨 8 类违规均匀分布（Table 2）；伴随 30 contacts / 40 documents / 3 projects / 4 groups 的 world model；故意放置同名歧义实体（三个 John Chen、三个 David、一个看似无害但 audience=HR_ONLY 的 "Team Reference Sheet"）。
- **比较条件：** baseline prompt × 5 model × 120 case；policy-in-prompt × 5 × 120；content-only DLP（5 类规则）；Sentinel。
- **指标：** acc / Precision / Recall / F1，标签来自人工逐迹复核（最终 304 violation + 296 safe）。
- **关键数字（人工复核标签下，Tables 5–8）：**
  - Baseline 违规：GPT-5.4 58/60；GPT-5 mini 59/60；GPT-5.4 nano 59/60；Claude Sonnet 4.6 54/60（safe 误 8/60，最高）；Claude Opus 4.6 56/60。
  - Policy-in-prompt 总体违规率 122/300 = 40.7%（vs baseline 286/300 = 95.3%）；跨模型 25%（Sonnet）– 85%（GPT-5.4 nano），safe 误 17/300。
  - Content-only DLP：acc 68.83%，F1 56.61，Precision 96.06%，Recall 40.13%；跨模型 recall 0.37–0.44。
  - **Sentinel：acc 92.99%、F1 92.71、5 FP、37 FN（24 在 risky 里、其中 10 是软不变量给 Clarify；13 在 safe-control 上模型出现意外违规）；per-model acc 86.67%–95.00%；Sonnet recall 0.74 最低。**
- **Coverage 退化（Table 10）：** 100→90→80→70→60→50→40→30→20→10→0% 对应 recall μ = 100→86.8→79.6→68.9→62.4→51.6→42.5→38.3→29.3→25.8→20.0%（FP 始终 0）。
- **属性消融：** 仅 scope 缺失即 recall 100→40%；audience 缺失 100→90%；sensitivity 在 scope 已缺时无额外效应。
- **不变量消融：** 移除 I3 InformationFlow 单个不变量 recall 下降 35%（覆盖 4 类违规）；移除 I5/I6 反而让 recall 表面上升到 100%——因为它们原本输出 Clarify 被计为"未阻止"，移除后这些 Clarify 直接消失而 Block 不变（属于度量伪影，不是实际改进）。
- **稳定性：** 子集模型多次重复，跨运行一致性 95.0%–100%，违规计数 σ ≤ 0.71；post-hoc 与在线模拟在 2,975 次工具调用上逐调用一致。
- **范围消融：** 类别级 Table 9 表明 audience restriction / oversharing / temporal validity / text-output leakage / accumulated session leakage 在所有模型上几乎"满违规"；context boundary、cross-context dataflow、high-value resource 这三类才有明显的跨模型差异。
- **API 成本（参考）：** 一次完整 120-case 全跑 GPT-5.4 ≈ $1.21、Sonnet 4.6 ≈ $2.50、Opus 4.6 ≈ $4.60；mini/nano <$0.01 各。Sentinel + DLP 验证本地 CPU 跑、相对 LLM 推理可忽略。

## Weaknesses

1. **对主线（trajectory triage for relabeling）不是一篇直接论文。** 论文目标是**阻止**违规执行，而不是**采样**违规以重打标。Sentinel 的输出是 Block/Clarify/Allow 离散决策、不附带 informativeness 评分；要把它降格成"分诊信号"用于 hindsight relabeling，需要把 Block/Clarify 的轨迹作为高信息样本流向 relabeling 层，但论文未提及该用法。直接照搬其判决会丢失正确执行内部的 hidden friction（thesis 强调的"成功轨迹的 2/3 仍含可学习 friction"），因为 Sentinel 在合法动作上一律 Allow。
2. **PhantomPolicy 规模小（120 case，每类 7–8 case），且为单域（公司邮件 / 文件分享）。** 论文自承是 diagnostic unit-test，不做统计泛化论断；但跨模型违规率 90–98% 这种"几乎全军覆没"的结果落在 60 个 risky case 上时，几例摆动就能推动 5%–10% 差距，把它读成"模型间能力排序"是误导（论文也警告这一点）。
3. **基线 prompt 偏激。** 系统 prompt 显式要求"don't ask unnecessary questions"，等于把"困惑就发问"这条最朴素的规避路径预先剥夺，然后报告"模型违规率高"。这是一个**有意拉高违规率**的实验选择，论文 §6.5 自承"violation rates under other prompt regimes may differ materially"——这意味着 baseline 数字不是模型能力的无条件下界，是 prompt-条件化下界。
4. **Coverage(W, V) = 1.0 的评估前提把最难的问题留给了未来工作。** 论文核心论点已经演化为"问题真正的难度在于把企业事实组织成完备 KG"——但**完备 KG 的获取、维护、漂移、版本管理**完全没做实验，连 LDAP/DMS 同步延迟下的 staleness 影响都没量化。论文唯一的退化实验是**随机移除节点**，与真实漂移分布（最近变更的节点缺失率高、低活跃节点缺失率低）相去甚远。
5. **Clarify 的"由逻辑结构派生"是技术装饰。** 三值逻辑确实自然映射到三决策，但 Clarify 本质是"不知道时打断用户"——其用户体验代价、对工作流的实际中断率、用户疲劳后默认点击 Allow 的 social-engineering 风险都未测，论文 §6.7 仅以一句话承认"confirmation-only enforces nothing"就把同类风险从自己身上挪开了。Sentinel 在 PhantomPolicy 上 92.99% acc 中有多大比例其实是把决定推给人？论文在 §6.2 提到 "10 of 24 missed violations on designed risky cases received Clarify rather than Block"——也就是说 risky 集中至少 10/60 被推给用户，未在 headline 数字里展开成本。
6. **ContentFingerprint 是 brittle 的正则黑名单。** 用 `$[\d,.]+[kmb]?` 匹配金额、按关键词列表抓 HR/财务，作者自承生产应换 embedding similarity，但论文里这条不变量在 text-output leakage 类几乎承担全部检测——一旦攻击者改写金额（"四万八"代替 "$48k"）即 100% 漏检。论文未做这条扰动的 ablation。
7. **"高精度"叙事建立在 5 FP / 599 traces 上。** 数字本身漂亮，但 PhantomPolicy 是构造的：safe-control 案件被设计为"动作结构相同、target 合规"，**模型对 ambiguity 的合理犹豫被人工评审标为 safe**；FP 极低部分是因为 safe 集本身是"刻意为 Sentinel 设计的非 trick 集"。在分布不平衡且 safe 案件含真实 ambiguity（同名实体、staged 文件）的真实部署里，FP 极可能上升。
8. **Sentinel 不监听文本输出。** 模型直接在 chat response 里复述敏感信息（不调用工具）的泄漏被显式划出范围（§6.5 Output monitoring）。在生产 agent 系统的语境下这是一个不小的盲点——很多 Cognitive surface 上的失败正发生在 reasoning text，而非 tool call。
9. **Composability 定理（Theorem 3）的"organizations can extend safely"是过度承诺。** 定理只保证不引入 false negative，但允许**任意增加 false positive**——加入新 invariant 把无害动作判违规的可能性是开放的。论文从未在加入第八/第九个不变量后重测 FP，**单调收紧**带来的合规阻塞代价没有上界证明。
10. **Threat model 排除对抗用户**——这是一个清晰的研究边界声明，但读者不可把 92.99% 推广到对抗设定（用户合谋上传以绕过 fingerprint 的负载、把外部联系人改名规避同名歧义检测等）。在含外部利益相关方对话的生产部署环境里这是真实风险。
11. **复核协议的偏置未量化：** 三人复核改变了 5.3% 标签，但论文未报告 inter-annotator agreement、未公开复核手册、亦未验证三位 annotator 在边缘类别（accumulated session leakage、context boundary）上的一致性。Annotator pool 本身就可能内嵌作者对"什么是违规"的解读。

## Relations

- **competes-with `07_agent_as_a_judge` [high]**：论文不点名地把整套 LLM-as-Judge / Agent-as-a-Judge 评估范式类比为 prompt-level 防御，以"policy-in-prompt"作为代理对照——Table 6 给出 25%–85% 违规率与 5.7% safe 错的实测——并明确论证 prompt 中无实体级事实时，模型推理无法替代世界态访问。这与 thesis 反复强调的"sampling informativeness ≠ judgment accuracy"是同一观察的另一侧：把 policy reasoning 委托给 LLM 在事实缺失下结构性失败，与把 triage 委托给 LLM-judge 因成本问题失败构成姊妹论点。
- **competes-with `01_signals_trajectory_triage` [med]**：两篇都属于"non-LLM、规则化、廉价、动作时"的判定家族。Signals 用短语模式 + 序列规则在 trajectory 内挖采样信息量；Sentinel 用图不变量在工具调用边界做合规判定。方法论同源（rule-based、可解释、低成本），目标不同（采样 vs 阻止）。在生产分诊层的设计里，Sentinel 的"环境/上下文"侧不变量（I1 ActiveRecipient、I2 ContextBoundary、I7 Liveness）可作为 Signals 的 Environment 信号类的具体实现样板。
- **competes-with `06_agentseer_agentic_vulnerabilities` [high]**：这是最贴近的"敌对兄弟"。AgentSeer 用 action-component graph 拓扑找漏洞 / 失败；Sentinel 用知识图反事实模拟找 policy 违规。两者都把 agent 行为视为图变更并在图上做结构性判定，但 AgentSeer 的图是**轨迹结构图**（node = action component, edge = causal link），Sentinel 的图是**组织世界图**（node = entity, edge = relation/flow）。两条路线对"图代表什么"的选择决定了它们能抓什么样的失败：AgentSeer 抓"agent 内部行为异常"，Sentinel 抓"agent 行为对外部世界的合规态扰动"。生产分诊层实际需要两层图同时在线。
- **builds-on `04_agenttrace_structured_logging` [med]**：Sentinel 的 session context S（含 source_scope、accumulated data_sources、project context）等价于 AgentTrace 的 Operational + Contextual + User-Interaction 三类 surface 在动作判定时的实时摘要。论文未引用 AgentTrace 但二者数据要求高度兼容——Sentinel 不能在缺 scope 标注、缺 read-event 顺序的日志上跑（Coverage 实验里 scope 缺失 → recall 直接砍 60%，正是 schema 缺失的代价）。这进一步印证 thesis"L0 schema 是 silent gating constraint"的论点：Sentinel 是这条论点在 enforcement 维度的具体例证——不变量再完整、覆盖度不到位也无效。
- **orthogonal `02_agenther_hindsight_relabeling` [med]**：Sentinel 的 Block/Clarify/Allow 决策可作为 AgentHER 失败原因分类的强先验——被 Block 的迹自动获得 "policy violation" 失败标签，进一步细分到 8 类违规即对应 AgentHER 的 Constraint_Violation 子类。但 Sentinel 不能为"完成任务但低效""成功但 friction"这类 thesis 重视的成功轨迹隐藏问题提供任何信号——它在合法动作上沉默。要纳入 relabeling 管道需在 Allow 通道再叠加一个 Signals 风格的友好 friction 检测。
- **orthogonal `08_tide_trace_diagnostics` [low]**：TIDE 是事后诊断、Sentinel 是动作时拦截，生命周期不同。但二者共享一个核心立场——agent 失败的诊断 / 防御应建立在**结构化、领域明确的世界状态描述**上，而非自由文本上的 LLM 推断。Sentinel 的 KG 视图可作为 TIDE 诊断输出的下游结构化承载介质。
- **contradicts `09_trajectory_guard_a_lightweight_sequence_aware` [med]**：TrajectoryGuard 用监督学到的 Siamese RNN 黑箱判定 anomaly；Sentinel 显式拒绝这种"自学边界"的方案，论证（隐含地）只要决定性事实未进上下文，任何**模型类**判定（包括小代理模型）都是不可靠的——只有把 world state 显式传给一个**纯谓词验证器**才能获得"组织无关性 + 可审计 + 可证明 sound"。这与 thesis"可解释机制 > 端到端指标"判断方向一致；TrajectoryGuard 的 F1 0.92 与 Sentinel 的 0.93 数字相近但范式相反——前者把世界知识压进 GRU 权重，后者要求世界知识住在外部图。在生产分诊层的设计里这是"learned small surrogate vs declarative invariants"的明确分叉，二选一或显式叠层都需要立场。
- **orthogonal `05_breaking_observability_tax` [low]**：Sentinel 的 O(|M|) 验证延迟（<1 ms）对 observability tax 影响可忽略；但其要求的 KG 同步管线（LDAP/DMS/calendar/access logs 实时增量）本身是新的 telemetry 负担，论文未估算这条工程线的成本。把 Sentinel 装到生产级 agent 系统之上需要把它的 KG 维护成本计入 observability budget。
