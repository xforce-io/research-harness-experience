---
zone: active
tags: []
pin: false
score: 0.9137872340425531
dwell: 1
---
# Recursive Experiential–Working Memory Evolution for Long-Horizon Agent Harnesses

> 冻结基座模型，用经验记忆–工作记忆耦合驱动状态锚定的技能调用，再把结构化轨迹交给固定 Meta-Agent 做组件级、验证门控的 Skill Memory 递归演化。

### Claims

- 长程 agent harness 的 RSI 瓶颈不在「有没有经验」，而在缺少可对齐当前执行需求的紧凑任务状态：仅按初始指令或整段历史检索技能，会随交互变长而失准 [24: §1]。
- Experiential–Working Memory Coupling（EM–WM）形成闭环：工作状态决定当前需要什么 → 从经验记忆取技能 → 执行反馈经 checker 验证后再更新状态 [24: §1, §2.2]。
- Recuris 把可演化对象收窄为 Skill Memory \(M_k=(E_k,W_k,\rho_k,C_k)\)（技能库、工作记忆规格、调用策略、checker 集），基座 LLM、工具集、Meta-Agent、定位/打补丁程序与验证门在轮次间保持固定 [24: §2.1.1]。
- 结构化轨迹 \(\Gamma_k\) 把每步的工作状态、被调技能、动作、观测、提议状态与 checker 判定绑在一起；相对 raw trajectory，它把失败归因到可修补组件的宏平均准确率从 13.0%（仅 outcome）/ 37.0%（raw）提到 64.8% [24: §2.1.2, §3.4.1, Table 4]。
- 候选补丁只改诊断牵涉的组件，且须在失败源任务上修复并通过 held-out development（含已解 anchor）的回归准则，否则维持原记忆 [24: §2.3.2–2.3.3]。
- 在四个长程基准、十个模型上，Recuris 在 37 个已完成 model–benchmark 对中的 35 个提高任务成功率；τ²-Retail 上 GPT-5.6 Sol +17.8、Claude Opus 5 +15.6（至 87.9%），SkillFlow 上 Qwen3.6-27B/35B 分别 +16.6/+13.5 [24: §3.2, Table 1]。
- 增益随交互 horizon 扩大而非衰减：τ²-Retail 按任务固有完成长度分位数，Recuris 相对 base 在各分位 +17.0 至 +44.7；最长任务档位可达约 +32.2；分离主要在 required-write recall（+26.7），而非读路径召回 [24: §1, §3.3.1, Fig. 4]。
- 消融显示工作状态而非技能内容承载主增益：τ²-Retail 上 EM-only +2.0（CI 含 0）、WM-only +23.9†、EM+WM +25.4†；同库全量注入、由模型自决调用的对照比 Recuris 低 18.0 点且 token/成功更高 [24: §3.3.2–3.3.4, Table 2–3]。
- 关键 harness 机制随域而异：τ²-Airline 去掉 write review −13.5†，τ²-Retail 去掉 status board −17.3†，构成双分离——故修复目标应从失败轨迹读出，而非设计时固定分配 [24: §3.3.3, Fig. 6]。
- 跨任务演化在 held-out 上稳定：同一 86 题冻结评测上，多轮 Meta-Agent 实现（Claude Code / DeepSeek Harness）相对 \(M_0\) 约 +9.3 至 +17.4 且多数 CI 排除 0；第二轮可再叠加增益，但也出现 plateau 与回撤 [24: §3.4.2–3.4.3, Table 5–6]。
- 在 mid-sized deployment model（doubao-seed-2-0-pro）上演化、再原样装到未见模型上，τ²-Retail 仍抬升 GPT-5.6 Sol / Opus 5；SkillFlow 增益更跟规模走，τ² 增益则取决于目标模型剩余失败头寸 [24: §3.1, §3.4.6, Table 7]。
- Terminal-Bench 2.1 无跨任务共享结构时，跨任务演化 13 轮未接纳任何补丁；同机 within-task adaptation 的 headline（相对单次 +26.4）几乎全部由 4 次重试预算解释，匹配预算下「学习」项仅 +2.3（\(p=0.774\)）[24: §3.2, §3.5, Table 8]。

### Assumptions

- 任务族存在可跨题复用的共享结构（工具、策略、或 SkillFlow 式流程族）；无此结构时跨任务记忆演化默认无货可搬，只能退到单任务适配 [24: §3.2]。
- 冻结的指令调优模型在 temperature 0、共享工具与预算下，性能差可归因于 memory-control 层而非采样或工具差异；τ² 上同一模型还扮演 simulated user [24: §3.1]。
- Meta-Agent（Claude Code / DeepSeek Harness 实现的 LLM agent）在「固定」协议下足以完成组件归因与 scoped patch；证据池而非 Meta-Agent 实现细节决定学到什么 [24: §3.1, §3.4.3]。
- Checker 完成谓词能对着 tool/环境回执（或任务侧 evaluator）判定，而不信任模型自述成功；错误拒绝/接受会进入 \(\Gamma\) 并可归因到 \(C_k\) [24: §2.2.3]。
- 单源演化（仅用 mid-sized deployment model 的失败）学到的是模型共享的失败结构，而非该模型 idiosyncrasy；因而同一 \(M\) 可不经再演化迁移到其他模型 [24: §3.1]。
- evolve / gate / test 三分法在跑之前固定，且 test 轨迹永不进入定位、打补丁或接纳决策 [24: §2.3.3, §3.1, §C]。

### Method

**双环结构。**

1. **任务内执行环（EM–WM）**  
   - 按 \(W_k\) 初始化工作状态 \(w_0\)：每个 goal 含内容、状态（pending/done/blocked）、证据与可选 blocker。  
   - 在定义好的执行事件上由 \(\rho_k\) 取技能：τ² 用 call-time（草稿状态变更工具调用时按工具名检索，先返回 synthetic not-executed，再带技能重拟动作）；Terminal-Bench 记忆用 boundary（如 `first_turn`）。  
   - 观测返回后，\(U_{W_k}\) 提议下一状态，\(C_k\) 对观测做完成谓词检验，固定 kernel \(K\) 只提交被支持的变更。  
   - 输出结构化 \(\Gamma_k\)，而非仅 \((a_t,o_t)\) 原始轨迹。

2. **跨任务演化环（bounded RSI）**  
   - 固定定位：\(D_k=A_{\mathrm{fixed}}(\Gamma_k,M_k)\)，把每个诊断失败归因到 \(\{E,W,\rho,C\}\) 之一（修复决策，非严格因果鉴定）。  
   - 固定打补丁：对牵涉组件各写一处编辑，\(M_{k+}=M_k\oplus_{Z_k}\{\Delta m_z\}\)。  
   - 固定门控 \(G_{\mathrm{fixed}}\)：须修复源失败且满足 held-out dev 回归准则才接纳为 \(M_{k+1}\)。  
   - 外环（基座、工具、Meta-Agent、门、定位/打补丁程序）不变，递归只发生在 memory-control 层。

3. **Test-time adaptation 模式**  
   - 证据池与补丁空间收窄到单任务；隐藏 verifier 只回一 bit；失败后 Meta-Agent 更新经验记忆再试，成功即停；与冻结初始记忆的同预算重试对照，并共享首次 rollout [24: §2.3.5, §3.5]。

### Eval

- **基准**：τ²-Retail（114）、τ²-Airline（50）— 策略约束工具对话，成功须环境 verifier 全奖励；SkillFlow（166/20 族）— 终身技能发现，程序化 verifier；Terminal-Bench 2.1（87）— 终端任务，主用于 within-task adaptation [24: §3.1, §3.5]。
- **配置对照**：benchmark 自带 reference agent；Recuris+\(M_0\)（中性初值）；Recuris+evolved \(M\)；同模型、工具、任务、种子、预算（τ² 含 user simulator）[24: §3.1–3.2]。
- **指标**：avg@4 任务成功率；required-write / read-action recall；失败模式相对缩放；held-out Δ vs \(M_0\)；localization 宏准确率；Terminal-Bench 另报 solved-within-budget 与 untruncated avg@4/pass@4 [24: §3.1–3.5]。
- **模型**：deployment = doubao-seed-2-0-pro（唯一演化源）；评测覆盖 Granite/Qwen/gpt-oss 开源族与 Gemini/GPT/Claude 等 frontier；权重全程不更新 [24: §3.1, Table 1]。
- **主结果摘要**：35/37 对正向；deployment 上 τ²-Retail +23.3†、SkillFlow +16.8†；全库常驻 prompt 对照比 Recuris 多 3111 token 却低约 18 点且每成功贵 46% [24: §3.2, §D]。
- **机制消融**：EM/WM/model-controlled 调用；write review / truth guard / status board / gate termination；fault-injection localization；多 Meta-Agent / 门限；跨任务与跨模型迁移；TTA vs retry [24: §3.3–3.5]。

### Weaknesses

- **前线演化成本被「固定 Meta-Agent」话术淡化**：定位与打补丁由 Claude Code / DeepSeek Harness 级 LLM agent 完成，属于 agent-judge 成本档，而非规则/小代理分诊；论文强调协议固定与可替换实现，但未报每轮定位/补丁的 token、美元或墙钟，难与「轻量信号 → 更新」生产叙事对齐 [24: §3.1, §3.4.3]。
- **Harness 增益缺少与同预算 sampling/TTS 的硬对照（τ²/SkillFlow）**：主表相对 reference agent 与 memory 消融成立，但未把「多开几次尝试 / parallel sampling」锁进与进化相同的反馈预算；Terminal-Bench 上作者自己把 headline 拆成重试，其余基准的表观 Δ 仍可能混有执行稳定性，与评价门要求不完全同构 [24: §3.2, §3.5]。
- **跨任务演化的可搬性被任务结构设计预置**：SkillFlow「流程共享」与 τ² 共享工具/策略是正增益主场；无共享结构时 13 轮零接纳——摘要仍把 Recuris 定位为通用长程 RSI 基础，弱化了「共享结构」前置条件 [24: §3.2, abstract]。
- **演化证据池极小（约 16 训练失败题）却支撑大范围迁移叙事**：held-out +9–17 点令人印象深刻，但也意味着补丁可能编码窄失败族；Airline held-out 因无可改进剩余失败而与 0 不可分，说明「迁移」高度依赖 held-out 是否仍含同类失败，主文跨模型表对此边界着墨不足 [24: §3.4.5–3.4.6]。
- **Checker / task-specific evaluator 的 oracle 依赖未量化**：完成谓词可依赖「结构化 tool receipt 或任务侧 evaluator」；若 evaluator 接近评测信号，则 WM 验证强度部分来自评测泄漏而非可部署观测，论文未报告 evaluator 与官方成功准则的重叠度 [24: §2.2.3]。
- **抽象层「最长任务 +32.2 / 失败模式最多降 80%」来自 Figure 1 营销缩放**：失败模式条以 agent alone=100 相对缩放，正文机制段更扎实的是 write recall 与消融；把相对缩放读成绝对失败率会高估生产收益 [24: Fig. 1, §3.3.1]。

### Relations

- competes-with 19_harness_r1_learning_to_edit_executable [med]：二者都冻结 target 权重、从失败轨迹改 harness；Harness-R1 训 9B engineer 用重跑 Δ 做 GRPO，Recuris 用固定 Meta-Agent 对 \(M=(E,W,\rho,C)\) 做组件级、门控补丁，不更新编辑器权重。
- competes-with 12_agentic_harness_engineering_observability_driven_automatic [med]：同属「结构化失败证据 → 可回滚/可验证的 harness 编辑」；AHE 覆盖七类文件级组件并带 change manifest，Recuris 把可演化面收窄到 Skill Memory 四元组并显式 EM–WM 执行耦合。
- competes-with 15_aevo_harnessing_agentic_evolution [med]：都把积累证据反哺「驱动未来行为的机制」；AEvo 编辑演化 procedure/agent context，Recuris 递归演化记忆控制层并坚持外环固定。
- extends 21_rethinking_the_evaluation_of_harness_evolution [high]：Terminal-Bench TTA 分解显示 matched-budget 学习项仅 +2.3（\(p=0.774\)），与 [21]「表观 harness 收益常可被多样本/重试解释」同向；Recuris 自身提供了可引用的正例与反例边界（有共享结构的跨任务 held-out 增益 vs 孤立任务重试主导）。
- builds-on 04_agenttrace_structured_logging [low]：\(\Gamma_k\) 把状态、技能调用、动作与 checker 判定写成可消费轨迹，功能上接近「L0 schema 使下游归因可计算」，但论文不引用 AgentTrace，关联为综合推断。
- orthogonal-to 16_when_agents_go_astray_course_correcting [med]：SWE-PRM 是轨迹中途 in-flight 纠偏（thesis 默认非更新）；Recuris 主路径是跨任务落盘的 Skill Memory 演化，TTA 模式虽单任务重试但仍改外部记忆而非逐步 PRM 灌评。
- orthogonal-to 01_signals_trajectory_triage [low]：同用 τ-bench 族长程工具对话暴露失败，但 Signals 做部署后轻量分诊采样，Recuris 做 harness 侧记忆演化与执行时状态锚定调用，闭环阶段不同。
