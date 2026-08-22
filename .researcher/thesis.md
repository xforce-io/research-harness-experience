# Thesis

> 本仓由 `research-harness-trace` **就地升级**为 experience 闭环（同一 GitHub 仓库对象）。Researcher 每轮读取本文件以判断新材料是支持 / 扩展 / 挑战 / 正交。Researcher 只报告矛盾，不编辑本文件。

## Working thesis

生产 agent 的改进介质应是**自身交互流**，而不是更多人类语料或新 trainer：捕获（结构化轨迹 + 轻量分诊）→ 更新（权重 **或** harness）→ 用对齐部署的评价验收。Sutton/Silver 的 Era of Experience 给出介质换代；姚顺雨的下半场给出瓶颈换位——评价与任务定义重于再发明 recipe。

**流（原 L0–L1，捕获章）。** 部署后改进的第一缺口仍是轨迹流与可学习信号之间的桥。L1 分诊应主要用非语义的、规则化或小代理检测器，作用于交互 / 执行 / 环境三类面，而非逐轨迹 LLM-as-Judge。Signals 在 τ-bench 上以 ~1.5× 采样效率达到 82% informativeness（未独立复现）是工作点证明。最被低估的收益在**成功**轨迹：约三分之二「任务完成」仍含可学习隐性摩擦。Schema（L0）是沉默门槛：操作 + 认知 + 上下文不够，还必须有用户话语与系统资源状态。观测成本用信号触发的哨兵升档，而非均匀降采样。LLM-judge 只做分诊后小子集上的深度诊断。判断链把确定性尽量推深：规则可枚举时机制化 oracle 优于 LLM；不可约语义谓词须包进窄 yes/no 决策树，拒绝 free-form「这条轨迹好不好」。

**更新。** 分诊后的轨迹有两条合法更新路径，平行不互斥：(a) hindsight relabel / 过程侧信用进入偏好或 RL；(b) 不改 target 权重、编辑可执行 harness（Harness-R1）。未分诊的 raw dump 或纯轨迹级 outcome 不够。过程监督若出现，须能对照 outcome-only；过密的 turn 信号可以引入噪声。通用 agentic RL recipe、scale-only、未接地的偏好优化**不是**本支柱的 inclusion。

**评价。** harness 或策略的表观涨点必须在搜索与终评分离、且与同预算采样对照后才算改进。Rethinking Harness Evolution Eval 表明 evolved harness 在 held-out 上平均仅 +0.6、表观收益常可被多采样解释。不审计反馈自身可靠性的自动修复闭环（只报端到端 Δ）降权。

可证伪：
- (流) 轻量信号在非 τ-bench 语料上达不到 >70% informativeness；或未分诊端到端 RL 在成本归一化生产指标上打赢闭环。
- (更新) L1 子集上的 hindsight 相对随机偏好无下游胜率；或 SWE-bench 类等预算下 outcome-only 匹配过程监督。
- (评价) 下一代通用模型不改 env/eval 即系统性压过 harness 改进；或人类偏好数据持续比接地 outcome 更便宜更稳。

## Taste

- 前线过滤偏好轻量、可部署的信号，而非 LLM-judge。
- 偏好机制性解释与可复现的检测器 / schema / 评价协议。
- 生产细节（成本、漂移、schema 版本）是一等关切。
- 轨迹级与闭环级优于单轮；成功轨迹的隐性摩擦是特性。
- 更新必须说明改的是权重还是 harness；评价必须能拆掉采样假象。

## Anti-patterns

- 纯 benchmark、无方法或检测器。
- 纯 LLM-as-Judge 且无视逐轨迹成本。
- 通用 agentic RL / scale-only / world model / serving / 决策编排——归其他支柱或上半场。
- 把采样 informativeness 与判断精度混为一谈。
- 不审计反馈可靠性的自动修复闭环。
- 搜索与终评共用同一 benchmark 却宣称 harness 进化。

## Examples

- 流：`notes/01_signals_trajectory_triage.md`、`notes/04_agenttrace_structured_logging.md`、`notes/11_near_miss_latent_policy_failure_detection.md`
- 更新：`notes/02_agenther_hindsight_relabeling.md`、`notes/19_harness_r1_learning_to_edit_executable.md`、`notes/23_trace_turn_level_reward_assignment_via.md`
- 评价：`notes/12_agentic_harness_engineering_observability_driven_automatic.md`、`notes/21_rethinking_the_evaluation_of_harness_evolution.md`、`notes/20_ai4ai_at_test_time_strong_to.md`
- 对照（非前线）：`notes/07_agent_as_a_judge.md`
