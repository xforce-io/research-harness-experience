# Thesis

> Working thesis lives here. Researcher 每轮读取它，判断新材料是支持 / 扩展 / 挑战 / 正交。Researcher 只报告矛盾，不编辑本文件。

## Working thesis

Silver & Sutton《Welcome to the Era of Experience》（2025）认为，人类数据驱动的进步正在见顶，下一截要靠 agent 自己与环境交互产生的 experience。姚顺雨《The Second Half》把同一转折说成：通用 recipe 已经够用，下半场比的是定义任务与评价，而不是再发明训练方法。

本支柱把这两点收成工程主张：生产 agent 的改进介质应是**自身交互流**——捕获（结构化轨迹 + 轻量分诊）→ 更新 → 用对齐部署的评价验收。更新是**两条并列路径**，不是主路/旁路：A 改权重，B 改 harness。评价是横切两支的门，不是流水线第三节。内部层层如何消费、何时走 A / B / 叠 / 不更新，见 Design Context。

**流（捕获）。** 部署后改进的第一缺口仍是轨迹流与可学习信号之间的桥。L1 分诊应主要用非语义的、规则化或小代理检测器，作用于交互 / 执行 / 环境三类面，而非逐轨迹 LLM-as-Judge。Signals 在 τ-bench 上以 ~1.5× 采样效率达到 82% informativeness（未独立复现）是工作点证明。最被低估的收益在**成功**轨迹：约三分之二「任务完成」仍含可学习隐性摩擦。Schema（L0）是沉默门槛：操作 + 认知 + 上下文不够，还必须有用户话语与系统资源状态。观测成本用信号触发的哨兵升档，而非均匀降采样。LLM-judge 只做分诊后小子集上的深度诊断。判断链把确定性尽量推深：规则可枚举时机制化 oracle 优于 LLM；不可约语义谓词须包进窄 yes/no 决策树，拒绝 free-form「这条轨迹好不好」。

**更新 A（改权重）。** 失败是能力或偏好错：工具用错、推理模式坏、合规绕过但能做对。分诊后走 hindsight relabel / 过程侧信用，进入偏好或 RL。未分诊的 raw dump 或纯轨迹级 outcome 不够。过程监督若出现，须能对照 outcome-only；过密的 turn 信号可以引入噪声。通用 agentic RL recipe、scale-only、未接地的偏好优化**不是**本支柱的 inclusion。

**更新 B（改 harness）。** 失败是基板或 runtime 错：缺工具、缺记忆、缺守卫、lifecycle 接口不对。不改 target 权重，编辑可执行 harness（文件级 substrate / lifecycle hook / 推理时编译的 runtime）。权重冻住、要可回滚、一条 patch 需泛化到多任务时走这条。B 与 A 并列、可叠，不是 A 做完才轮到的旁路。

**评价（横切门）。** A 的新策略与 B 的新 runtime 都须过同一套门才算改进：搜索与终评分离、同预算采样对照、held-out。Rethinking Harness Evolution Eval 表明 evolved harness 在 held-out 上平均仅 +0.6、表观收益常可被多采样解释。不审计反馈自身可靠性的自动修复闭环（只报端到端 Δ）降权。执行中途纠偏（in-flight）不落盘，**不是**更新，不得计入进化。

可证伪：
- (流) 轻量信号在非 τ-bench 语料上达不到 >70% informativeness；或未分诊端到端 RL 在成本归一化生产指标上打赢闭环。
- (更新 A) L1 子集上的 hindsight 相对随机偏好无下游胜率；或 SWE-bench 类等预算下 outcome-only 匹配过程监督。
- (更新 B) 与 parallel/sequential sampling 同预算且 held-out 分离后，仅改 harness 无稳定增量（[21] 的 +0.6 量级不算）；或冻结 target 时 harness 编辑系统性压不过「只多样本」基线。
- (评价) 下一代通用模型不改 env/eval 即系统性压过 harness **与** 权重侧改进；或人类偏好数据持续比接地 outcome 更便宜更稳。

## Design Context

CHARTER 的「捕获 → 更新 → 评价」是**支柱边界**（experience 拥有整段闭环，不另立 evolution）。下面是闭环**内部**架构。报告 spine 按本节约目标展开，不要只用三词口号当章节。

### 分层与流程

```
L0 schema → L1 分诊 →（可选归因）┬─► 更新 A 改权重 ──┐
                                 ├─► 更新 B 改 harness ┼─► 评价门 → 回写
                                 ├─► A 与 B 都走 ──────┘
                                 └─► 非更新：in-flight 纠偏（不落盘）
```

生命周期三个位置——deployment post-hoc / training-loop / online——是**何时跑**，不是第三套分层。online 上的 in-flight 纠偏默认走非更新。

| 层 | 吃 | 吐 | 缺了会怎样 |
|---|---|---|---|
| L0 schema | raw trace | 可消费轨迹 | 缺字段则下游静默失效 |
| L1 分诊 | 可消费轨迹 | 值得动作的子集 + 信号 | 前线若上 LLM-judge，成本与方差不可接受 |
| 归因（可选） | 已失败 / 可疑轨迹 | 哪一步坏 | 不是分诊（哪条值得看），也不是评价（好不好） |
| 更新 A | 偏好对 / 过程信用 | 新权重 | raw dump 几乎学不回来 |
| 更新 B | failure packet | 可执行 harness patch | 无组件/hook 定位则无法编 |
| 评价门 | 新策略或新 runtime | 验收通过才回写 | 无同预算对照、无 held-out，不算改进 |
| 非更新 | 执行中途窗口 | 临时纠偏 | 弱 judge 会把策略带偏；不沉淀 |

旧四层栈（L0 tracing → L1 分诊 → L2 重打标 → L3 模型迭代）是 **A 路径** 的层间消费，嵌在捕获 + 更新 A 里，不再当整仓唯一骨架。Harness 编辑是与 A 并列的更新 B，不是 off-axis。

### G1 捕获：schema 是门槛，前线用轻量信号

L0 先行：操作 + 认知 + 上下文不够，还要用户话语与系统资源状态，否则 L1 的 Interaction / Environment 信号无法计算。前线用轻量信号（规则 / 小代理 / 机制化 oracle）；LLM-judge 只留给分诊后小子集。成功轨迹的隐性摩擦是成熟系统的主矿，不是猎奇。观测成本用信号触发的哨兵升档，不均匀降采样。

### G2 更新 A：能力或偏好错则改权重

场景：要内化、要跨环境迁移、有可配对正负样本（失败 hindsight，或成功轨迹上「绕过 vs 合规」）。输入是偏好对或过程信用，不是 raw dump。过程监督必须能对照 outcome-only。

### G3 更新 B：基板或 runtime 错则改 harness

场景：权重冻住（API / 弱模型 + 强 builder）、要可回滚、一条 patch 能泛化到许多任务、失败能定位到组件或 lifecycle hook。输入是 failure packet → 可执行 diff/hook。表观涨点必须过评价门，不能把多采样写成进化。

### G4 双轨分流：A 与 B 并列，可叠，先分流再更新

A 回答「策略会不会这件事」；B 回答「runtime 让不让它做成这件事」。不是主路/旁路，也不是「先 A 后 B」。co-evolve 是互补证据，不是替代证据。

分诊后四路：

1. **失败 + 可配对纠正** → A
2. **系统性问题、可定位到组件/hook** → B
3. **成功但有隐性摩擦** → 优先作 A 的原料（不是 B 的主场）
4. **单次可救、不该沉淀** → 非更新（in-flight）；必须信号门控，禁止无条件逐步 LLM

可叠：同一条轨迹可以既出偏好对又出 harness patch。不可把 in-flight 收成第三支进化。

### G5 评价是门：两条更新共用同一套纪律

搜索 ≠ 终评。与同预算 sampling 对照。held-out。自动修复要审计反馈自身（训练期 outcome 接地与推理期可证伪合同是两轴，生产要两轴过线）。A 的 win rate 与 B 的 Δ reward 都过这扇门，评价不是只卡 harness 的第三节。

## Taste

- 前线过滤偏好轻量、可部署的信号，而非 LLM-judge。
- 偏好机制性解释与可复现的检测器 / schema / 评价协议。
- 生产细节（成本、漂移、schema 版本）是一等关切。
- 轨迹级与闭环级优于单轮；成功轨迹的隐性摩擦是特性。
- 更新必须先分流再动手：改权重还是改 harness 是并列选择，不是标签。
- 评价横切 A 与 B；能拆掉采样假象才算改进。
- in-flight 纠偏默认非更新，除非明确落盘为 A 或 B 的产物。

## Anti-patterns

- 纯 benchmark、无方法或检测器。
- 纯 LLM-as-Judge 且无视逐轨迹成本。
- 通用 agentic RL / scale-only / world model / serving / 决策编排——归其他支柱或上半场。
- 把采样 informativeness 与判断精度混为一谈。
- 不审计反馈可靠性的自动修复闭环。
- 搜索与终评共用同一 benchmark 却宣称 harness 进化。
- 用 CHARTER 三词口号代替内部分层；或把旧 L0–L3 当作仍覆盖 harness 的唯一骨架。
- 把更新 B 写成 off-axis / 旁路，或把 in-flight 算进进化。
- 有数据就 SFT、有失败就全量喂 editor：未分诊、未分流。

## Examples

- 流 / G1：`notes/01_signals_trajectory_triage.md`、`notes/04_agenttrace_structured_logging.md`、`notes/11_near_miss_latent_policy_failure_detection.md`
- 更新 A / G2：`notes/02_agenther_hindsight_relabeling.md`、`notes/23_trace_turn_level_reward_assignment_via.md`、`notes/22_sample_efficient_learning_from_agent_experience.md`
- 更新 B / G3：`notes/19_harness_r1_learning_to_edit_executable.md`、`notes/12_agentic_harness_engineering_observability_driven_automatic.md`、`notes/20_ai4ai_at_test_time_strong_to.md`
- 分流 / G4：`notes/02_agenther_hindsight_relabeling.md` 与 `notes/19_harness_r1_learning_to_edit_executable.md` 对照（权重 vs harness，可 co-evolve）
- 评价门 / G5：`notes/21_rethinking_the_evaluation_of_harness_evolution.md`、`notes/12_agentic_harness_engineering_observability_driven_automatic.md`
- 非更新（对照）：`notes/16_when_agents_go_astray_course_correcting.md`、`notes/07_agent_as_a_judge.md`
