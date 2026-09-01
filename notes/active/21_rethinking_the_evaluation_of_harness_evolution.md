---
zone: active
tags: ["evaluation_protocol"]
pin: false
score: 0.4764255319148937
dwell: 2
---
# Rethinking the Evaluation of Harness Evolution for Agents

> Topic integration note derived from Library read artifact `.researcher-workspace/library/papers/paper_arxiv_2607_12227/reads/read_paper_arxiv_2607_12227.md`.

## Library read

---
kind: library-read-identity
authors: ["Yike Wang","Huaisheng Zhu","Zhengyu Hu","Yige Yuan","Zhengyu Chen","Shakti Senthil","Hannaneh Hajishirzi","Yulia Tsvetkov","Pradeep Dasigi","Teng Xiao"]
source_id: "arxiv:2607.12227"
source_url: "https://arxiv.org/abs/2607.12227"
pdf_url: "https://arxiv.org/pdf/2607.12227"
---
> 不再让 harness evolution 在同一套 benchmark 上"既搜索又评测”：把它与 parallel sampling、sequential refinement 锁进同一份 K=5 反馈/推理预算下对齐比较，并用不相交任务集检验迁移——结果显示其收益大多可由“多采样”解释。

## Essence

**问题**。现有自动 harness evolution 方法(Meta-Harness、AHE、AEVO)用 benchmark 单测反馈迭代搜索 harness 配置，又在同一 benchmark 上报告最终性能：无法区分收益来自更好的 harness 设计还是测试时额外搜索，且有过拟合评测集的风险。

**做法**。把四种方法统一到一个预算框架(更新对象 × 反馈信号 × 预算 K=5):parallel sampling 在固定 harness 下并行采 K 条轨迹；sequential refinement 逐条自我修订；harness evolution 由 meta agent 读跨任务经验库迭代改 harness;新提出的 harness scaling 把同一修订循环搬到单任务级。所有方法共用极简初始 harness(仅一个 bash 工具)和同一个 summarization 组件。

**证据**。在 45/10/34 的不相交 train/val/test 划分上，evolved harness 在 held-out 任务平均仅 +0.6 pass@1(GPT-5.4 上 +0.0),远小于同任务“搜了再测”时的表观收益。

**边界**。结论限定于 Terminal-Bench 2.1、K=5 预算、单一被裁剪的 AHE 实例(explore agent 关闭)；不是"harness engineering 无用”的证明——作者自己指出 TB 可能对 harness 设计不敏感。

## Claims

1. Harness evolution 本身是迭代搜索过程，只有与 test-time scaling 基线在匹配的反馈与推理预算下对比后，其收益才能归因于 harness 设计而非额外搜索 (§1)。
2. 搜索与最终评测共用同一 benchmark 会混淆"harness 改进”与“对评测实例的适配”，表观增益会高估真实改进 (§1, §4.4)。
3. 有单测时，harness evolution 的收益只出现在允许跨候选选择的 pass@5 上，pass@1 几乎不动——说明它没有通过更好的 harness 解出新任务，收益主要来自多次尝试本身 (§4.3)。
4. Meta agent 的编辑方向是理性的(prompt 层行为规则 → middleware 层运行时强制 → 工具层修正)，但 "most edits memorize fixes rather than distilling strategies" (§5.1);且持续膨胀的持久 prompt 文本带来 context bloat,抵消剩余收益 (§5.1)。
5. 无单测(仅自我反馈)时，harness evolution 三模型平均 67.4,低于 direct sampling 的 68.2;在 GPT-5.4 上从 75.3 降至 69.7——自我反馈引导的 harness 修订会主动伤害强模型 (§4.2, Table 1)。
6. 有单测时 harness evolution 仍落后:pass@1 平均 75.8 vs Parallel Sampling 86.0;pass@5 86.2 vs Sequential Refinement 91.8 (§4.3, Table 2)。
7. Disjoint split 下，evolved harness 在测试集平均仅 +0.6(Claude Opus 4.6 +1.2,GPT-5.4 +0.0),与同任务评测时的收益形成鲜明反差——修订编码的是任务特定捷径而非可迁移的设计原则 (§4.4, Table 3)。
8. 任务级的 harness scaling 部分弥补：无单测平均 71.8,并在 Claude Opus 4.6 上取得单模型最佳 76.0,但仍不构成对简单基线的一致优势 (§4.2)。

## Assumptions

- “匹配预算”以轮数/rollout 数计(K=5;AHE 每 harness 每任务 m=1),meta agent 与 Agent Debugger 的额外推理不计入预算 (§4.1, Table 4)。
- 关闭 explore agent 的 AHE 可代表"harness evolution"这一类方法；Meta-Harness 与 AEVO 被引用但未被执行 (§3.4, 附录 A.3)。
- Policy、meta agent、debugger、self-judge 全部由同一底层模型实现即可 (§3.1, Table 4)。
- 2 次独立运行的平均足以控制 rollout 方差 (§4.1)。
- 极简初始 harness(仅 bash 工具、无 skills/middleware/记忆)对所有方法是公平的共同起点 (附录 A.1)。
- 基础设施异常(sandbox 崩溃、API 超时)计 0 分而非剔除 (附录 A.5)。

## Method

对比旧协议：

| | 旧协议 | 本文协议 |
|---|---|---|
| 预算 | 不对齐，只报最终分 | 统一 K=5,显式指定反馈与更新对象 |
| 评测集 | 搜索与评测同 benchmark | 增加 disjoint train/val/test |
| 基线 | 无 test-time scaling 对照 | parallel / sequential / harness scaling |

形式化：policy πθ;任务 x∼X;harness h 下轨迹 y∼πθ(·|x;h);有单测 g 时 outcome R(y,g)∈{0,1};summarization map Φ(由 AHE 的 Agent Debugger 实现)把经验库压缩为失败模式/冗余/成本摘要。

- **Parallel sampling**:固定 h 独立采 K 条；无单测用 self-judge 取 argmax J(y_k),有单测取任一 R=1。
- **Sequential refinement**:y_k 条件于 Φ(y_{k−1})(无单测)或 Φ(y_{k−1}, R)(有单测)；返回末条或任一通过者。
- **Harness evolution**:抽 n 个任务成批，每轮每任务 m=1 rollout,证据 e_k=(h_k, rollouts[, outcomes]) 累入经验库 C_k;meta agent 生成 h_k = M(Φ(C_{k−1}));无单测取 h_K,有单测取聚合 outcome 最高的 ĥ,再在 ĥ 下采目标任务的最终轨迹。实例化为 AHE,explore agent 被禁用(理由：该组件是从外部检索 benchmark 专属 harness,混淆演化与检索)。
- **Harness scaling(新)**：单任务版——h_k = M(x, Φ(h_{k−1}, y_{k−1}[, R])),逐轮改 harness 并重采样同一任务。

输出：每任务一条最终轨迹 ŷ;指标 pass@1 / pass@5。

## Eval

- **数据**:Terminal-Bench 2.1,修复 TB 2.0 中 28/89 任务的依赖漂移、资源预算错配与指令-测试不一致，共 89 个终端任务 (§4.1)。
- **模型**:Claude Opus 4.6、GPT-5.4、GPT-5.4 mini;128k max generation、high reasoning;每 rollout 独立 E2B sandbox;结果为 2 次运行平均。
- **基线**:direct sampling(初始 harness)、parallel sampling、sequential refinement、harness scaling、harness evolution(AHE 裁剪版)；全部从同一极简初始 harness 出发，K=5。
- **指标**:pass@1(主)；有单测时加报 pass@5;infra 异常计 0。
- **设置与结果**：无单测(Table 1,三模型平均:parallel 72.3 > harness scaling 71.8 > sequential 69.3 > direct 68.2 > harness evolution 67.4);有单测(Table 2:pass@1 parallel 86.0 / harness evolution 75.8;pass@5 sequential 91.8 / harness evolution 86.2);disjoint(Table 3:测试集平均 +0.6)。

## Weaknesses

1. 只实测了一个被裁剪的 AHE 实例，却把结论写成对"automatic harness evolution"整类的质疑；Meta-Harness 与 AEVO 未被执行，也没有全量 AHE(含 explore agent)的对照数字来量化该组件对已发布收益的贡献 (§3.4, §6, 附录 A.3)。
2. 预算等价按 rollout/轮数对齐而非 token:meta agent(上限 500 turns)与 Agent Debugger 的推理不计入，"comparable inference budgets" 是结构性等价而非成本等价 (§1, §4.1, Table 4)。
3. 无显著性检验、无方差/区间报告，仅 2 次运行平均；部分结论级差距(如 harness scaling 71.8 vs parallel 72.3)很可能在噪声范围内 (§4.1, Table 1)。
4. 泛化结论只覆盖单一 benchmark 内部的任务级迁移(34 个测试任务)，未测跨 benchmark 迁移；且仅用 10 个验证任务做 harness 选择，选择本身噪声很大 (§4.4)。
5. 无单测设置下各方法的选择规则不对称:parallel 用 self-judge 挑最优候选，sequential 只取最后一条，可能系统性低估后者 (§3.2–§3.3)。
6. Table 1 含 GPT-5.4 mini 而 Table 2/3 略去，未解释的不对称 (§4.2–§4.4)。
7. 头条论断建立在作者自己在 §5.2 论证“可能对 harness 不敏感、剩余失败多源于模型能力”的 benchmark 上，且未在任何 harness 敏感套件上复核——证据范围与结论措辞(对整个领域的警示)之间存在张力 (§5.2 vs §6)。
8. TB 2.1 修复了 28/89 任务，与既有工作在 TB 2.0 上的已发表数字不可直接对照，因此本 re-evaluation 并未直接驳倒已发布结果，论文未讨论这层不可比性 (§4.1)。
9. pass@5 口径下的预算记账未说明：harness evolution 的 K=5 花在进化阶段后，最终 pass@5 的多条采样如何计入预算、direct sampling 的 pass@5 为何缺省("-"),均未交代 (§3.4, Table 2)。

## Relations

- contradicts Lin et al. 2026 (Agentic Harness Engineering, AHE) [high]: 本文以(裁剪版)AHE 为唯一 harness evolution 实例，实测其平均落后于 test-time scaling 基线，直接对立于 AHE 的核心主张；矛盾对象是关闭 explore agent 后的版本。
- contradicts Lee et al. 2026 (Meta-Harness) [med]: 协议批评(搜索与评测共用同一 benchmark)直接指向其评估方式，但未运行 Meta-Harness 本身，矛盾停留在协议层面而非测量层面。
- builds-on Snell et al. 2024 / Wang et al. 2022 / Brown et al. 2024 / Madaan et al. 2023 [high]: parallel sampling 与 sequential refinement 基线显式取自这条 test-time scaling 文献线(§2, §3.2–§3.3)。
- extends test-time scaling 文献线 [high]: §3.5 明确将新提出的 harness scaling 定位为 harness 层面的 test-time scaling 类比，把推理时算力的作用轴从轨迹扩展到 harness。
- builds-on Merrill et al. 2026 (Terminal-Bench) [high]: 全部实验在 TB 2.1 上进行(§4.1)。
- orthogonal Zhang et al. 2025 (ACE) / Zhao et al. 2024 (ExpEL) [low]: 二者为单组件经验外化方法；本文“记忆化 vs 策略蒸馏”的诊断与之一致相关，但未评测它们，连接是间接推断。

## Takeaway

- 方法论可移植：对任何“迭代搜索式改进”方法(prompt 优化、harness 演化、memory 积累)，用统一的(更新对象 × 反馈 × 预算)三元组与 test-time scaling 基线对齐，并把优化反馈与最终测量分离到不相交任务集——这是本文最值得复制的评估协议。
- 需要复核：单一裁剪实例 + 单一(可能 harness 不敏感)benchmark + 2 次运行 + 无显著性检验；“harness evolution 无效”与“harness evolution 在 TB 2.1 上无效”之间尚有距离。
- 记住一件事：evolved harness 在 held-out 任务上平均只 +0.6 分——收益主要来自采样，而不是 harness 设计。
