# 论文阅读笔记：《Agentic Harness Engineering: Observability-Driven Automatic Evolution of Coding-Agent Harnesses》

> **Created:** 2026-05-01
> **状态：** ✅ 已深读
> **arXiv:** [2604.25850](https://arxiv.org/abs/2604.25850)（v2, 2026-04-29）
> **作者:** Jiahang Lin, Shichun Liu, Chengjun Pan, Lizhi Lin, Shihan Dou, Xuanjing Huang, Hang Yan, Zhenhua Han, Tao Gui（复旦 / 北大 / 上海启迹智锋）
> **代码:** https://github.com/china-qijizhifeng/agentic-harness-engineering
> **分类轴：** layer = 主体属 off-axis（**harness self-improvement**，非 thesis 四层栈），但其 trajectory-distillation pipeline 投影到 Triage 层方法论；signal_kind = cognitive（用 Agent Debugger 让 LLM-agent 读 trace 提取根因，再加 git-diff 这一弱 execution 信号）；cost_profile = agent_judge（三个 role agent 共享 GPT-5.4 high，evolution 一次跑 32 小时）；lifecycle = 介于 training_time 与 deployment_time 之间——它优化的是部署时使用的 harness，但优化循环本身需要类训练循环式的 rollout 与回归测试；deployability = open_implementation（开源仓库 + 论文给出 prompt 全文）。
> **角色定位：** 这是一篇**自我演化 coding agent**的工作，不是 trajectory triage / hindsight relabeling 工作。它对生产级 agent 系统的价值不在于"是不是又一个 Triage 方法"——它不是；而在于：(a) Agent Debugger 把"百万 token raw trajectory"压缩成"分层可下钻证据语料"是一个可被 Triage / Data-Reconstruction 层直接借用的方法论原语；(b) "change manifest = 自声明预测 + 下一轮真值核对"是一个 falsifiable contract 模式，可移植到 detector versioning / 信号阈值演化的治理设计；(c) 论文用 fix vs. regression precision/recall 把 evolve agent 自我归因的可靠性显式量化（fix 5× random，regression 仅 2× random）——这是对"agent 能否对自己编辑结果做可靠预言"的少见量化报告，提示在做任何 Triage→Data-Reconstruction 自动化反馈闭环时必须独立核验回归预测。

---

## Claims

1. Coding-agent harness 的自动演化的瓶颈不在 evolve agent 的能力，而在 observability：只要 evolve agent 拿到结构化的 action space 和分层的 trajectory 证据，它就能稳定收敛到更好的 harness 设计 [12: §1, §3]。
2. Harness 应当被分解为 7 类**正交、文件级、可独立编辑**的组件：system prompt / tool description / tool implementation / middleware / skill / sub-agent config / long-term memory；解耦使每类失败模式映射到单一组件类，给 evolve agent 干净的 action space [12: §3.1]。
3. AHE 在 Terminal-Bench 2（89 任务）上把 NexAU0 bash-only seed 的 pass@1 从 69.7% 提升到 77.0%（10 轮迭代，~32 小时），超过人工设计的 Codex（71.9%）以及自演化基线 ACE（68.9%）和 TF-GRPO（72.3%）[12: §4.2 Table 1]。
4. 演化得到的 harness **冻结后可跨基准迁移**：在 SWE-bench-verified (500 任务) 上 aggregate 成功率 75.6%，**比 NexAU0 seed 高 0.4 pp 且 token 减少 12%**，比 ACE 减 32% token，比 TF-GRPO 减 21% token [12: §4.3 Table 2]。
5. **跨模型迁移**：把 GPT-5.4 high 上演化出的 harness 直接用于其他基模，在 5 个 base 上全部正向：deepseek-v4-flash +10.1 pp、qwen3.6-plus +6.3 pp、gemini-3.1-flash-lite-preview +5.1 pp，弱模型获益更大 [12: §4.3 Figure 3]。
6. Component-level ablation 显示**增益的载体不是 system prompt**：把 long-term memory / tools / middleware 单独换入 NexAU0 分别 +5.6 / +3.3 / +2.2 pp，**system prompt 单独换入则 −2.3 pp**；prompt-only 自演化（ACE / TF-GRPO）在 SWE-bench 上反而比 seed 更差，正说明它们没碰到增益的承载层 [12: §4.4.1 Table 3]。
7. 三类单组件增益相加为 +11.1 pp，但 full AHE 仅 +7.3 pp——components **非加性、彼此干扰**，且 evolve agent 优化的聚合指标被 55 个 Medium 任务支配，导致它收敛到对 Hard tier 不利的折中（memory-only 在 Hard 上 63.3% 反超 full AHE 的 53.3%）[12: §4.4.1]。
8. Evolve agent 的**修复预测**精度 33.7% / 召回 51.4%，约为随机基线（6.5% / 10.6%）的 5×——证据驱动的 targeting 不是猜测；但其**回归预测**精度 11.8% / 召回 11.1% 仅约随机基线的 2×，即 evolve agent 大体能命中"我会修好哪些"，但**几乎无法预言"我会弄坏哪些"** [12: §4.4.2 Figure 4]。
9. Loop 用三道硬约束实现 controllability：evolve agent 只允许写 workspace/、`runs/`/tracer/verifier/LLM config 全部只读、seed system prompt 标记为不可删——以排除 self-modifier 的捷径（关闭 verifier、换模型、抬 reasoning budget）[12: §3.3]。
10. 每条 edit 都附带一份 change manifest（failure_pattern / root_cause / predicted_fixes / risk_tasks），**下一轮的 task-level delta 与该 manifest 的交集**给出 per-edit verdict，未生效的 edit 在文件粒度被回滚——把 rationale-driven self-justification 换成 contract-by-next-evaluation [12: §3.3, Algorithm 1]。
11. Agent Debugger 把 trajectories 当作文件系统：每条消息一个文件，同 query 的多 trace 共置一个目录，调试器用通用 shell + scripting 工具浏览，按 progressive disclosure 输出 per-task 报告 + benchmark-level overview，作为 evolve agent 的入口文档 [12: §3.2]。
12. seed harness **必须是最小化**（只有 bash 工具、无 middleware / skill / sub-agent）才能让每个新增组件靠测得的 rollout 挣得位置；预先把 seed 调到接近 benchmark 的形态会污染所有后续 edit 的归因 [12: §3.1]。

## Assumptions

- **Base model 在 evolve / debug / code 三个角色上能力对称**——三者共享同一 GPT-5.4 high。如果 evolve agent 显著弱于 code agent，"observability 而非能力是瓶颈"的论证就崩塌；论文未做"较弱 evolve agent + 较强 code agent"的对照，因此该论点只在三者同构时成立。
- **k=2 rollouts/task 足以让 pass@1 稳定到能驱动迭代决策。** 89 任务 × 2 rollouts = 178 binary 信号，论文用 partial-pass 任务做"对比诊断"以稳定信号，但 k=2 在二项分布意义上对单任务通过率给出的方差极大；evolve agent 把这种噪声当作 per-task verdict 输入。
- **Workspace 的 git history 完整反映了 evolve agent 的 logical 决策。** 一次 logical edit = 一次 commit，rollback 在文件粒度可逆；这要求 evolve agent 严格遵守"chg-N: <desc>"的 commit 协议（论文附录 B.2 的 prompt 中作为硬规约）。
- **Manifest 中的 predicted_fixes / risk_tasks 是 evolve agent 自由声明的，没有外部强制让它"必须列足"。** 因此 fix-recall 与 regression-recall 同时取决于"agent 列了多少候选"和"列得准不准"——论文报告的是它的自然输出分布，不是给定 candidate set 上的预测准确度。
- **Terminal-Bench 2 的 89 任务对 long-horizon agentic coding 是一个**有代表性**的 distribution。** 论文承认这是"高 variance setting"（Limitations），但仍以此为唯一演化驱动器。
- **"Pass@1 是唯一优化目标"在 prompt 中显式强制**（附录 B.2 中"sole optimization target is pass@1"），但 evolve agent 实际还有 reasoning budget（隐含通过 token 成本）和 timeout 双重约束——这些非显式的副指标实际塑造了向 Medium-heavy trade-off 的收敛。
- **Component "正交"实质是 NexAU 框架的工程契约，而非论文证明的属性。** 三个 single-component swap 增益相加超出 full AHE 增益的事实（11.1 pp vs 7.3 pp）直接反驳"完全正交"假设；正交在工程上成立（不会破坏框架装载），但在效果上不成立。
- **Evolve agent 不会通过 prompt 注入 task-specific 信息。** 论文 prompt（附录 B.2）显式禁止"task-specific logic or hardcoded solutions" 与"reverse-engineer test cases from trajectories"；遵守与否依赖 evolve agent 的自觉，没有外部检测器扫描这种 leakage。

## Method

**总框架（Algorithm 1）。** 输入：seed harness H₀、base model M、benchmark D、每任务 rollout 数 k、最大迭代数 N。每轮 t=1..N 执行 6 阶段：
1. **Rollout**：用 (M, Hₜ₋₁) 在 D 上跑 k 次 → 原始 trace 集 Tₜ。
2. **Clean**：去 base64、去重 tool 输出 → T̃ₜ。
3. **Attribute**（t≥2）：把上一轮 manifest Cₜ₋₁ 与本轮观测 Tₜ 交叉得到 per-edit verdict Vₜ；按 verdict 在文件级 rollback Hₜ₋₁ 中被否决的 edit。
4. **Distill**：T̃ₜ 进 Agent Debugger → 分层 evidence corpus Rₜ。
5. **Evolve**：(Hₜ₋₁, Rₜ, Vₜ) → 新 workspace Hₜ + 新 manifest Cₜ。
6. **Commit**：git tag iteration-t，记录工作区与 manifest。

每轮结束后用 pass@1(Tₜ) 与 Hbest 比较更新 Hbest。

**3 大 observability pillar。**

- **Component observability — NexAU 解耦底座（§3.1）。** 七类组件以约定路径文件落盘：`systemprompt.md`、`tool_descriptions/*.tool.yaml`、`tools/*.py`、`middleware/*.py`、`skills/<name>/SKILL.md`、`sub_agents/<name>/agent.yaml`、`LongTermMEMORY.md`。每类组件失败模式映射到单一文件类；每个 logical edit = 一次 git commit，提供文件级 diff 与回滚粒度。Seed H₀ 仅含一个 `run_shell_command` 工具与一份 79 行最小 system prompt（附录 B.1，禁止删除）。
- **Experience observability — Agent Debugger（§3.2）。** 复用 Agent Debugger（Lin 2026）：把 trajectory 当作 navigable file environment，每条 message 一个文件，相同 query 的 traces 共置一目录。Debugger 是一个 LLM agent，被要求"分析失败根因或成功 pattern"，输出 per-task 分析报告（含 pass/fail status）；最后聚合为 benchmark-level overview.md。原始与 cleaned trace 一并提供（progressive disclosure 节省 token）。运行参数：concurrency 16、per-task timeout 600 s。
- **Decision observability — Evolve Agent + change manifest（§3.3）。** Evolve agent 读 evidence corpus，决定 add/modify/remove 哪些组件；每条 edit 写入 manifest 条目：{id, type, description, files, failure_pattern, predicted_fixes[], risk_tasks[], constraint_level, why_this_component}。Manifest 写到工作区根（不是 workspace 内），与 git history 配套。下一轮 attribute 阶段把 predicted_fixes / risk_tasks 与实际 task-level delta 求交集，得到 per-edit verdict（confirmed / falsified），falsified 的 edit 自动在文件级回滚。
- **三道硬约束（controllability）。** evolve agent 只允许写 workspace/；`runs/`、tracer、verifier、LLM config 只读；seed system prompt 不可删。这三条排除掉"关闭 verifier / 换模型 / 抬 reasoning budget"等 self-modifier 捷径，把所有可归因的增益锁在 harness 层。
- **辅助：one-shot explore agent（附录 B.3）。** 与 iteration 1 并行运行，从 NexAU 源码与公开 coding-agent 资料中蒸馏少量 reusable skills 作为 evolve agent 的"框架知识"与"SOTA 参考"两份 SKILL.md。这些 skill 不享受特别保护，迭代 2 起 evolve agent 可保留、改写或删除。

**实现细节（Setup §4.1 + 附录 A）。** 框架：NexAU；调度与隔离：Harbor；执行沙箱：E2B remote；trace 存储：InMemoryTracer + Langfuse 镜像；并发 16；per-task timeout 600 s。Trial 级 infra exception 计为 r=0（与官方榜单一致）但从 token 均值中剔除以避免截断。pass@1 = (1/(k|D|)) ∑ rᵢ,ⱼ；token cost = mean prompt+completion 跨完成 trial。Succ/Mtok = pass@1 × 10⁶ / mean_tokens。

**总结。** 这套方法把"自动 prompt 优化"重新定义为"自动 harness 全栈优化"——其核心方法贡献不是某一种 detector 或 schema，而是**把 evolve agent 的每一条编辑变成一份 falsifiable contract**：edit 自带预测，下一轮的 task delta 自动判决，被否决的 edit 在文件级回滚。三个 observability pillar 是这套契约成立的工程前置（结构化 action space + 分层证据 + 可执行 verdict）。

## Eval

- **主结果（§4.2 Table 1，Terminal-Bench 2 / 89 任务，按官方难度划分 4/55/30）。**
  - Human-designed：opencode 47.2%、terminus-2 62.9%、Codex 71.9%。
  - 同 seed 自演化：NexAU0 69.7%、ACE 68.9%、TF-GRPO 72.3%、AHE **77.0%**。
  - 难度分层：AHE 在 Easy（100%）、Medium（88.2%）领先；Hard tier 53.3% 略低于 Codex 的 56.7%（论文归因为 components 之间的 closure-style 重检干扰）。
- **跨基准迁移（§4.3 Table 2，SWE-bench-verified / 500 任务）。** AHE 75.6%（aggregate 最高），NexAU0 75.2%、TF-GRPO 74.2%、ACE 74.6%。Token: AHE 461k/trial、NexAU0 526k、TF-GRPO 582k、ACE 679k——AHE 比 seed 减 12%、比 ACE 减 32%、比 TF-GRPO 减 21%。子仓库表现：django/sphinx-doc/matplotlib/sympy 上 AHE 居首；scikit-learn/pydata/astropy 三个最小仓库上 AHE 反而下滑（论文以 small-N variance 自释）。
- **跨模型迁移（§4.3 Figure 3，Terminal-Bench 2，AHE workspace 不再演化、仅换 base）。** 5 个 base 全部正向：GPT-5.4 medium +2.3 pp（65.7→68.0）、GPT-5.4 high +7.3 pp（来自主表）、GPT-5.4 xhigh +2.3 pp（72.5→74.7）、qwen3.6-plus +6.3 pp（56.2→62.5）、gemini-3.1-flash-lite-preview +5.1 pp（36.5→41.6）、deepseek-v4-flash +10.1 pp（51.7→61.8）。论文论点：cross-family 增益（5.1–10.1 pp）系统性大于 within-family（2.3 pp）；越远离饱和的 base 越依赖 AHE 固化进 tools/middleware/memory 的"协调模式"。
- **Component ablation（§4.4.1 Table 3）。** 把 AHE 单一组件换入 NexAU0：
  - +memory only：75.3%（+5.6 pp）；Easy 50.0%（**降**）、Medium 83.6%、Hard 63.3%（**反超 full AHE 的 53.3%**）。
  - +tool only：73.0%（+3.3 pp）；Easy 75.0%（与 seed 持平）、Medium 87.3%、Hard 46.7%。
  - +middleware only：71.9%（+2.2 pp）；Easy **100%**、Medium 81.8%、Hard 50.0%。
  - +system_prompt only：67.4%（**−2.3 pp，唯一退化**）；Easy 75.0%、Medium 78.2%、Hard 46.7%。
  - Full AHE 77.0%；三正向单项相加 11.1 pp 远超 full 的 7.3 pp，证明非加性。
- **Self-attribution accuracy（§4.4.2 Figure 4，跨 9 轮均值）。**
  - Fix prediction：precision 33.7%（random 6.5%）、recall 51.4%（random 10.6%）——约 5× baseline。
  - Regression prediction：precision 11.8%（random 5.6%）、recall 11.1%（random 5.4%）——约 2× baseline。
- **缺失的对照。** 没有"evolve agent 用更弱基模 + code agent 用更强基模"的拆分（无法区分"observability vs capability"瓶颈论的真假）；没有"manifest 写但不 attribute / 不 rollback"的消融（无法量化 falsifiable contract 本身的贡献，相对于"evolve agent 单纯拿到分层 evidence"）；没有"Agent Debugger 替换为 raw trace dump"的消融（无法量化分层蒸馏的边际贡献）。
- **成本叙事。** 一次 10 轮 campaign ~32 小时；论文未拆"rollout / debug / evolve"三类 LLM 调用各占多少 token——按 SWE-bench 上 NexAU0 = 526k tok/trial 估算，单轮 89 任务 × 2 rollouts ≈ 100M token 仅 rollout，加 Agent Debugger 与 Evolve Agent 估计同量级，10 轮总 token 在 10⁹ 量级。Limitations 自承"will report these costs explicitly in the final paper"——本版本未给。

## Weaknesses

1. **三角色共享同一 base model 让"observability vs capability"瓶颈论无法证伪。** §1 论文断言"瓶颈是 observability，不是 evolve agent 能力"，但实验里 evolve agent 与 code agent 同为 GPT-5.4 high。如果 evolve agent 换成 GPT-5.4 medium、debug 与 code 不变，AHE 还能保住 +7.3 pp 增益吗？论文没回答这个问题，于是整个 motivation 缺少最关键的对照。
2. **Pass@1 在 k=2 上的统计可靠性不足以驱动迭代决策。** 89 任务 × 2 rollouts，per-task 通过率只能取 {0, 0.5, 1.0} 三档。Hard tier 30 个任务上 53.3% 与 56.7% 的差距对应 16/30 vs 17/30，单个 task 翻转就改变排序；evolve agent 把这种二项噪声当作"per-edit verdict"输入，attribute 阶段的 rollback 决策本质上在小样本噪声上回归。论文未做 bootstrapping、未跨随机种子重复，难以区分"AHE 真的找到 +7.3 pp 设计"与"AHE 沿着噪声梯度漂到一个噪声有利的局部"。
3. **Regression-prediction 几乎随机却仍被纳入"falsifiable contract"叙事。** §4.4.2 自己报告 regression-precision 11.8% / recall 11.1%（随机 5.6% / 5.4%）；这意味着 manifest 中 risk_tasks 的命中率约等于"瞎猜小集合上的 baseline"。但 §3.3 把整个 contract 机制——尤其是 rollback——卖点是"falsifiable by next evaluation"。若回归侧的预测无信号，那么 attribute 阶段实际只能利用 fix 侧的 verdict 做 rollback 决策；这与论文宣称的"对称契约"明显错位，但在主文中没有承认。
4. **Component 单项增益相加 11.1 pp 远超 full AHE 的 7.3 pp，但论文用"Medium-heavy 折中"一句话带过。** 这是结构性问题：三类组件在 closure-style 验证上彼此重复（memory 加 boundary case 重检 / middleware 加 finish-hook 重检 / system prompt 鼓励 closure），导致 turn budget 被冗余检查吃掉。AHE 的演化目标（pass@1 聚合）天然偏向 Medium，于是它系统性地放弃了 Hard tier 上的 memory 收益。这意味着同一份"Pass@1 演化器"如果用在不同难度分布的 benchmark 上会得到不同的最优 harness——Cross-benchmark transfer 章节展示的 SWE-bench 增益不能反驳这一点（SWE-bench 与 Terminal-Bench 同样以中等难度为多）。
5. **"冻结 harness 跨模型迁移"的增益实质混入了 timeout-budget 与 reasoning depth 双重耦合。** §4.3 自承 step budget / per-task timeout 是按 GPT-5.4 high 调的，medium 多了 slack 但少了一档推理、xhigh 因为推理更长更易触发 per-task timeout（被 pass@1 计为失败）。这意味着 +2.3 / +7.3 / +2.3 pp 的非单调曲线主要由 timeout 与 base capability 的相互作用决定，而非 harness 编码的"协调模式"本身。Cross-family 那 5–10 pp 的增益里有多少属于这类副作用，论文没拆。
6. **Aggregate "AHE 在 SWE-bench 比 seed 高 0.4 pp"在 500 任务量级几乎落入噪声带。** 75.6% vs 75.2% 仅 2 个任务的差距，标准误估计 ±2 pp 量级；论文却把它包装为 "tops aggregate success at 12% fewer tokens"。Token 减 12% 是真实可信的（更精简的 prompt），但 pass@1 上的领先在统计上不显著——这是叙事上的修辞越界。
7. **System prompt 单独换入造成 −2.3 pp 退化的发现，反过来质疑 evolve agent 的 system prompt 改写质量。** 论文把这项结果解读为"system prompt 描述的'纪律'的可执行性依赖其它三类组件"——但等价的解释是"evolve agent 写出的 prompt 实际上就是错的，需要其他组件代偿"。论文不去检查 evolved system prompt 与 seed 系统提示之间的具体差异、未做"用 seed prompt + AHE 其他三类组件"的对照（这其实是天然的 4th ablation cell）。如果该单格也涨分，prompt 的"零贡献甚至负贡献"的论点更强；论文回避了这次实验。
8. **Agent Debugger 这一 LLM-as-Judge 重组件未做消融。** 通篇把 Agent Debugger 视为"experience observability"的实现，从未问"如果 evolve agent 直接读 cleaned raw trace 会怎样"。Agent Debugger 本身就是一个 LLM-driven sub-pipeline，每轮会消耗与 rollout 同量级的 token；它的存在与"observability 层"的概念论证混在一起，使读者无法判断是"分层结构"在起作用，还是"又一次 LLM 重新解释 trace"在起作用。
9. **Evolve agent 写出的 manifest 是其自由声明输出，未给定 candidate task set。** Fix-recall 51.4% 这个数字的分母是"实际本轮 fix 的全部任务"，但 manifest 中 predicted_fixes 列表大小由 evolve agent 自己决定。如果 evolve agent 倾向于把列表写长，召回会自然上升、精度会自然下降；论文没报告 |predicted_fixes| 的分布，使 P/R 数字难以解读。一个 prompt-level 改动（"列出 ≥10 个候选"）就可以把 recall 推到接近 1，所以"evolution 是 evidence-driven"的核心证据具有 prompt 工程脆弱性。
10. **没有任何外部检测器扫描 evolve agent 的 task-specific leakage。** 附录 B.2 prompt 显式禁止 hardcoded solutions / reverse-engineer test cases，但执行依赖 LLM 自觉。考虑到 agent 能直接读取每个失败 task 的完整 trace（含 task name、命令、输出），"leakage 是否真的没发生"完全无法核验。SWE-bench 的"transfer"声称在此意义上不构成完整反驳——SWE-bench 与 Terminal-Bench 的 task family 差异足够大，但同质 (e.g. "django 上常见错误"在两边都出现) 的可能性论文未排除。
11. **回滚机制只在文件级，对组件之间的耦合改动不安全。** 一条 logical edit 经常跨多个文件（例如新增 middleware 同时改 `code_agent.yaml`），论文将这些放入同一 commit，按 commit 回滚——但当 evolve agent 在多次连续 commit 中迭代修改同一组件时，文件级 git revert 可能产生与 commit-tree 期望不一致的中间态。论文没有讨论这一冲突如何处理，附录 B.2 的 prompt 也没给出回滚冲突的恢复协议。
12. **"成本与效益"完全不可比。** 32 小时 + 数十亿 token 的演化得到一个 +7.3 pp 提升的 harness。论文反复强调"frozen harness transfers without re-evolution"，但既然新模型每隔 N 月就出一代、harness 的最优解会随之漂移，"演化一次用很久"的成本叙事必须给出半衰期估计。本论文 1 个月内已横跨 GPT-5.4 / qwen3.6 / gemini-3.1 / deepseek-v4——如果 6 个月后 GPT-5.6 上线、AHE 必须重跑、那么"transferable"的工程价值就比论文叙事弱得多。
13. **Limitations 自承不完整：governance 机制（bounded edits / attribution / rollback）不构成"完整 guardrail stack"。** §Limitations 承认长期 harness cleanup 与 misuse prevention 不完整，但没给出"哪些已知失效模式没被防住"的清单——读者没法判断把 AHE 搬进生产时还需要补什么。这种自承形式上诚实，实质上把风险描述外包给读者。

## Relations

- **orthogonal `01_signals_trajectory_triage` [high]**：两篇论文研究的对象虽然都是 trajectory，但优化目的不同——Signals 用 trajectory 信号选**哪条轨迹值得 LLM-judge / 人审**，AHE 用 trajectory 证据选**哪个 harness 组件值得改**。两者共享同一种工程直觉（"raw trace 不可消费、分层结构化才能可消费"），但 Signals 的 cost 在保持低（rule-based detector），AHE 的 Agent Debugger 是 LLM-driven 重组件——cost_profile 直接对立。可视为同一个 observability 论点在两个生命周期上的镜像：Signals 在 deployment-time 给 L1 triage，AHE 在 training-loop-time 给 harness self-improvement。在 thesis "lightweight signal beats LLM-judge for front-line filtering" 的语境下，AHE 的 Agent Debugger 实际是 thesis 反例的一个具体实现——它就是 LLM-judge 路线，且对其相对于"raw trace + 让 evolve agent 自己读"的边际贡献未做 ablation。
- **builds-on `04_agenttrace_structured_logging` [high]**：AHE 的 component observability 与 AgentTrace 的 schema 论点同构——给每类可观察对象一个稳定 schema、每个 schema 一组可消费的访问接口。AHE 的 7 类组件文件 + git history + change manifest 实际上就是为"harness mutation"设计的一套 trace schema。在 thesis "L0 schema 是 silent gating constraint" 的论证里，AHE 提供了第二个例证：**没有解耦的文件级 harness substrate，evolve agent 没法定位每个失败模式的归属组件**——这是一个新的 schema 必要性论据，且相对 AgentTrace 在 deployment-time 的论证转向了 training-loop-time。
- **competes-with `07_agent_as_a_judge` [high]**：AHE 的 Agent Debugger 与 Agent-as-a-Judge 范式同源——都是"用 LLM-agent 读 trajectory 做语义判定"。AHE 的设计区别仅在于：(a) 它不输出"对错评分"而输出"failure pattern + root cause"；(b) 输出消费方是 evolve agent 而非 outcome-evaluator。但 cost、可解释性、可复现性问题完全继承自 Agent-as-a-Judge——AHE 没有讨论这些遗传缺陷。在 thesis "把 LLM-judging 留给 triaged 后的小子集而非 front-line"的判断方向上，AHE 把 LLM-judging 用在了**全部 trajectory**（Agent Debugger 跑每个任务的 k=2 traces），这是 thesis 明确反对的部署模式。
- **orthogonal `08_tide_trace_diagnostics` [high]**：TIDE 是 deployment-time post-hoc 失败诊断，AHE 的 Agent Debugger 是 training-loop-time 失败诊断驱动 harness 演化。两者在"用 LLM 解读 trace 提取根因"这一步的方法论几乎相同；差别在 downstream 消费方：TIDE 输出给人类工程师做修复决策，AHE 输出给 evolve agent 做自动修复决策。AHE 的贡献是把 TIDE-style 诊断**喂回到一个自动循环**里——这是在考虑用 LLM 诊断驱动 detector versioning 时可以借鉴的闭环结构（但需要先解决 thesis 关心的成本与可信度问题）。
- **orthogonal `02_agenther_hindsight_relabeling` [med]**：AgentHER 把失败轨迹做 hindsight relabel 成 SFT/DPO 数据用于 model 演化；AHE 把失败轨迹做 root-cause analysis 驱动 harness 演化。两者位于 thesis 四层栈的不同点：AgentHER → L2/L3，AHE → 工程层（不在四层栈内）。但关键观察是：**AHE 的实证表明 system prompt 单换入会退化 −2.3 pp，long-term memory 单换入 +5.6 pp**——意味着 model 没变、把"经验"显式落到外部 memory 文件比落在 prompt 里效果更稳定。这给 AgentHER 路线一个不愉快的拷问：如果"externalize experience to artifacts"的工程方案就能 +5.6 pp，相对于 hindsight relabel + SFT 训练的成本-收益比，model-side relabeling 的"必要性"需要重新论证。
- **orthogonal `05_breaking_observability_tax` [low]**：observability tax 论文关心 production trace 的采样与压缩成本；AHE 关心 evolution-loop 中 trace 的分层蒸馏成本。两者面对的"trace 量级问题"不同：前者是 10⁶ traces × 1× cost / per-trace（部署期），后者是 10² tasks × k rollouts × multi-million-token traces（演化期）。AHE 的 Agent Debugger 实际是 observability tax 论文 sentinel sampling 的反向：每条 trace 都进 LLM 解读，没有采样、没有 sentinel——因为 evolution loop 的 N 远小于生产的 N。这反衬出 thesis 论证的"sampling 之于 production"与"distillation 之于 evolution"是两个不可互换的成本结构。
- **competes-with `09_trajectory_guard_a_lightweight_sequence_aware` [low]**：TrajectoryGuard 用学习到的小型 sequence model 做 trajectory anomaly 判定；AHE 的 Agent Debugger 用 GPT-5.4 high 做 trajectory 根因诊断。两条路线代表"learned small surrogate vs LLM-as-judge in distillation role"的极化对照。TrajectoryGuard 的 ~0.92 F1 用在"哪条 trace 异常"这个粗粒度任务上够用；AHE 用 LLM 诊断目的是"输出可读的 root cause 文本给另一个 agent 消费"，不是 0/1 anomaly——所以严格说不是同一任务，但 cost 与可解释性维度构成实质对比。
- **builds-on `06_agentseer_agentic_vulnerabilities` [low]**：AgentSeer 用 action-component graph 找轨迹结构异常；AHE 把 harness 自身分解为 7 类正交"action-component"。两者都把"组件分解"当作可观察性的前置条件。但 AgentSeer 是在 trajectory 层做组件级分析（把跑出来的行为分解为 component graph），AHE 是在 harness 层做组件级分解（把可编辑面切成 7 类文件）。两个层级的分解是互补的：生产分诊层若同时部署，可以把 AgentSeer 的 trajectory-level component graph 反查到 AHE 的 harness-level component（"哪个 middleware / tool 的失败导致 graph 中这条边异常"），这是一个潜在工程组合点。
- **contradicts `11_near_miss_latent_policy_failure_detection` [med]**：Near-Miss 把"判定应当机制化、可审、不依赖 LLM 推理"作为方法论立场（用 guard code + 结构化历史搜索代替 LLM-judge）；AHE 的 Agent Debugger 几乎走相反路线——把诊断完全交给 LLM agent 在文件系统上自由探索。两篇论文对"trajectory-level 自动化判定该不该用 LLM"给出了对立答案。在 thesis "mechanistic explanations over correlation studies"和"reproducibility-of-method matters"的判断方向上，Near-Miss 路线更受偏好；AHE 的实证成功（+7.3 pp）则证明 LLM-driven 路线在某些工程目标下也能交付，但代价是 32 小时 + 数十亿 token 的成本曲线不可与 Near-Miss 同台比较。
- **orthogonal `10_policy_invisible_violations_in_llm_based` [low]**：Sentinel 在线拦截显式违规、在 deployment-time 工作；AHE 在 training-loop-time 修改 harness。两者在生命周期与判定信号上正交，但若把 AHE 视为"工程层 self-improvement"，那么 Sentinel 这类 deployment-time policy enforcement 提供的违规日志反过来可以作为 AHE 的失败证据输入——给 evolve agent 一个非 LLM 来源的 ground truth label，部分缓解上文 weakness #10 的 leakage 风险。这是生产化视角下的潜在组合点，论文双方均未提及。
- **orthogonal `03_tsr_trajectory_search_rollouts` [low]**：TSR 是 training-time rollout selection（哪条 rollout 进 RL 训练数据）；AHE 是 training-loop-time harness mutation。两者都"在跑出来的 rollouts 上做选择"，但 TSR 选数据、AHE 选 harness 编辑。在生命周期标签上 AHE 更接近 TSR 而非 Signals/Near-Miss——它是一个**演化优化器**，只不过演化对象从 model parameters / data 换成了 harness components。
